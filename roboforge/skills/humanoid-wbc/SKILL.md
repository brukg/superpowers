---
name: humanoid-wbc
description: Use when working on humanoid whole-body control, motion retargeting (MoCap → robot, SMPL → robot), or humanoid policy pipelines (GR00T-WBC, OmniH2O, ASAP, HumanPlus). Surfaces the concepts and stacks; pair with `manipulation-primitives` for arm-only humanoid work.
---

Reference for humanoid whole-body control (WBC) and motion retargeting. Distinct from arm-only manipulation because of contact constraints (feet, hands), balance, and the need to coordinate locomotion + manipulation.

## Whole-body control concepts

WBC = control all DoF (legs + arms + torso) jointly under contact and balance constraints.

**Key constraints (hard, must hold):**
- **Contact stability**: foot/hand contact forces inside the friction cone.
- **CoM in support polygon** (or extended polygon for dynamic gaits).
- **Joint and torque limits**.

**Key tasks (soft, optimized):**
- **CoM trajectory** (where the body should go).
- **End-effector trajectories** (hand/foot poses).
- **Posture regularization** (stay near a "natural" pose).

Implementations solve a QP every control step (~500–1000 Hz) that balances tasks against constraints.

## Common WBC stacks

| Stack | Origin | What it solves | Status |
|---|---|---|---|
| **OCS2** | ETH Zürich | MPC + WBC for legged | Mature, well-documented |
| **TSID** | LAAS-CNRS | Task-space inverse dynamics QP | Reference impl; small footprint |
| **Pinocchio + osqp** | Roll-your-own | Custom QPs over rigid-body library | When TSID/OCS2 don't fit |
| **PromisedLand / cube-mpc** | Various | High-rate MPC for humanoids | Research-grade |

For most humanoid policy work today, the WBC layer is **embedded inside the RL policy** — the policy outputs joint targets and the constraints are enforced via reward shaping + termination. The "hand-designed WBC" stacks are mostly used at the low-level controller below the policy.

## Motion retargeting

Two big sources of humanoid demo data:

1. **MoCap (AMASS, OptiTrack, Vicon, MoCapAct)** — joint angles in SMPL or BVH skeleton.
2. **Video-based** (Hamer for hands, MotionBERT, GVHMR for body) — 3D pose estimates from video.

Both produce a **source skeleton** that doesn't match the **target robot skeleton** (different segment lengths, joint counts, joint-axis conventions). Retargeting fixes that.

**Retargeting stacks:**
- **PHC (Perpetual Humanoid Control)** — MoCap → humanoid via RL with imitation reward.
- **GMR (General Motion Retargeting)** — geometric retargeting + correction; works for most humanoid robots.
- **Mink + Pink** — IK-based retargeting using MuJoCo MJX.
- **HumanoidVerse / TWIST / TWIST2** — task-conditioned retargeting and tracking pipelines.
- **ProtoMotions** — physics-based motion tracking + control on humanoids.

**Watch out for:**
- **Skeleton mismatch.** SMPL has 24 joints; Unitree H1 has 19; H1-2 has 27. Joint mapping is non-trivial.
- **Foot-skating.** Source mocap on a flat floor; target robot has different foot geometry. Apply foot-IK correction.
- **Self-collision.** Source skeleton allows poses target robot can't physically achieve. Filter or project.
- **Frame conventions.** SMPL Z-up vs ROS Z-up vs robot base frame. Stage transforms explicitly.

## Humanoid policy pipelines (current SOTA)

| Pipeline | Approach | When to consider |
|---|---|---|
| **GR00T (N1, N2, WholeBodyControl)** | NVIDIA VLA + WBC, Isaac Lab native | If you're on NVIDIA stack and want a foundation model |
| **OmniH2O** | Sim2real teleop and learning, retargeted MoCap | Strong baseline for teleop-driven imitation |
| **HumanPlus** | Imitation from human video + RL fine-tune | Strong for short-horizon manipulation skills |
| **ASAP** | Aligning sim and real for humanoid skills | When sim2real gap is large; alignment is the focus |
| **TWIST / TWIST2** | Whole-body teleop and tracking | When teleop demos drive the data pipeline |
| **HumanoidVerse** | Library of humanoid tasks + tracking baselines | Research baselines on Unitree / H1 |

## Action chunking for humanoids

Humanoid action spaces are large (~30+ DoF). Two consequences:

1. **Predict longer chunks** (16–64 steps) than for arms. The policy's effective horizon needs to cover at least one half-gait cycle.
2. **Decouple low-frequency upper body from high-frequency lower body.** Common pattern: policy at 30 Hz commands joint targets; PD controller at 1 kHz tracks them with stiffness/damping tuned per limb.

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| Treating humanoid as "two arms + a head" | Lose balance constraints; first deploy falls over |
| Retargeting MoCap once, training many policies on it | Retarget errors compound; redo retargeting per-platform |
| Naive joint-target output without limits | Saturate motors; thermal trip, joint damage |
| Using arm-only sim envs to train humanoid policies | Action space mismatch; policies don't generalize |
| Forgetting foot contact in the policy reward | Policy lifts off the ground for free reward |

## When you also need

- `isaac-lab-workflows` — for sim training of humanoid policies.
- `manipulation-primitives` — for the arm-side of bimanual tasks.
- `safe-hardware-deployment` — before any humanoid hardware run (especially: free-space evaluations need crash mats).
- `vla-policies-overview` — when picking GR00T vs Pi0 vs custom for the upper-body policy.
