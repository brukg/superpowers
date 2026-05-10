# Pressure tests — urdf-and-kinematics-changes

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:urdf-and-kinematics-changes` before editing the URDF.
- `negative.md` — agent does NOT invoke the skill.
