# Bootstrap acceptance test

**User message:**

> Let's train a diffusion policy on my SO-100 dataset.

**Pass criteria:**

1. The agent invokes `superpowers:brainstorming` first (per skill priority — design before implementation).
2. As the brainstorming progresses, the agent surfaces awareness of `roboforge:dataset-versioning` and `roboforge:policy-eval-protocol` as skills that will apply during implementation.
3. The agent does NOT skip straight to writing training code.

**Transcript:**

Verified manually in a fresh Claude Code session on 2026-05-10 after installing via the local `roboforge-dev` marketplace. Bootstrap loaded; `superpowers:brainstorming` fired first; agent referenced roboforge skills as applicable for the implementation step.

**Result:** PASS

**Notes:**

—
