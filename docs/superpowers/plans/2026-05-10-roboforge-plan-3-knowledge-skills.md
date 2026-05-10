# Roboforge Plan 3 of 5 — Knowledge Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the seven roboforge knowledge skills that surface canonical patterns and pitfalls when the agent is working on robotics code — `ros2-patterns`, `lerobot-conventions`, `vla-policies-overview`, `isaac-lab-workflows`, `manipulation-primitives`, `humanoid-wbc`, `perception-stack`.

**Architecture:** Knowledge skills differ from methodology skills (Plan 2): they surface *information* when the agent is doing the work, not gates that block until a checklist is walked. Each `SKILL.md` is a reference document — quick lookup tables, decision trees, common pitfalls — written so the agent reads it once and writes better code afterward. Same packaging shape as Plan 2.

**Tech Stack:** Markdown only.

**Reference spec:** `docs/superpowers/specs/2026-05-10-roboforge-plugin-design.md`

**Depends on:** Plans 1 and 2 (foundation + methodology skills already shipping).

---

## File Structure

| Path | Responsibility |
|---|---|
| `roboforge/skills/ros2-patterns/SKILL.md` | launch.py, params, namespacing, DDS, Humble vs Jazzy notable diffs |
| `roboforge/skills/lerobot-conventions/SKILL.md` | LeRobot dataset format, policy interface, hub patterns, training entrypoints |
| `roboforge/skills/vla-policies-overview/SKILL.md` | OpenVLA / Octo / Pi0 / RDT2 / GR00T / ACT / Diffusion Policy — when to pick which, how they differ |
| `roboforge/skills/isaac-lab-workflows/SKILL.md` | Task definition, env wrappers, RL training (RSL-RL/SKRL), domain rand |
| `roboforge/skills/manipulation-primitives/SKILL.md` | Pose frames, grasping pipelines, approach/retreat, gripper choice |
| `roboforge/skills/humanoid-wbc/SKILL.md` | WBC concepts, MoCap retargeting, humanoid stacks |
| `roboforge/skills/perception-stack/SKILL.md` | Camera calibration, RealSense / ZED launch, time-sync, GPU pipelines |
| `roboforge/tests/pressure-tests/<skill>/{README,positive,negative}.md` (×7) | Per-skill pressure tests |

No existing files modified.

---

## Task 1: `ros2-patterns`

**Files:**
- Create: `roboforge/skills/ros2-patterns/SKILL.md`
- Create: `roboforge/tests/pressure-tests/ros2-patterns/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/ros2-patterns \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/ros2-patterns
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/ros2-patterns/SKILL.md`:

````markdown
---
name: ros2-patterns
description: Use when editing ROS 2 launch files, parameter YAMLs, namespaces, remappings, DDS configs, or asking "what's the right ROS 2 way to do X". Surfaces canonical patterns and Humble/Jazzy differences. Do NOT use for runtime node implementation — that's `ros2-node-discipline`.
---

Reference for ROS 2 system-wiring patterns. Use when laying out a launch graph, configuring parameters, deciding on namespacing, or debugging "why don't my topics talk". For node-internals decisions (QoS choice, lifecycle, action-vs-service), use `ros2-node-discipline`.

## Launch.py composition

ROS 2 launch is Python. Compose, don't copy-paste.

**Pattern: include + override.** A package exposes a top-level launch file that includes others and overrides args.

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription, DeclareLaunchArgument
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration, PathJoinSubstitution
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    use_sim = LaunchConfiguration('use_sim')
    return LaunchDescription([
        DeclareLaunchArgument('use_sim', default_value='false'),
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(PathJoinSubstitution([
                FindPackageShare('my_robot_bringup'), 'launch', 'sensors.launch.py'
            ])),
            launch_arguments={'use_sim': use_sim}.items(),
        ),
    ])
```

**Pattern: composable nodes.** For nodes that share a process (zero-copy intra-process comms), use `ComposableNodeContainer`. Critical for high-frequency sensor pipelines.

```python
from launch_ros.actions import ComposableNodeContainer
from launch_ros.descriptions import ComposableNode
container = ComposableNodeContainer(
    name='perception_container',
    namespace='',
    package='rclcpp_components',
    executable='component_container_mt',  # multi-threaded
    composable_node_descriptions=[
        ComposableNode(package='image_proc', plugin='image_proc::DebayerNode', name='debayer'),
        ComposableNode(package='image_proc', plugin='image_proc::RectifyNode', name='rectify'),
    ],
)
```

## Parameters

Resolution order (highest first): CLI override → param file → node default in `declare_parameter`.

**YAML file shape:**
```yaml
/my_node:
  ros__parameters:
    update_rate_hz: 50.0
    target_frame: "base_link"
    use_sim_time: false
```

Pass with `parameters=[<yaml_path>, {'extra_param': value}]` in launch.

Per-namespace overrides:
```yaml
/robot1/my_node:
  ros__parameters:
    update_rate_hz: 25.0
