---
name: perception-stack
description: Use when configuring cameras (RealSense, ZED, generic CSI), calibrating intrinsics or extrinsics, syncing time across sensors, or building a GPU-accelerated perception pipeline. Surfaces the patterns and silent failure modes.
---

Reference for the camera / depth / time-sync layer underneath robotics work. Most "the policy doesn't see what I see" failures are perception-stack failures: wrong color order, wrong depth units, unsynced timestamps.

## Camera calibration

Two distinct calibrations:

1. **Intrinsics** (focal length, principal point, distortion). Per camera. Use `camera_calibration` (ROS) or OpenCV directly with a checkerboard/charuco target. Save as `camera_info.yaml` and load via the camera driver.
2. **Extrinsics** (camera → robot frame). Hand-eye calibration. Use `easy_handeye2` or `MoveIt 2 Calibration`. The four common variants:
   - **Eye-on-hand** (camera mounted on EE) → solves `T_ee_camera`.
   - **Eye-to-hand** (camera fixed, watching the arm) → solves `T_world_camera`.
   Each requires moving the arm through ≥10 distinct poses with a target visible.

**Calibration drifts** — re-do whenever:
- The camera or its mount moves (even small bumps).
- The lens is replaced or refocused.
- The robot base moves (eye-to-hand only).
- A new tool is mounted (eye-on-hand: `T_ee_camera` doesn't change but downstream `T_camera_tool` does).

## RealSense (D4xx series)

Launch via `realsense2_camera`. Critical params:

```yaml
align_depth.enable: true        # depth registered to color frame; you almost always want this
enable_sync: true               # hardware-sync depth and color streams
depth_module.profile: "640x480x30"
rgb_camera.profile: "640x480x30"
pointcloud.enable: false        # off unless you actually consume it (CPU/bandwidth heavy)
spatial_filter.enable: true     # smoother depth at cost of edge detail
temporal_filter.enable: true    # ditto
hole_filling_filter.enable: false  # adds artifacts; usually skip
```

**Watch out for:**
- **Depth in mm.** RealSense returns uint16 mm. Convert to meters before consumption.
- **First-frame zero depth.** The IR projector takes ~1 sec to settle. Skip the first 30 frames.
- **USB bandwidth.** D435 + D435 on one USB controller saturates. Spread across controllers.
- **Reflective / transparent objects.** Depth is invalid; the policy must tolerate NaN/zero or use color-only inputs.

## ZED (Stereolabs)

Launch via `zed-ros2-wrapper`. Critical params:

```yaml
general.camera_model: "zed2i"     # or zedx, zed-mini
general.resolution: "HD720"
depth.depth_mode: "NEURAL"        # neural depth = better; needs Jetson or RTX GPU
depth.min_depth: 0.3              # below this, no depth
sensors.publish_imu_tf: false     # unless you need IMU TF
pos_tracking.area_memory: false   # turn off unless using SLAM
```

**Watch out for:**
- **NEURAL depth requires CUDA.** On CPU-only or weak GPU, fall back to ULTRA or QUALITY.
- **IMU is in camera frame, not robot frame.** Apply `T_robot_camera` to use it for state estimation.
- **Tracking and depth share GPU.** If you want max depth FPS, disable positional tracking.

## Time synchronization

Three problems, three answers:

1. **Sync within a single multi-sensor camera (e.g. RGB + depth on RealSense).** Hardware-sync via the camera (`enable_sync: true`). Trust the timestamps.
2. **Sync across separate sensors (camera + IMU + joint encoders).** Use `message_filters.ApproximateTimeSynchronizer` in ROS 2:

```python
from message_filters import Subscriber, ApproximateTimeSynchronizer
img_sub = Subscriber(self, Image, "/camera/color/image_raw")
js_sub = Subscriber(self, JointState, "/joint_states")
sync = ApproximateTimeSynchronizer([img_sub, js_sub], queue_size=10, slop=0.05)
sync.registerCallback(self.synced_cb)
```

3. **Sync across machines.** PTP (chrony with `ptp4l`) on a wired network. NTP isn't tight enough for vision-control loops.

**Use `use_sim_time: true` consistently across the launch graph** when running with `/clock`. Mixed sim/wall time is the most common "TF says my transform is in the future" cause.

## GPU-accelerated pipelines

For high-rate perception:

- **CUDA stream from camera straight to GPU.** Avoid bouncing image buffers through CPU. RealSense and ZED both have CUDA-direct paths.
- **JPEG decode on GPU.** `nvjpeg` for offline; `nvJPEG2000` for compressed sensor streams.
- **Composable nodes (ROS 2)** for zero-copy intra-process. Combine debayer, rectify, depth-align in one process.
- **TensorRT for inference.** Convert ONNX → TRT engine; expect 2–5× over plain PyTorch on Jetson / RTX.

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| RGB vs BGR mix between training and deploy | Policy outputs look "right" in sim, totally wrong colors at inference |
| Depth in mm vs meters | Off by 1000×; policy commands collisions or hovers far above target |
| Depth not aligned to color | Pixel-level lookup gives the wrong depth; grasps come up empty |
| Trusting `header.stamp` from a sensor that doesn't sync to system time | TF interpolation fails; "transform from past" errors |
| Using a single calibration across multiple physical cameras of "same model" | Per-unit lens variance is real; calibrate each one |
| Pointcloud-from-depth on every frame for visualization only | CPU pegged; control loop misses deadlines |

## When you also need

- `ros2-patterns` — for launch / namespacing of perception nodes.
- `ros2-node-discipline` — for QoS choices (sensor topics → `BEST_EFFORT`).
- `manipulation-primitives` — when perception output drives a grasp.
