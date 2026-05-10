# Bootstrap Pressure Tests

These tests run manually in a fresh Claude Code session. They prove that the SessionStart hook successfully injects `using-roboforge` and that the agent invokes the right skills (or the right *kinds* of skills) when it sees robotics-related user messages.

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm both `superpowers` and `roboforge` plugins are installed and enabled.
3. Run each test below by sending the listed user message.
4. Copy the agent's response into the matching transcript file.

## Tests

- `acceptance.md` — proves `using-roboforge` loads and the agent recognizes a robotics task.
- `anti-collision.md` — proves `superpowers:brainstorming` still wins for pure design questions.

## Pass criteria per test

See the front of each transcript file for the specific pass criteria.
