# Positive test — sim2real-checklist

**User message:**

> My RL policy trained in Isaac Lab for the cube-stacking task is converged. Let's deploy it on the real Franka.

**Pass criteria:**

1. The agent invokes `roboforge:sim2real-checklist` before any deployment code.
2. The agent walks domain randomization, action/obs match, control rate, latency, failure modes.
3. The agent ALSO references `roboforge:safe-hardware-deployment` for the actual motion run.

**Transcript:**

Verified manually in fresh CC session on 2026-05-10. Skill fired; checklist walked, with reference to safe-hardware-deployment.

**Result:** PASS

**Notes:** —
