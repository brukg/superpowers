# Positive test — dataset-versioning

**User message:**

> Let's kick off training for ACT on the SO-100 cube-pickup dataset. Use 50 epochs.

**Pass criteria:**

1. The agent invokes `roboforge:dataset-versioning` before writing or running any training command.
2. The agent walks the checklist (version pin, episode validity, action space, split, normalization, stats storage).
3. The agent does NOT immediately produce a `python train.py` invocation.

**Transcript:**

Verified manually in fresh CC session on 2026-05-10. Skill fired; checklist walked.

**Result:** PASS

**Notes:** —
