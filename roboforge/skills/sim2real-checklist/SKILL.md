---
name: sim2real-checklist
description: Use when about to deploy a sim-trained policy on real hardware for the first time, or after any retraining that changes sim assumptions (action space, observation, control rate, randomization). Bridges the gap that causes most "worked in sim, broken on real" incidents. Combine with safe-hardware-deployment for the actual motion run.
---

Sim2real failures are usually predictable in retrospect: a control rate that didn't match, an action unit that wasn't converted, a camera FOV that wasn't randomized, a delay that wasn't modeled. This checklist surfaces them before the real arm moves.

## When this skill applies

Invoke this skill before:
- The first real-hardware deployment of a sim-trained policy.
- Re-deploying after retraining with changed sim assumptions.
- Switching from one real platform to another (e.g. sim trained for UR5e, deploying on UR10e).
- Any time you change the action or observation interface in real without retraining.

This skill does NOT apply to:
- Sim-only iteration.
- Real-only iteration with no sim component (use `safe-hardware-deployment` directly).

After this checklist, you must ALSO invoke `safe-hardware-deployment` for the actual motion run.

## Required checklist

1. **Domain randomization audit.** What was randomized in training? Lighting, textures, camera pose, friction, mass, gripper geometry, distractor objects, robot base offset. List the ranges. Anything NOT randomized is a sim-real assumption you'd better hold.
2. **Action space matches.** Same dimension order, same units (rad vs deg, m vs mm, normalized vs absolute), same scaling (delta vs absolute). If conversion is needed, the conversion lives at deployment, not in user's head. Test the conversion with a static command before policy rollout.
3. **Observation space matches.** Image resolution, format (RGB vs BGR), value range (0–1 vs 0–255), normalization (per-channel mean/std), camera intrinsics, FOV. Joint encoders: same units, same sign convention, same zero configuration.
4. **Control rate matches.** If the sim ran the policy at 30 Hz and real runs at 50 Hz, you're commanding 1.67× faster trajectories. Either match the rate, or interpolate, and document which.
5. **Latency budget.** Measure: camera capture → preprocessing → inference → action send → actuator response. Sum it. Compare to the policy's assumed latency (often near-zero). If real latency is materially higher, expect lag-induced oscillation. Mitigations: action chunking horizon, predictive offset, lower control rate.
6. **Failure-mode list.** Write down at least three failure modes you expect: "policy commands out-of-workspace pose if cube is hidden", "gripper opens early if depth is noisy", "policy oscillates near goal pose". For each, write the graceful response (clamp, hold, slow down).
7. **First real run uses `safe-hardware-deployment` defaults.** Slow scaling, full logging, second person, e-stop tested. Do not skip.

## Red flags

| Thought | Reality |
|---|---|
| "Sim works, deploy" | Sim worked under sim assumptions. Confirm those hold on real. |
| "It's the same robot model" | Same model ≠ same calibration, same tool, same payload. Check. |
| "Latency is small enough to ignore" | Measured? Or assumed? |
| "We'll handle failures with e-stop" | E-stop is the last resort. Define graceful failures upstream. |
| "Action chunking absorbs latency" | Up to a point. Beyond that, oscillation. Measure first. |

## After the run

- Log the deployment delta: what was different between sim and real (control rate, calibration offsets, latency, anything you tweaked). Save it next to the checkpoint.
- If the run failed: do NOT just retry. Diagnose which checklist item was the cause. Most "weird sim2real failures" are item 3 (observation mismatch) or item 5 (latency).
- Document the gap so the next deployment of a similar policy starts ahead.
