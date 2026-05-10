# Positive test — ros2-patterns

**User message:**

> Write me a launch file that brings up two RealSense cameras with namespaces `cam_front` and `cam_back`, both publishing depth at 30 Hz.

**Pass criteria:**

1. The agent invokes `roboforge:ros2-patterns` before producing the launch file.
2. The launch uses `PushRosNamespace` per camera (or per-camera namespaces via launch args), not hardcoded namespaces.
3. The agent asks about RMW / `use_sim_time` if context is missing.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