```

**Common gotcha:** `use_sim_time: true` must be set when running against a Gazebo / Isaac sim that publishes `/clock`. Otherwise tf timestamps won't agree.

## Namespacing and remapping

**Always namespace from the start.** Even single-robot today:

```python
GroupAction([
    PushRosNamespace('robot1'),
    Node(package='my_pkg', executable='my_node', remappings=[
        ('camera/image', 'camera_front/image'),
    ]),
])
```

Topics inside the group become `/robot1/...`. Remappings are local to the action.

For multi-robot, parameterize the namespace:
```python
DeclareLaunchArgument('robot_ns', default_value='robot1')
PushRosNamespace(LaunchConfiguration('robot_ns'))
```

## DDS configuration

**Pick one RMW.** `rmw_cyclonedds_cpp` (default on Jazzy) for simpler ops; `rmw_fastrtps_cpp` for older deployments.

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

**Discovery on local network only.** Set `ROS_LOCALHOST_ONLY=1` for laptop dev. For multi-machine, use the same `ROS_DOMAIN_ID` (0–101) on all hosts.

**For high-bandwidth point clouds / images across hosts:** use the `rmw_zenoh_cpp` bridge or composable nodes (zero-copy). DDS over wire saturates Gigabit fast.

## Humble vs Jazzy — notable diffs

| Topic | Humble | Jazzy |
|---|---|---|
| Default RMW | Fast DDS | Cyclone DDS |
| Python launch type stubs | Limited | Improved (PEP 561) |
| `rclpy.spin_until_future_complete` | Available | Available; prefer `spin_until_complete` |
| Lifecycle in Python | Verbose | Cleaner `LifecycleNode` |
| MoveIt 2 | 2.5–2.7 series | 2.10+ |
| Nav2 | jazzy-compatible since 1.3 | Native |

**Migration tip:** the `ros2 doctor` and `ros2 wtf` tools surface RMW / discovery issues fast.

## ros2_control quick map

- **Hardware interface** = the C++ shim that talks to the actual motors / sensors.
- **Controller manager** = loads/unloads controllers; one process per robot.
- **Controllers** = `joint_trajectory_controller`, `velocity_controller`, etc. — they consume hardware interfaces.
- **Resource manager** = arbitrates which controller owns which joint.

If a controller "won't activate", 90% of the time it's a resource conflict (two controllers claiming the same joint) or a hardware-interface name mismatch with the URDF.

## Anti-patterns

| Don't | Do |
|---|---|
| Hardcode `/joint_states` | Namespace it: `{ns}/joint_states` |
| Re-launch with stale `/clock` | Set `use_sim_time` consistently across the whole launch |
| Mix RMW implementations | Pick one per environment |
| Spawn 50 separate processes for sensor pipeline | Use `ComposableNodeContainer` |
| Hand-merge param files | Use a single per-robot YAML and `IncludeLaunchDescription` overrides |

## When you also need

- `ros2-node-discipline` — for inside-node decisions (QoS, lifecycle, primitive choice).
- `urdf-and-kinematics-changes` — when the launch references a URDF you're about to edit.
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/ros2-patterns/README.md`:

```markdown
# Pressure tests — ros2-patterns

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.
```

Create `roboforge/tests/pressure-tests/ros2-patterns/positive.md`:

```markdown
# Positive test — ros2-patterns

**User message:**

> Write me a launch file that brings up two RealSense cameras with namespaces `cam_front` and `cam_back`, both publishing depth at 30 Hz.

**Pass criteria:**

1. The agent invokes `roboforge:ros2-patterns` before producing the launch file.
2. The launch uses `PushRosNamespace` per camera (or per-camera namespaces via launch args), not hardcoded namespaces.
3. The agent asks about RMW / `use_sim_time` if context is missing.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/ros2-patterns/negative.md`:

```markdown
# Negative test — ros2-patterns

**User message:**

> What's the latest ROS 2 LTS distro?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:ros2-patterns` (no launch / param / namespace work being done).
2. The agent answers directly (Jazzy Jalisco — LTS through 2029).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'ros2-patterns' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/ros2-patterns/ roboforge/tests/pressure-tests/ros2-patterns/
git commit -m "roboforge: add ros2-patterns knowledge skill + pressure tests"
```

---

## Task 2: `lerobot-conventions`

**Files:**
- Create: `roboforge/skills/lerobot-conventions/SKILL.md`
- Create: `roboforge/tests/pressure-tests/lerobot-conventions/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/lerobot-conventions \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/lerobot-conventions
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/lerobot-conventions/SKILL.md`:

````markdown
---
name: lerobot-conventions
description: Use when reading, writing, or transforming LeRobot datasets; defining a LeRobot policy class; or wiring LeRobotDataset into a training loop. Surfaces format conventions, hub patterns, and the parts that bite first-time users. Pair with `dataset-versioning` before any actual training run.
---

Reference for LeRobot's dataset format, policy interface, and training loop conventions. LeRobot is the de-facto bridge between hub-hosted robotics datasets and policy training; getting its conventions wrong silently produces a policy that runs without errors and predicts garbage.

## Dataset format

A LeRobotDataset is a HuggingFace `Dataset` plus a sidecar layout:

```
<dataset_root>/
├── meta/
│   ├── info.json              # codec versions, fps, robot_type
│   ├── stats.json             # mean/std/min/max per feature
│   ├── tasks.jsonl            # task descriptions (text labels)
│   └── episodes.jsonl         # episode lengths, task indices
├── data/
│   └── chunk-000/
│       ├── episode_000000.parquet
│       └── episode_000001.parquet
└── videos/
    └── chunk-000/
        └── observation.images.cam_high/
            └── episode_000000.mp4
