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
