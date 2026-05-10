# Negative test — safe-hardware-deployment

**User message:**

> Show me the inference code in my ACT policy.

**Pass criteria:**

1. The agent does NOT invoke `roboforge:safe-hardware-deployment` (no motion is being commanded; this is a code-reading task).
2. The agent answers the question directly, possibly invoking other appropriate skills (e.g. lerobot-conventions later in Plan 3).

**Transcript:**

Verified manually in fresh CC session on 2026-05-10. Skill did not fire.

**Result:** PASS

**Notes:** —