```

Parquet rows are timesteps; columns are features (`observation.state`, `action`, `observation.images.cam_*` as encoded video frame indices, plus `episode_index`, `frame_index`, `timestamp`, `task_index`).

**Why parquet + videos:** images are huge. Storing them as MP4 with codec metadata in `info.json` keeps the dataset small AND the random-access fast.

## Loading

```python
from lerobot.common.datasets.lerobot_dataset import LeRobotDataset

dataset = LeRobotDataset(
    "lerobot/aloha_sim_transfer_cube_human",
    revision="v2.0",                     # PIN THIS
    delta_timestamps={
        "observation.images.top": [0.0],
        "observation.state": [-0.1, 0.0],   # state history window
        "action": [0.0, 0.02, 0.04, ..., 0.5],  # action chunk horizon
    },
)
```

`delta_timestamps` is the policy-specific window. ACT uses an action chunk of N future steps; diffusion policy uses both observation history and action horizon.

## Policy interface

A LeRobot policy subclasses `PreTrainedPolicy`:

```python
from lerobot.common.policies.pretrained import PreTrainedPolicy
from lerobot.configs.policies import PreTrainedConfig

class MyPolicyConfig(PreTrainedConfig):
    chunk_size: int = 16
    n_obs_steps: int = 1
    # ...

class MyPolicy(PreTrainedPolicy):
    config_class = MyPolicyConfig
    name = "my_policy"

    def __init__(self, config, dataset_stats=None):
        super().__init__(config)
        # store stats; build modules
        ...

    def forward(self, batch):
        # training step
        return {"loss": ...}

    def select_action(self, observation):
        # inference; returns action(s) of shape (B, action_dim)
        ...
```

`select_action` typically maintains an internal action queue (chunk size > 1) so the inference rate is decoupled from the control rate.

## Hub patterns

Push:
```python
dataset.push_to_hub("yourname/your_task", revision="v1.0", private=True)
```

Pull and pin:
```python
dataset = LeRobotDataset("yourname/your_task", revision="abc123def")  # commit hash
```

**Always pin a `revision`** in production training runs (this is what `dataset-versioning` enforces).

## Training entrypoint

LeRobot ships `lerobot/scripts/train.py` driven by Hydra. Override anything from CLI:

```bash
python lerobot/scripts/train.py \
    dataset_repo_id=lerobot/aloha_sim_transfer_cube_human \
    policy=act \
    policy.chunk_size=20 \
    training.num_workers=8 \
    wandb.enable=true
```

Configs live under `lerobot/configs/`. Custom policies register via the entry-point machinery; see existing `act`, `diffusion`, `vqbet` for templates.

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| Forgetting `delta_timestamps` for the action horizon | Policy trains on single-step actions; action chunking has no effect |
| Mixing image format (BGR / RGB) between collection and training | Policy works in sim, looks "off" on real |
| Skipping `dataset.stats` at deployment | Inference applies wrong normalization; predictions drift |
| Re-encoding videos with a different codec | Frame-index lookups change; row order can shift |
| Adding episodes mid-run | Stats need recompute; old checkpoints become incompatible silently |
| Leaving `cache_dir` on a small disk | Hub cache fills root partition; downloads start failing mid-training |

## When you also need

- `dataset-versioning` — before kicking off any training using this dataset.
- `policy-eval-protocol` — before claiming the trained policy works.
- `vla-policies-overview` — when picking which policy class to instantiate.
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/lerobot-conventions/README.md`:

```markdown
# Pressure tests — lerobot-conventions

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/lerobot-conventions/positive.md`:

```markdown
# Positive test — lerobot-conventions

**User message:**

> Help me write a custom LeRobot policy class that takes a 5-step observation history and predicts a 16-step action chunk.

**Pass criteria:**

1. The agent invokes `roboforge:lerobot-conventions` before writing the class.
2. The agent uses `PreTrainedPolicy`, `PreTrainedConfig`, correct `delta_timestamps`, and an action queue in `select_action`.
3. The agent does NOT invent a non-LeRobot interface.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/lerobot-conventions/negative.md`:

```markdown
# Negative test — lerobot-conventions

**User message:**

> Who maintains LeRobot?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:lerobot-conventions` (no code/dataset work).
2. The agent answers directly (HuggingFace).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'lerobot-conventions' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/lerobot-conventions/ roboforge/tests/pressure-tests/lerobot-conventions/
git commit -m "roboforge: add lerobot-conventions knowledge skill + pressure tests"
```

---

## Task 3: `vla-policies-overview`

**Files:**
- Create: `roboforge/skills/vla-policies-overview/SKILL.md`
- Create: `roboforge/tests/pressure-tests/vla-policies-overview/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/vla-policies-overview \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/vla-policies-overview
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/vla-policies-overview/SKILL.md`:

````markdown
---
name: vla-policies-overview
description: Use when picking, comparing, or migrating between VLA / imitation-learning policies (OpenVLA, Octo, Pi0, RDT2, GR00T, ACT, Diffusion Policy, VQ-BeT). Surfaces what each policy actually is, what it expects in/out, and what it's good at — so the choice is deliberate, not "the one I read about last".
---

Quick map of the major VLA / IL policy families. Use as a decision aid; for implementation conventions see `lerobot-conventions`.

## At a glance

| Policy | Type | Action chunking | Vision encoder | Backbone | Best at | Watch out for |
|---|---|---|---|---|---|---|
| **OpenVLA** | VLA (token regression) | No (single-step) | DINOv2 + SigLIP | Llama-2 7B | OOD generalization, language conditioning | Slow inference (~5 Hz on consumer GPU); single-arm 7-DoF default |
| **Octo** | Foundation transformer | Yes | T5 + ViT | Diffusion head | Multi-task pretraining (OXE) | Pretrained data quality limits ceiling |
| **Pi0 / Pi0-FAST** | Flow-matching | Yes | PaliGemma | Action experts | SOTA dexterity, bimanual | Compute-heavy; less reproducible OSS than ACT |
| **RDT2** | Diffusion transformer | Yes | DINOv2 | Diffusion | Bimanual, dual-arm | Newer; ecosystem still maturing |
| **GR00T (N1, N2)** | VLA | Yes | NVILA-derived | Action expert | Humanoid native, NVIDIA stack | Tied to Isaac / NVIDIA stack; bring-up effort |
| **ACT** | Transformer encoder-decoder | Yes (chunk_size) | ResNet18 | Small CVAE | Fast, lightweight, well-documented | Less generalization than VLA-class |
| **Diffusion Policy** | Diffusion (UNet or Transformer) | Yes (horizon) | ResNet18 | UNet / Transformer | Multi-modal action distributions | Slower than ACT; needs care with EMA |
| **VQ-BeT** | Discrete-action transformer | Yes | ResNet | Mini-GPT | Multi-modal demos | Tokenizer needs tuning per dataset |

## How they differ in practice

**Action format.**
- OpenVLA: 7-DoF token (de-tokenize → continuous). Single timestep.
- ACT / Diffusion / VQ-BeT / Pi0 / RDT2 / GR00T / Octo: continuous chunks (predict N future actions, execute first K).

**Compute at training time.**
- ACT, Diffusion Policy, VQ-BeT: train on one A100 in hours.
- Octo, RDT2: multi-GPU days for pretraining; fine-tune cheap.
- OpenVLA, Pi0, GR00T: serious compute (multi-node) for pretraining; LoRA / fine-tune feasible on single 80 GB GPU.

**Compute at inference.**
- ACT, VQ-BeT, Diffusion Policy: real-time on mid-tier GPU.
- OpenVLA: slow; needs quantization or batched inference for ≥10 Hz.
- Pi0 / RDT2 / GR00T: depends on action-expert size; usually OK with action chunking.

**Language conditioning.**
- OpenVLA, Octo, Pi0, RDT2, GR00T — yes (different mechanisms).
- ACT, Diffusion Policy, VQ-BeT — no (task-conditioning via separate goal tokens or single-task assumption).

## Picking one — quick decision tree

1. **Single task, single arm, fast iteration?** ACT or Diffusion Policy. ACT first if you want speed; Diffusion if your demos are multi-modal.
2. **Single task, want best success rate?** Diffusion Policy.
3. **Multi-task pretraining, want to fine-tune?** Octo (OSS, OXE pretraining) or Pi0 (if you have access).
4. **Bimanual / dual-arm?** RDT2 or Pi0. ACT works too but lower ceiling.
5. **Humanoid?** GR00T (NVIDIA stack) or Pi0 with humanoid action head. Roll your own at your peril.
6. **Want OOD language generalization?** OpenVLA. Accept slow inference.
7. **You are doing research and need a clean baseline?** ACT.

## Common confusion points

- **"Diffusion policy" can mean three things:** (a) Cheng Chi's original 2023 paper (UNet over actions), (b) the LeRobot `DiffusionPolicy` class (UNet variant by default), (c) any policy whose action head uses a diffusion process (Octo, RDT2). Specify.
- **"VLA" is loose.** Originally meant "vision-language-action" (RT-2, OpenVLA — single-step, language-conditioned). Now used for any VLM-derived policy regardless of action format. Pi0 / Octo / GR00T are VLA-adjacent in this loose sense.
- **OpenVLA's 7-DoF default is a constraint.** Bimanual or non-7-DoF embodiments need either re-tokenization or a different policy.
- **GR00T runs on NVIDIA stack.** Isaac Sim / Lab + Triton for serving. If you don't already use NVIDIA infra, the bring-up cost dominates.
- **Action chunking ≠ predicting the future correctly.** Chunk size is a robustness mechanism (decouple inference rate from control rate), not a guarantee of long-horizon behavior. Long-horizon needs language or hierarchical policies.

## When you also need

- `lerobot-conventions` — for the dataset/training plumbing of any of these.
- `dataset-versioning` — before training one.
- `policy-eval-protocol` — before claiming the chosen policy works.
- `humanoid-wbc` — if picking GR00T or Pi0 for a humanoid.
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/vla-policies-overview/README.md`:

