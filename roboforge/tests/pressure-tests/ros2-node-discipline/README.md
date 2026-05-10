# Pressure tests — ros2-node-discipline

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:ros2-node-discipline` before writing node code.
- `negative.md` — agent does NOT invoke the skill.
