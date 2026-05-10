# Roboforge — Robotics Skills Plugin (Design)

**Date:** 2026-05-10
**Status:** Design approved, awaiting implementation plan
**Author:** brukg + Claude Code

## Summary

`roboforge` is a domain-specific skills plugin for AI coding agents working on robotics — covering ROS 2, VLA / imitation-learning research, simulation (Isaac Lab / Sim), and real-hardware bring-up. It ships as a sibling plugin to Superpowers core (not a replacement) and is structured to install alongside it without name collisions.

The plugin is a **broad v1**: ~18 skills spanning methodology (auto-trigger before risky robotics work), domain knowledge (canonical patterns and gotchas), and scaffolding (generators for ROS 2 packages, LeRobot policies, Isaac tasks, training entrypoints, teleop pipelines). Process discipline (brainstorming, TDD, debugging) is delegated entirely to Superpowers core — roboforge does not duplicate it.

The plugin targets the same harness matrix as Superpowers (Claude Code, Codex CLI, Cursor, Gemini CLI, OpenCode, Factory Droid, Copilot CLI), reusing Superpowers' packaging machinery.

## Goals

1. Make a roboticist's coding agent reliably run the right pre-flight checks before touching real hardware, datasets, kinematics, or policy evaluations.
2. Surface canonical patterns for ROS 2, LeRobot, OpenVLA-family policies, Isaac Lab, manipulation, humanoid WBC, and perception — so the agent stops inventing wrong patterns from training data.
3. Cut scaffolding time for the recurring file shapes (ROS 2 pkg, LeRobot policy, Isaac task, training entrypoint, teleop loop).
4. Coexist cleanly with Superpowers core — no namespace collisions, no fighting bootstraps, no duplicated process skills.
5. Keep upstream Superpowers sync clean — no edits to `skills/` core directory or to `using-superpowers`.

## Non-Goals

- Not a full robotics framework. No runtime code, no Python package, no CLI.
- Not a debugging-recipe library. (Excluded by user choice; debugging is handled by `superpowers:systematic-debugging`.)
- Not a replacement for Superpowers core. Process skills (brainstorming, TDD, debugging, planning, parallel agents) stay in core.
- No support for ROS 1, JAX/TensorFlow, Mujoco/PyBullet, Gazebo Classic in v1. May be added later as variants.
- No automated CI for skill transcripts in v1. Pressure tests are human-judged.

## Default Tech Stack Assumptions

Skills are written assuming the SOTA / commonly-used stack:

- **ROS 2** Humble / Jazzy (rclpy/rclcpp, colcon, launch.py, lifecycle, ros2_control, MoveIt 2, Nav2, Gazebo Garden/Harmonic).
- **PyTorch + HuggingFace Hub**, with **LeRobot** dataset/policy conventions and **OpenVLA-family** models (OpenVLA, Octo, Pi0, RDT2, GR00T, ACT, diffusion policy).
- **Isaac Sim / Isaac Lab** for GPU sim and sim2real.
- **Real hardware**: Franka, UR, SO-100/101, Unitree, Aloha; RealSense / ZED cameras; IMUs; teleop (Quest, leader-follower).

## Architecture

### Plugin Layout

`roboforge` is a **nested sub-plugin** inside the existing Superpowers fork at `/home/phoenix/vla/superpowers/`. The fork's plugin name remains `superpowers` (preserves clean upstream sync). `roboforge` is a separate plugin in its own subdirectory.

```
/home/phoenix/vla/superpowers/
├── skills/                              ← upstream Superpowers core (untouched)
├── hooks/                               ← upstream Superpowers hooks (untouched)
├── .claude-plugin/plugin.json           ← upstream Superpowers manifest (untouched)
└── roboforge/                           ← NEW plugin
    ├── .claude-plugin/plugin.json       ← name: "roboforge"
    ├── .codex-plugin/plugin.json
    ├── .cursor-plugin/plugin.json
    ├── gemini-extension.json
    ├── package.json
    ├── AGENTS.md / CLAUDE.md / GEMINI.md
    ├── README.md
    ├── assets/
    ├── hooks/
    │   ├── hooks.json                   ← SessionStart → run-hook.cmd session-start
    │   ├── hooks-cursor.json
    │   ├── run-hook.cmd                 ← copied from superpowers/hooks/
    │   └── session-start                ← bash, emits roboforge bootstrap
    ├── skills/
    │   ├── using-roboforge/             ← bootstrap skill (analog of using-superpowers)
    │   ├── safe-hardware-deployment/
    │   ├── dataset-versioning/
    │   ├── policy-eval-protocol/
    │   ├── urdf-and-kinematics-changes/
    │   ├── sim2real-checklist/
    │   ├── ros2-node-discipline/
    │   ├── ros2-patterns/
    │   ├── lerobot-conventions/
    │   ├── vla-policies-overview/
    │   ├── isaac-lab-workflows/
    │   ├── manipulation-primitives/
    │   ├── humanoid-wbc/
    │   ├── perception-stack/
    │   ├── scaffold-ros2-package/
    │   ├── scaffold-lerobot-policy/
    │   ├── scaffold-isaac-task/
    │   ├── scaffold-training-entrypoint/
    │   └── scaffold-teleop-pipeline/
    ├── scripts/
    │   ├── bump-version.sh
    │   └── sync-to-codex-plugin.sh
    └── tests/
        └── pressure-tests/<skill>/      ← positive + negative transcript per skill
```