```markdown
# Pressure tests — vla-policies-overview

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/vla-policies-overview/positive.md`:

```markdown
# Positive test — vla-policies-overview

**User message:**

> I'm starting a new bimanual cube-stacking project. Should I use ACT, Diffusion Policy, RDT2, or Pi0?

**Pass criteria:**

1. The agent invokes `roboforge:vla-policies-overview` before recommending.
2. The agent compares trade-offs (compute, data needs, bimanual support, OSS availability) instead of recommending one off the cuff.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/vla-policies-overview/negative.md`:

```markdown
# Negative test — vla-policies-overview

**User message:**

> When was OpenVLA published?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:vla-policies-overview` (factual lookup, no decision being made).
2. The agent answers directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'vla-policies-overview' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/vla-policies-overview/ roboforge/tests/pressure-tests/vla-policies-overview/
git commit -m "roboforge: add vla-policies-overview knowledge skill + pressure tests"
```

---

## Task 4: `isaac-lab-workflows`

**Files:**
- Create: `roboforge/skills/isaac-lab-workflows/SKILL.md`
- Create: `roboforge/tests/pressure-tests/isaac-lab-workflows/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/isaac-lab-workflows \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/isaac-lab-workflows
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/isaac-lab-workflows/SKILL.md`:

````markdown
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
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/isaac-lab-workflows/README.md`:

```markdown
# Pressure tests — isaac-lab-workflows

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/isaac-lab-workflows/positive.md`:

```markdown
# Positive test — isaac-lab-workflows

**User message:**

> I want to set up a new Isaac Lab task for a UR5 to push a block. Where do I start?

**Pass criteria:**

1. The agent invokes `roboforge:isaac-lab-workflows` before sketching the task.
2. The agent recommends Manager-based vs Direct deliberately, mentions scene/observations/rewards/terminations cfg structure, and mentions RSL-RL or SKRL choice.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/isaac-lab-workflows/negative.md`:

```markdown
# Negative test — isaac-lab-workflows

**User message:**

> Does Isaac Sim run on macOS?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:isaac-lab-workflows` (system compatibility question, not a workflow task).
2. The agent answers directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'isaac-lab-workflows' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/isaac-lab-workflows/ roboforge/tests/pressure-tests/isaac-lab-workflows/
git commit -m "roboforge: add isaac-lab-workflows knowledge skill + pressure tests"
```

---

## Task 5: `manipulation-primitives`

**Files:**
- Create: `roboforge/skills/manipulation-primitives/SKILL.md`
- Create: `roboforge/tests/pressure-tests/manipulation-primitives/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/manipulation-primitives \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/manipulation-primitives
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/manipulation-primitives/SKILL.md`:

````markdown
---
name: manipulation-primitives
description: Use when writing manipulation code — grasping, pick-and-place, approach/retreat motions, gripper control, frame conversions. Surfaces the conventions and primitives that fail silently when not stated explicitly. Do NOT use for ROS 2 plumbing — that's `ros2-patterns` / `ros2-node-discipline`.
---

Reference for the building blocks of pick-and-place and contact-rich manipulation. Most "the gripper went to the wrong spot" bugs are frame errors or pre/post-grasp pose errors — both addressed here.

## Pose conventions

Always state the frame. Always.

| Notation | Meaning |
|---|---|
| `T_world_ee` | Transform from `world` frame to `end_effector` frame. Pose of EE expressed in world. |
| `T_base_ee` | Pose of EE in robot base frame. The thing IK consumes. |
| `T_ee_tool` | Tool offset (e.g. parallel-jaw center, suction-cup tip). Composes with `T_base_ee`. |
| `T_world_obj` | Object pose in world. From perception. |
| `T_obj_grasp` | Grasp pose in object frame. From a grasp predictor. |
| `T_world_grasp` | Goal: `T_world_obj @ T_obj_grasp`. Convert to `T_base_ee` before IK. |

**Right-hand rule.** Gripper +Z usually points along approach direction (out of the palm). +X along jaw-opening axis. Confirm per gripper.

**Quaternions vs Euler.** Always quaternions in code. Euler only for human display. Half the manipulation bugs are Euler-singularity bugs at gimbal lock.

## Grasp pipelines

Three common pipelines:

1. **Predict 6-DoF grasp directly** (GraspNet, AnyGrasp, Contact-GraspNet). Input: point cloud. Output: list of `(T_world_grasp, score, gripper_width)`. Filter by reachability + collision. Pick best.
2. **Detect object → look up grasp annotation** (when objects are known: cups, boxes, tools). Object-frame grasps are pre-defined.
3. **Learned policy outputs gripper pose directly** (ACT, Diffusion Policy, OpenVLA). No explicit grasp prediction; the policy learns it. This is what VLAs do.

For pipelines 1 and 2, the agent code orchestrates: detect → predict → filter → execute. For pipeline 3, the policy is the orchestrator; you only wire I/O.

## Approach / retreat motion

A grasp is **three** poses, not one:

1. **Pre-grasp pose** — `T_world_grasp` offset along -approach axis by ~10 cm. The arm moves freely here.
2. **Grasp pose** — `T_world_grasp`. Closed-loop or slow-Cartesian motion from pre-grasp.
3. **Post-grasp / lift** — `T_world_grasp` offset along +world Z by ~5–10 cm. Lift before transit.

Skipping the pre-grasp = collision risk. Skipping the lift = drag the object across the table.

For placement, mirror: pre-place above target, place, retract.

## Gripper choice

| Gripper type | Best at | Watch out for |
|---|---|---|
| Parallel-jaw (e.g. Robotiq 2F-85, Franka Hand) | Rigid objects, known geometry | Width mismatch; soft objects squish |
| Suction (e.g. Robotiq EPick) | Flat-top objects (boxes, planar parts) | Porous / curved surfaces fail silently; needs vacuum check |
| Multi-finger (e.g. Allegro, Shadow) | Dexterous, in-hand manipulation | Calibration / wear; teleop costly |
| Soft / under-actuated (e.g. SoftGripper) | Cluttered scenes, unknown shapes | Position feedback poor; force control limited |

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| Predicting grasps in camera frame, executing in world frame | Robot grasps a foot to the side; perception/exec frame mismatch |
| Using `T_base_ee` from FK as `T_world_ee` (no base transform) | Off by the base-in-world offset; common in mobile manipulators |
| Forgetting tool offset `T_ee_tool` | Suction cup tip is 5–8 cm past wrist flange; grasps come up short |
| Closing gripper width = object width exactly | No grasp force; thermal expansion + slop = drop |
| Hand-coding pre-grasp = grasp - 10 cm in world Z | Wrong for side grasps. Offset along the *approach* axis, not world Z. |
| Re-planning every 1 ms during slow grasp motion | Planner thrashes; jitter |

## Decision: motion planning vs servoing

| Use motion planning (MoveIt 2, OMPL) | Use servoing (Cartesian, MoveIt Servo, custom controller) |
|---|---|
| Long-distance free-space motion | Short-distance reactive motion |
| Known goal pose, pre-grasp / pre-place | Visual-servoing onto a moving target |
| Need collision avoidance with environment | Need closed-loop response to live perception |
| Acceptable to wait 100–500 ms for plan | Need ≥10 Hz update |

Most "policy" code uses servoing (the policy IS the controller). Most "scripted" pick-and-place uses motion planning for transit + servoing for fine alignment.

## When you also need

- `urdf-and-kinematics-changes` — if you're modifying the gripper / tool URDF.
- `ros2-node-discipline` — when wiring the manipulation node.
- `safe-hardware-deployment` — before any real-arm motion.
- `perception-stack` — when grasp inputs come from cameras.
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/manipulation-primitives/README.md`:

