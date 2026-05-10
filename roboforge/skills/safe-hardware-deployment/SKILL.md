---
name: safe-hardware-deployment
description: Use when about to run a controller, policy, trajectory, or any code path that sends commands to a physical actuator (real arm, gripper, mobile base, humanoid, IMU-driven joint). Required before any motion command, regardless of how "safe" the motion seems. Skipping this is how robots break and people get hurt.
---

<EXTREMELY-IMPORTANT>
The robot does not know whether the human in the room is paying attention. The robot does not know whether the workspace was cleared. The robot does not know whether the previous trajectory left a tool in its path. You have to know these things, and you have to encode them as physical and software guards before commanding any motion. Treat every "small" motion as a real one. Most robot incidents happen during "I just want to test something quick".
</EXTREMELY-IMPORTANT>

## When this skill applies

Invoke this skill before:
- Sending any command to a real motor, joint, or end-effector.
- Running a policy on real hardware (even one that "worked yesterday").
- Replaying a recorded trajectory on real hardware.
- Switching from simulation to real-hardware mode in a config or launch file.
- Bringing up a robot from a power-cycled state.
- Running teleop on a real robot.

This skill does NOT apply to:
- Sim-only runs (use `sim2real-checklist` when crossing into real).
- Reading sensors / cameras with no actuation.
- Building / compiling code that does not run.

## Required checklist

Walk every item with the user. Do not proceed to motion until every item is acknowledged.

1. **E-stop is physically wired and tested.** Press the e-stop right now (or have the user do it). Verify motors actually disengage. A wired-but-untested e-stop is not an e-stop.
2. **Software safety limits set below datasheet maxes.** For first runs, target ≤50% joint velocity, ≤30% torque/effort, ≤70% workspace volume. Limits live in the controller config or in code that wraps the policy output, not just in the user's head.
3. **Workspace bounds enforced in code.** End-effector position is clamped or the trajectory is rejected if it would exceed the defined safe volume. Tabletop arms: define table extents and a vertical floor. Mobile bases: define a virtual fence.
4. **The exact trajectory has run in simulation without violating limits.** "Same controller, same model, same config" — and the sim run logged max joint velocity, max effort, min collision distance, and they're all within limits.
5. **Full state logging is on.** Joint state, command, sensor data, and (for policies) inference inputs are logged to disk at ≥100 Hz. If something goes wrong you need this to debug.
6. **A second person is present, OR the robot is in a confined safe zone.** Lone-operator runs only after the first dozen successful supervised runs.
7. **First run uses a slow scaling factor.** Time-scale the trajectory ≥3× slower than target, OR scale velocity/torque to ~30%. Confirm behavior. Then scale up.
8. **Explicit rollback plan exists.** If motion goes wrong: who hits e-stop, who power-cycles the robot, who pulls cables. Verbalize this before commanding motion.

## Red flags — these thoughts mean STOP

| Thought | Reality |
|---|---|
| "It's just a small motion" | Small motions hit things at full speed when offsets are wrong. |
| "We tested in sim, it's fine" | Sim/real divergence is the most common failure mode. Treat real as a different system. |
| "The e-stop is right there" | Untested e-stops fail. Test it now. |
| "I'll set the limits after this run" | The first run is the run most likely to crash. Set them first. |
| "I'll log later" | If something breaks now, you have nothing to debug with. Log first. |
| "The user said it's safe" | Confirm what specifically — workspace cleared? second operator? scaling factor? |
| "We do this every day" | Familiarity is the most common precursor to incidents. Run the checklist anyway. |

## After the run

- Save the log immediately. Tag it with the policy/controller commit hash.
- If anything unexpected happened (jerk, near-collision, oscillation): stop. Don't assume the next run will go better. Diagnose first.
- If the run succeeded: incrementally relax scaling toward target speeds, one factor at a time.