### Coexistence with Superpowers Core

- Plugin name `roboforge` (no collision with `superpowers`).
- All skill internal names are domain-prefixed or domain-specific (`safe-hardware-deployment`, never `brainstorming`).
- Bootstrap skill named `using-roboforge` (no collision with `using-superpowers`).
- `using-roboforge` explicitly defers process work to `superpowers:*` skills. Roboforge **adds to** core process discipline; it does not replace it.
- `roboforge/skills/` does not include any process / methodology-of-software-engineering skills (no brainstorming, no TDD, no debugging). These are owned by Superpowers core.

### Multi-Harness Packaging

Mirrors Superpowers' approach exactly. Each per-harness manifest at the plugin root, content sourced canonically from `roboforge/skills/`, derived copies generated by `scripts/sync-to-codex-plugin.sh` (and equivalents). No new harness mechanics are invented.

| Harness         | Manifest                           |
|-----------------|------------------------------------|
| Claude Code     | `.claude-plugin/plugin.json`       |
| Codex CLI / App | `.codex-plugin/plugin.json`        |
| Cursor          | `.cursor-plugin/plugin.json`       |
| Gemini CLI      | `gemini-extension.json` + `GEMINI.md` |
| OpenCode        | `package.json`                     |
| Factory Droid   | TBD — verify against Factory plugin docs during implementation |
| Copilot CLI     | `AGENTS.md`                        |

## Skill Catalog (v1)

Three categories, 18 skills total.

### Methodology / Discipline (auto-trigger before risky work) — 6

| Skill | Trigger | Enforces |
|---|---|---|
| `safe-hardware-deployment` | About to run a controller/policy/trajectory on real hardware | Safety envelope, e-stop wired and tested, joint/torque/velocity limits, workspace bounds, sim dry-run first, full-state logging |
| `dataset-versioning` | Before training, retraining, or appending episodes | Dataset version pin, episode validity check, action-space consistency, normalization stats recompute, train/val split, leakage check |
| `policy-eval-protocol` | Before claiming a policy works | Eval episode set defined, success metrics defined, seeds controlled, ID vs OOD eval, sim-eval gate before real-eval |
| `urdf-and-kinematics-changes` | Before editing URDF / Xacro / SRDF | FK/IK validation, joint limit sanity, mass/inertia sanity, collision mesh sanity, MoveIt SRDF sync |
| `sim2real-checklist` | Before deploying a sim-trained policy on real hardware | Domain-randomization audit, action/observation calibration, control-rate match, latency budget, failure-mode list |
| `ros2-node-discipline` | Creating or modifying a ROS 2 node | QoS by topic class (sensor / control / default), lifecycle states, parameters declared, action-vs-service-vs-topic decision |

### Domain Knowledge / Pitfalls (lookup-style, on demand) — 7

| Skill | Trigger | Content |
|---|---|---|
| `ros2-patterns` | Editing launch files, params, namespaces, DDS configs | launch.py composition, param overrides, remapping, DDS QoS gotchas, Humble vs Jazzy diffs |
| `lerobot-conventions` | Touching LeRobot datasets, policies, training | parquet+videos format, policy interface, hub patterns, training entrypoints |
| `vla-policies-overview` | Picking or comparing a VLA / IL policy | I/O contracts and trade-offs across OpenVLA, Octo, Pi0, RDT2, GR00T, ACT, diffusion policy |
| `isaac-lab-workflows` | Working in Isaac Sim / Lab | Task definition, env wrappers, RL training (RSL-RL / SKRL), domain randomization knobs |
| `manipulation-primitives` | Writing manipulation code | Pose conventions (T_world_ee), grasping pipelines, collision-free planning, suction vs parallel-jaw |
| `humanoid-wbc` | Working on humanoid control / retargeting | WBC patterns, balance constraints, MoCap retargeting, action-chunking for humanoids |
| `perception-stack` | Touching cameras / depth / time-sync | RealSense / ZED calibration, stereo gotchas, time-sync with IMU/joints, GPU pipelines |

### Scaffolding / Generators (on request) — 5

