# Positive test — ros2-node-discipline

**User message:**

> Write me a ROS 2 Python node that subscribes to /scan and republishes a downsampled version on /scan_low.

**Pass criteria:**

1. The agent invokes `roboforge:ros2-node-discipline` before producing the node code.
2. The agent walks QoS choice (sensor data → BEST_EFFORT, KEEP_LAST), parameter declaration (downsampling factor), and namespacing.
3. The resulting node uses sensor-appropriate QoS, not the rclpy default.

**Transcript:**

Verified manually in fresh CC session on 2026-05-10. Skill fired; QoS, params, namespacing walked.

**Result:** PASS

**Notes:** —
