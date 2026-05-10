---
name: manipulation-primitives
description: Use when writing manipulation code — grasping, pick-and-place, approach/retreat motions, gripper control, frame conversions. Surfaces the conventions and primitives that fail silently when not stated explicitly. Do NOT use for ROS 2 plumbing — that's `ros2-patterns` / `ros2-node-discipline`.
---

Reference for the building blocks of pick-and-place and contact-rich manipulation. Most "the gripper went to the wrong spot" bugs are frame errors or pre/post-grasp pose errors — both addressed here.

## Pose conventions

Always state the frame. Always.

| Notation | Meaning |
|---|---|
| `T_world_ee` | Transform from `world` frame to `end_effector` frame. Pose of EE expressed in world. |
| `T_base_ee` | Pose of EE in robot base frame. The thing IK consumes. |
| `T_ee_tool` | Tool offset (e.g. parallel-jaw center, suction-cup tip). Composes with `T_base_ee`. |
| `T_world_obj` | Object pose in world. From perception. |
| `T_obj_grasp` | Grasp pose in object frame. From a grasp predictor. |
| `T_world_grasp` | Goal: `T_world_obj @ T_obj_grasp`. Convert to `T_base_ee` before IK. |

**Right-hand rule.** Gripper +Z usually points along approach direction (out of the palm). +X along jaw-opening axis. Confirm per gripper.

**Quaternions vs Euler.** Always quaternions in code. Euler only for human display. Half the manipulation bugs are Euler-singularity bugs at gimbal lock.

## Grasp pipelines

Three common pipelines:

1. **Predict 6-DoF grasp directly** (GraspNet, AnyGrasp, Contact-GraspNet). Input: point cloud. Output: list of `(T_world_grasp, score, gripper_width)`. Filter by reachability + collision. Pick best.
2. **Detect object → look up grasp annotation** (when objects are known: cups, boxes, tools). Object-frame grasps are pre-defined.
3. **Learned policy outputs gripper pose directly** (ACT, Diffusion Policy, OpenVLA). No explicit grasp prediction; the policy learns it. This is what VLAs do.

For pipelines 1 and 2, the agent code orchestrates: detect → predict → filter → execute. For pipeline 3, the policy is the orchestrator; you only wire I/O.

## Approach / retreat motion

A grasp is **three** poses, not one:

1. **Pre-grasp pose** — `T_world_grasp` offset along -approach axis by ~10 cm. The arm moves freely here.
2. **Grasp pose** — `T_world_grasp`. Closed-loop or slow-Cartesian motion from pre-grasp.
3. **Post-grasp / lift** — `T_world_grasp` offset along +world Z by ~5–10 cm. Lift before transit.

Skipping the pre-grasp = collision risk. Skipping the lift = drag the object across the table.

For placement, mirror: pre-place above target, place, retract.

## Gripper choice

| Gripper type | Best at | Watch out for |
|---|---|---|
| Parallel-jaw (e.g. Robotiq 2F-85, Franka Hand) | Rigid objects, known geometry | Width mismatch; soft objects squish |
| Suction (e.g. Robotiq EPick) | Flat-top objects (boxes, planar parts) | Porous / curved surfaces fail silently; needs vacuum check |
| Multi-finger (e.g. Allegro, Shadow) | Dexterous, in-hand manipulation | Calibration / wear; teleop costly |
| Soft / under-actuated (e.g. SoftGripper) | Cluttered scenes, unknown shapes | Position feedback poor; force control limited |

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| Predicting grasps in camera frame, executing in world frame | Robot grasps a foot to the side; perception/exec frame mismatch |
| Using `T_base_ee` from FK as `T_world_ee` (no base transform) | Off by the base-in-world offset; common in mobile manipulators |
| Forgetting tool offset `T_ee_tool` | Suction cup tip is 5–8 cm past wrist flange; grasps come up short |
| Closing gripper width = object width exactly | No grasp force; thermal expansion + slop = drop |
| Hand-coding pre-grasp = grasp - 10 cm in world Z | Wrong for side grasps. Offset along the *approach* axis, not world Z. |
| Re-planning every 1 ms during slow grasp motion | Planner thrashes; jitter |

## Decision: motion planning vs servoing

| Use motion planning (MoveIt 2, OMPL) | Use servoing (Cartesian, MoveIt Servo, custom controller) |
|---|---|
| Long-distance free-space motion | Short-distance reactive motion |
| Known goal pose, pre-grasp / pre-place | Visual-servoing onto a moving target |
| Need collision avoidance with environment | Need closed-loop response to live perception |
| Acceptable to wait 100–500 ms for plan | Need ≥10 Hz update |

Most "policy" code uses servoing (the policy IS the controller). Most "scripted" pick-and-place uses motion planning for transit + servoing for fine alignment.

## When you also need

- `urdf-and-kinematics-changes` — if you're modifying the gripper / tool URDF.
- `ros2-node-discipline` — when wiring the manipulation node.
- `safe-hardware-deployment` — before any real-arm motion.
- `perception-stack` — when grasp inputs come from cameras.