| Skill | Generates |
|---|---|
| `scaffold-ros2-package` | colcon-friendly Python or C++ pkg with launch, params, ament deps |
| `scaffold-lerobot-policy` | Minimal trainable policy class wired to LeRobot's interfaces |
| `scaffold-isaac-task` | Isaac Lab task: env config, observations, rewards, termination |
| `scaffold-training-entrypoint` | Hydra config layout + training loop with W&B logging, checkpointing, resume |
| `scaffold-teleop-pipeline` | Leader-follower or Quest teleop loop with record-to-LeRobot-dataset |

### Known overlaps to revisit in v2

- `humanoid-wbc` and `manipulation-primitives` both touch retargeting; may merge or split.
- `perception-stack` is broad; may split into calibration / time-sync / depth in v2.

## Auto-Trigger Strategy

### Bootstrap (`using-roboforge`)

- Loaded at `SessionStart` (matchers: `startup|clear|compact`) via `roboforge/hooks/hooks.json`, identical hook shape to `superpowers/hooks/hooks.json`.
- Tells the agent that roboforge skills exist and to invoke `Skill` *before* doing robotics work.
- Explicitly defers all process/methodology work to `superpowers:*`.

### Trigger-Phrase Convention

Each skill's `description:` frontmatter follows Superpowers' tested pattern: "Use when..." + concrete signals. Examples:

- `safe-hardware-deployment` → "Use when about to run a controller, policy, or trajectory on physical hardware (real arm, gripper, mobile base, humanoid). Required before any code path that sends commands to a real motor, regardless of how 'safe' the motion seems."
- `lerobot-conventions` → "Use when reading, writing, or transforming LeRobot datasets; defining a LeRobot policy class; or wiring LeRobotDataset into a training loop."
- `ros2-node-discipline` → "Use when creating a new ROS 2 node, adding publishers/subscribers, or changing QoS / lifecycle / parameter declarations."

### Priority vs. Superpowers Core

1. `superpowers:brainstorming` runs first when the user proposes new robotics work.
2. `superpowers:test-driven-development` and `superpowers:systematic-debugging` keep their normal precedence.
3. Roboforge methodology skills fire **after** the relevant process skill but **before** the implementation step they govern.
4. Roboforge knowledge / scaffold skills fire on demand.

### Anti-Collision

`using-roboforge` instructs: "If a Superpowers core skill applies, use it. Roboforge skills add to, not replace, core process discipline." Prevents the failure mode where two bootstraps fight for control.

## Testing & Validation

Behavior-shaping content needs adversarial transcripts, not unit tests. Same model Superpowers core uses.

### 1. Per-Skill Pressure Tests

`roboforge/tests/pressure-tests/<skill>/`. For each skill, two transcripts:

- **Positive**: A user message that should auto-trigger the skill. Pass = skill is invoked AND its checklist is followed.
- **Negative**: A user message that *looks* related but should not trigger. Pass = skill is NOT invoked.

### 2. Bootstrap Acceptance Test

Clean session in each target harness, user message:

> Let's train a diffusion policy on my SO-100 dataset.

Pass: `superpowers:brainstorming` fires first; once design is settled, `roboforge:dataset-versioning` and `roboforge:policy-eval-protocol` surface at the right moments.

### 3. Anti-Collision Test

Clean session. User message:

> Help me brainstorm a new manipulation skill.

Pass: `superpowers:brainstorming` fires (NOT a roboforge skill).

### 4. Cross-Harness Packaging Test

After running the sync script, a diff script confirms Codex / Cursor / Gemini copies have identical skill content to the canonical `roboforge/skills/`.

### Definition of Shippable v1

- All 18 skills have positive + negative transcripts that pass on Claude Code.
- Bootstrap acceptance test passes on Claude Code + at least one other harness.
- Anti-collision test passes.

## Build Sequence (high level — detailed plan comes from writing-plans)

1. **Scaffolding** — create `roboforge/` subtree, manifests for all harnesses, hooks, bootstrap skill `using-roboforge`. Verify on Claude Code that the bootstrap loads at SessionStart.
2. **Methodology skills first** — these are the highest-leverage and the hardest to tune. Build, pressure-test, iterate one at a time.
3. **Knowledge skills** — bulk-author with shared review for consistency.
4. **Scaffold skills** — produce templates, validate generated artifacts compile / lint / pass smoke tests.
5. **Multi-harness validation** — run sync script, run acceptance + anti-collision tests on a second harness.

## Open Questions for Implementation Phase

- Which second harness to validate against for v1 (Codex CLI is closest in behavior to CC; Gemini is the most behaviorally distinct).
- Whether `scaffold-isaac-task` should target Isaac Lab's task-based API or the gym-style env API (likely Isaac Lab task API, but worth confirming during implementation).
- Whether `vla-policies-overview` lives as one large skill or as a small index pointing to per-policy sub-references.

These are deferred to the writing-plans phase.
