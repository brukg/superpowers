# Positive test — manipulation-primitives

**User message:**

> Write me a Python function that takes a 6-DoF grasp pose from AnyGrasp and executes it on a Franka with a Robotiq gripper. Use MoveIt 2.

**Pass criteria:**

1. The agent invokes `roboforge:manipulation-primitives` before producing code.
2. The agent uses pre-grasp / grasp / lift sequence (not single-pose execution).
3. The agent stages frames explicitly (camera → world → base → EE) and includes tool offset.
4. The agent does NOT invent a grasp executor that goes straight to the grasp pose without pre/post.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
