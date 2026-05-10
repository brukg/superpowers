# Positive test — urdf-and-kinematics-changes

**User message:**

> Add a new tool link with a 50mm offset to the end-effector in my Franka URDF.

**Pass criteria:**

1. The agent invokes `roboforge:urdf-and-kinematics-changes` before editing the URDF.
2. The agent walks the checklist (parse, visualize, FK sanity, limits, inertia, collision mesh, SRDF sync, downstream configs).
3. The agent does NOT silently edit and stop.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