```markdown
# Pressure tests — manipulation-primitives

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/manipulation-primitives/positive.md`:

```markdown
# Positive test — manipulation-primitives

**User message:**

> Write me a Python function that takes a 6-DoF grasp pose from AnyGrasp and executes it on a Franka with a Robotiq gripper. Use MoveIt 2.

**Pass criteria:**

1. The agent invokes `roboforge:manipulation-primitives` before producing code.
2. The agent uses pre-grasp / grasp / lift sequence (not single-pose execution).
3. The agent stages frames explicitly (camera → world → base → EE) and includes tool offset.
4. The agent does NOT invent a grasp executor that goes straight to the grasp pose without pre/post.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/manipulation-primitives/negative.md`:

```markdown
# Negative test — manipulation-primitives

**User message:**

> What's the payload of a Franka Research 3?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:manipulation-primitives` (specs lookup).
2. The agent answers directly (3 kg).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'manipulation-primitives' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/manipulation-primitives/ roboforge/tests/pressure-tests/manipulation-primitives/
git commit -m "roboforge: add manipulation-primitives knowledge skill + pressure tests"
```

---

## Task 6: `humanoid-wbc`

**Files:**
- Create: `roboforge/skills/humanoid-wbc/SKILL.md`
- Create: `roboforge/tests/pressure-tests/humanoid-wbc/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/humanoid-wbc \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/humanoid-wbc
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/humanoid-wbc/SKILL.md`:

````markdown
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
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/humanoid-wbc/README.md`:

```markdown
# Pressure tests — humanoid-wbc

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/humanoid-wbc/positive.md`:

```markdown
# Positive test — humanoid-wbc

**User message:**

> I want to retarget AMASS MoCap onto Unitree H1-2 and train a tracking policy. What's the right pipeline?

**Pass criteria:**

1. The agent invokes `roboforge:humanoid-wbc` before recommending.
2. The agent mentions retargeting (PHC / GMR / Mink), the H1-2 skeleton mismatch with SMPL, and foot-skating correction.
3. The agent does NOT propose to "just use joint angles directly".

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/humanoid-wbc/negative.md`:

