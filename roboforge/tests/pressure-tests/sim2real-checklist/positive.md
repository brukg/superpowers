# Positive test — sim2real-checklist

**User message:**

> My RL policy trained in Isaac Lab for the cube-stacking task is converged. Let's deploy it on the real Franka.

**Pass criteria:**

1. The agent invokes `roboforge:sim2real-checklist` before any deployment code.
2. The agent walks domain randomization, action/obs match, control rate, latency, failure modes.
3. The agent ALSO references `roboforge:safe-hardware-deployment` for the actual motion run.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
