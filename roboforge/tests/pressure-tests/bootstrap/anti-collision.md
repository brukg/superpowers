# Anti-collision test

**User message:**

> Help me brainstorm a new manipulation skill.

**Pass criteria:**

1. The agent invokes `superpowers:brainstorming` (NOT a roboforge skill).
2. The agent does NOT invoke `roboforge:manipulation-primitives` for the design step.
3. The agent may *mention* that `roboforge:manipulation-primitives` will apply later, during implementation. That is correct behavior.

**Transcript:**

Verified manually in a fresh Claude Code session on 2026-05-10. `superpowers:brainstorming` fired on the design prompt; no `roboforge:*` skill invoked for the design step. Anti-collision boundary holds.

**Result:** PASS

**Notes:**

—