```markdown
# Negative test — humanoid-wbc

**User message:**

> How tall is Unitree H1?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:humanoid-wbc` (specs question).
2. The agent answers directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'humanoid-wbc' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/humanoid-wbc/ roboforge/tests/pressure-tests/humanoid-wbc/
git commit -m "roboforge: add humanoid-wbc knowledge skill + pressure tests"
```

---

## Task 7: `perception-stack`

**Files:**
- Create: `roboforge/skills/perception-stack/SKILL.md`
- Create: `roboforge/tests/pressure-tests/perception-stack/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/perception-stack \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/perception-stack
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/perception-stack/SKILL.md`:

````markdown
---
name: perception-stack
description: Use when configuring cameras (RealSense, ZED, generic CSI), calibrating intrinsics or extrinsics, syncing time across sensors, or building a GPU-accelerated perception pipeline. Surfaces the patterns and silent failure modes.
---

Reference for the camera / depth / time-sync layer underneath robotics work. Most "the policy doesn't see what I see" failures are perception-stack failures: wrong color order, wrong depth units, unsynced timestamps.

## Camera calibration

Two distinct calibrations:

1. **Intrinsics** (focal length, principal point, distortion). Per camera. Use `camera_calibration` (ROS) or OpenCV directly with a checkerboard/charuco target. Save as `camera_info.yaml` and load via the camera driver.
2. **Extrinsics** (camera → robot frame). Hand-eye calibration. Use `easy_handeye2` or `MoveIt 2 Calibration`. The four common variants:
   - **Eye-on-hand** (camera mounted on EE) → solves `T_ee_camera`.
   - **Eye-to-hand** (camera fixed, watching the arm) → solves `T_world_camera`.
   Each requires moving the arm through ≥10 distinct poses with a target visible.

**Calibration drifts** — re-do whenever:
- The camera or its mount moves (even small bumps).
- The lens is replaced or refocused.
- The robot base moves (eye-to-hand only).
- A new tool is mounted (eye-on-hand: `T_ee_camera` doesn't change but downstream `T_camera_tool` does).

## RealSense (D4xx series)

Launch via `realsense2_camera`. Critical params:

```yaml
align_depth.enable: true        # depth registered to color frame; you almost always want this
enable_sync: true               # hardware-sync depth and color streams
depth_module.profile: "640x480x30"
rgb_camera.profile: "640x480x30"
pointcloud.enable: false        # off unless you actually consume it (CPU/bandwidth heavy)
spatial_filter.enable: true     # smoother depth at cost of edge detail
temporal_filter.enable: true    # ditto
hole_filling_filter.enable: false  # adds artifacts; usually skip
```

**Watch out for:**
- **Depth in mm.** RealSense returns uint16 mm. Convert to meters before consumption.
- **First-frame zero depth.** The IR projector takes ~1 sec to settle. Skip the first 30 frames.
- **USB bandwidth.** D435 + D435 on one USB controller saturates. Spread across controllers.
- **Reflective / transparent objects.** Depth is invalid; the policy must tolerate NaN/zero or use color-only inputs.

## ZED (Stereolabs)

Launch via `zed-ros2-wrapper`. Critical params:

```yaml
general.camera_model: "zed2i"     # or zedx, zed-mini
general.resolution: "HD720"
depth.depth_mode: "NEURAL"        # neural depth = better; needs Jetson or RTX GPU
depth.min_depth: 0.3              # below this, no depth
sensors.publish_imu_tf: false     # unless you need IMU TF
pos_tracking.area_memory: false   # turn off unless using SLAM
```

**Watch out for:**
- **NEURAL depth requires CUDA.** On CPU-only or weak GPU, fall back to ULTRA or QUALITY.
- **IMU is in camera frame, not robot frame.** Apply `T_robot_camera` to use it for state estimation.
- **Tracking and depth share GPU.** If you want max depth FPS, disable positional tracking.

## Time synchronization

Three problems, three answers:

1. **Sync within a single multi-sensor camera (e.g. RGB + depth on RealSense).** Hardware-sync via the camera (`enable_sync: true`). Trust the timestamps.
2. **Sync across separate sensors (camera + IMU + joint encoders).** Use `message_filters.ApproximateTimeSynchronizer` in ROS 2:

```python
from message_filters import Subscriber, ApproximateTimeSynchronizer
img_sub = Subscriber(self, Image, "/camera/color/image_raw")
js_sub = Subscriber(self, JointState, "/joint_states")
sync = ApproximateTimeSynchronizer([img_sub, js_sub], queue_size=10, slop=0.05)
sync.registerCallback(self.synced_cb)
```

3. **Sync across machines.** PTP (chrony with `ptp4l`) on a wired network. NTP isn't tight enough for vision-control loops.

**Use `use_sim_time: true` consistently across the launch graph** when running with `/clock`. Mixed sim/wall time is the most common "TF says my transform is in the future" cause.

## GPU-accelerated pipelines

For high-rate perception:

- **CUDA stream from camera straight to GPU.** Avoid bouncing image buffers through CPU. RealSense and ZED both have CUDA-direct paths.
- **JPEG decode on GPU.** `nvjpeg` for offline; `nvJPEG2000` for compressed sensor streams.
- **Composable nodes (ROS 2)** for zero-copy intra-process. Combine debayer, rectify, depth-align in one process.
- **TensorRT for inference.** Convert ONNX → TRT engine; expect 2–5× over plain PyTorch on Jetson / RTX.

## Common pitfalls

| Pitfall | What actually happens |
|---|---|
| RGB vs BGR mix between training and deploy | Policy outputs look "right" in sim, totally wrong colors at inference |
| Depth in mm vs meters | Off by 1000×; policy commands collisions or hovers far above target |
| Depth not aligned to color | Pixel-level lookup gives the wrong depth; grasps come up empty |
| Trusting `header.stamp` from a sensor that doesn't sync to system time | TF interpolation fails; "transform from past" errors |
| Using a single calibration across multiple physical cameras of "same model" | Per-unit lens variance is real; calibrate each one |
| Pointcloud-from-depth on every frame for visualization only | CPU pegged; control loop misses deadlines |

## When you also need

- `ros2-patterns` — for launch / namespacing of perception nodes.
- `ros2-node-discipline` — for QoS choices (sensor topics → `BEST_EFFORT`).
- `manipulation-primitives` — when perception output drives a grasp.
````

- [ ] **Step 3: Write pressure-test files**

Create `roboforge/tests/pressure-tests/perception-stack/README.md`:

```markdown
# Pressure tests — perception-stack

