---
name: isaac-lab-workflows
description: Use when defining a task in Isaac Lab, building an env wrapper, training an RL policy on it, or tuning sim parameters (friction, mass, randomization). Pair with `sim2real-checklist` before deploying anything trained here on real hardware.
---

Reference for Isaac Lab — NVIDIA's GPU-parallelized RL training stack on top of Isaac Sim. Use when working in `IsaacLab/source/isaaclab_tasks/`, training an RL policy, or tuning sim assumptions.

## Three flavors of task

Isaac Lab tasks come in three styles. Pick deliberately.

| Flavor | When to use | Key file |
|---|---|---|
| **Manager-based** | Standard RL tasks; many configurable scene/observation/reward terms | `<task>_env_cfg.py` with `SceneCfg`, `ObservationsCfg`, `RewardsCfg`, `TerminationsCfg` |
| **Direct** | Custom step logic, fine control over physics tick / observation timing | Subclass `DirectRLEnv` or `DirectMARLEnv` |
| **Mimic / Teleop** | Imitation learning, human demos | `IsaacLabMimic`, plus a teleop config |

Manager-based is the default — you usually want it. Go Direct when you need step-level control or non-standard observation timing.

## Manager-based env skeleton

```python
@configclass
class CartpoleEnvCfg(DirectRLEnvCfg):
    # env
    episode_length_s = 5.0
    decimation = 2  # sim ticks per env step
    action_space = 1
    observation_space = 4
    state_space = 0

    # sim
    sim: SimulationCfg = SimulationCfg(dt=1/120, render_interval=decimation)

    # robot
    robot_cfg: ArticulationCfg = CARTPOLE_CFG.replace(prim_path="/World/envs/env_.*/Robot")

    # scene
    scene: InteractiveSceneCfg = InteractiveSceneCfg(num_envs=4096, env_spacing=4.0, replicate_physics=True)
```

`num_envs=4096` is the killer feature — you train on thousands of envs in parallel on one GPU.

## RL framework choice

Isaac Lab plays with several runners:

| Runner | When to use |
|---|---|
| **RSL-RL** | Default for locomotion / contact-heavy tasks. PPO. Battle-tested. |
| **SKRL** | More algorithms (SAC, TD3, PPO+). Cleaner abstractions. |
| **rl_games** | Original NVIDIA runner; PPO. Slowly being deprecated in Isaac Lab. |
| **Stable-Baselines3** | Single-env compatibility; mostly for sanity checks. |

Default to RSL-RL for locomotion and SKRL for everything else.

## Domain randomization

Randomize at construction (per-env) and per-episode. Both are first-class.

```python
events: EventCfg = EventCfg()

@configclass
class EventCfg:
    physics_material = EventTerm(
        func=mdp.randomize_rigid_body_material,
        mode="reset",
        params={"asset_cfg": SceneEntityCfg("robot", body_names=".*"),
                "static_friction_range": (0.7, 1.3),
                "dynamic_friction_range": (0.5, 1.0),
                "restitution_range": (0.0, 0.0)},
    )
    add_payload = EventTerm(
        func=mdp.randomize_rigid_body_mass,
        mode="startup",
        params={"asset_cfg": SceneEntityCfg("robot", body_names="base"),
                "mass_distribution_params": (-1.0, 1.0),
                "operation": "add"},
    )
```

**Modes:** `startup` (once at env construction), `reset` (every episode reset), `interval` (timed during episode).

**Don't over-randomize.** Wide ranges hurt convergence. Start narrow, widen if sim2real fails.

## Asset paths

USD assets live under `Isaac/Robots/...`. Use `ISAACLAB_NUCLEUS_DIR` and the `ArticulationCfg` helpers; don't hardcode absolute paths.

## Common gotchas

| Gotcha | What actually happens |
|---|---|
| Setting `decimation` without recomputing reward scale | Per-step reward changes magnitude; PPO clips weirdly |
| Forgetting `replicate_physics=True` in scene | Per-env physics state diverges; perf drops 5–10× |
| Using `randomize_rigid_body_material` with `asset_cfg=robot` (no `body_names`) | Matches all bodies including non-articulated; can crash |
| Training on `num_envs=4096` then deploying single-env | Different observation normalization scaling silently |
| `sim.use_fabric=True` with custom contact sensors | Fabric path may not surface contact data; switch to Direct env or disable Fabric |
| Mixing physics dt and rendering dt | RGB lag; observations stale by N sim ticks |

## Sim2real bridge

Isaac Lab → real flow:
1. Train with domain randomization in Isaac Lab.
2. Export the policy (TorchScript or ONNX).
3. Wrap with the same observation pipeline (`IsaacLab.envs` mirror) at deploy.
4. Run `sim2real-checklist` before the first real run.
5. Run `safe-hardware-deployment` for the actual motion.

## When you also need

- `sim2real-checklist` — before deploying.
- `manipulation-primitives` — when the task involves grasping / contact-rich manipulation.
- `humanoid-wbc` — for humanoid locomotion / WBC tasks.