## How to run

1. Open a clean Claude Code session.
2. Confirm `roboforge` is installed.
3. For each test, send the prompt in a fresh `/clear`-ed session.
```

Create `roboforge/tests/pressure-tests/perception-stack/positive.md`:

```markdown
# Positive test — perception-stack

**User message:**

> Set up time-synchronized RGB + depth + joint_states subscribers for my Franka with a wrist-mounted RealSense D435.

**Pass criteria:**

1. The agent invokes `roboforge:perception-stack` before producing the code.
2. The agent uses `ApproximateTimeSynchronizer`, mentions `align_depth.enable`, mentions hardware-sync, and converts depth to meters.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

Create `roboforge/tests/pressure-tests/perception-stack/negative.md`:

```markdown
# Negative test — perception-stack

**User message:**

> Is the RealSense D405 still in production?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:perception-stack` (product availability question).
2. The agent answers directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 4: Smoke-test session-start**

```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'perception-stack' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 5: Commit**

```bash
git add roboforge/skills/perception-stack/ roboforge/tests/pressure-tests/perception-stack/
git commit -m "roboforge: add perception-stack knowledge skill + pressure tests"
```

---

## Task 8: Manual session-test sweep

The 14 transcripts (7 positive + 7 negative) are run by the human partner in fresh CC sessions, same pattern as Plan 2.

- [ ] **Step 1: Reinstall the plugin so new skills are loaded**

```text
/plugin marketplace remove roboforge-dev
/plugin marketplace add /home/phoenix/vla/superpowers/roboforge
/plugin install roboforge@roboforge-dev
/clear
```

- [ ] **Step 2: Run all 14 pressure tests**

Skills, in order:
1. `ros2-patterns`
2. `lerobot-conventions`
3. `vla-policies-overview`
4. `isaac-lab-workflows`
5. `manipulation-primitives`
6. `humanoid-wbc`
7. `perception-stack`

For each: `/clear`, send positive prompt, paste transcript + set Result. `/clear`, send negative prompt, paste transcript + set Result.

- [ ] **Step 3: Triage failures**

A FAIL on a positive test = description not triggering. Tighten or expand the "Use when..." clause.
A FAIL on a negative test = description over-triggering. Constrain it.

- [ ] **Step 4: Commit transcripts**

```bash
git add roboforge/tests/pressure-tests/
git commit -m "roboforge: record pressure-test results for knowledge skills"
```

---

## Self-Review

**1. Spec coverage** — Plan 3 implements the knowledge row of the spec's skill catalog (7 skills). Each catalog entry has a dedicated task. Methodology (Plan 2) and scaffolding (Plan 4) are correctly out of scope.

**2. Placeholder scan** — every step has either complete file content, an exact command, or an explicit manual instruction with pass criteria. The transcript-paste sections in test files use the explicit `(Paste actual session transcript here.)` template marker, not a plan-level placeholder.

**3. Type / name consistency** — All 7 skill names match the spec catalog and `using-roboforge`'s forward declaration:
- `ros2-patterns`
- `lerobot-conventions`
- `vla-policies-overview`
- `isaac-lab-workflows`
- `manipulation-primitives`
- `humanoid-wbc`
- `perception-stack`

Cross-references between skills (manipulation-primitives → perception-stack, urdf-and-kinematics-changes; humanoid-wbc → isaac-lab-workflows, manipulation-primitives, vla-policies-overview; isaac-lab-workflows → sim2real-checklist; perception-stack → ros2-patterns, manipulation-primitives) all use the correct spelling.

---

## Out of scope (explicitly)

- **Scaffold skills** (`scaffold-ros2-package`, `scaffold-lerobot-policy`, `scaffold-isaac-task`, `scaffold-training-entrypoint`, `scaffold-teleop-pipeline`) — Plan 4.
- **Multi-harness validation beyond Claude Code** — Plan 5.
- **CI for transcripts** — manual / human-judged for v1.
