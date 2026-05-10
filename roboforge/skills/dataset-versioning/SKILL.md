---
name: dataset-versioning
description: Use when about to train, fine-tune, retrain, or append episodes to a dataset used for any learned policy or model. Pins what data the run uses, validates episode integrity, and prevents the "we trained on a different dataset and didn't notice" failure. Required before any `train.py` invocation.
---

Most "the policy regressed" incidents are dataset incidents in disguise: an episode was added, a normalization stat shifted, a split leaked, or "the dataset" silently means two different things to two different runs. This checklist makes the dataset state explicit and reproducible before training consumes it.

## When this skill applies

Invoke this skill before:
- Running any training, fine-tuning, or distillation command.
- Appending new episodes to an existing dataset.
- Switching dataset revisions (HuggingFace Hub, local snapshot, S3 path).
- Recomputing normalization statistics.
- Generating a new train/val/test split.

This skill does NOT apply to:
- Reading or visualizing data (no model state changes).
- Pure sim policy rollouts that don't write to a dataset.

## Required checklist

1. **Dataset version is pinned.** A specific commit hash, hub revision, or local snapshot identifier is in the run config and logged with the run. Bare `lerobot/aloha_sim_transfer_cube_human` is NOT pinned — `lerobot/aloha_sim_transfer_cube_human@<revision>` is.
2. **Episode count + total duration logged.** Print at run start. If the count or duration silently changes between runs you must notice.
3. **Episode validity check ran.** Walk the dataset and confirm: no truncated episodes, no NaN actions, no all-zero action sequences, action shapes match expected, observation modalities present. A 3-line sanity script is fine.
4. **Action space defined.** Document what each action dimension means (joint vs end-effector, absolute vs delta, gripper as binary or continuous, units). Mismatched action spaces between collection and training silently produce a policy that does the wrong thing.
5. **Train/val/test split is deterministic.** Seed is pinned. Same seed → same split. Splits are episode-level, not step-level (no leakage of within-episode trajectories across splits).
6. **Normalization stats recomputed from train split only.** Never include val/test in normalization. Recompute when the train split changes.
7. **Stats stored alongside checkpoints.** Save mean/std/quantile JSON next to the checkpoint, so deployment uses the exact stats training used. A policy that uses different stats at inference is a different policy.
8. **Run config logs the answers above.** W&B / tensorboard / a sidecar JSON — somewhere durable, queryable later.

## Red flags

| Thought | Reality |
|---|---|
| "I just added a few episodes" | A few episodes can shift normalization stats and silently degrade existing checkpoints. |
| "We always use the latest" | "Latest" is not a version. Pin a hash. |
| "Validation is identical to last run" | If the seed wasn't pinned, it isn't. |
| "I'll recompute stats later" | Forgotten until deployment fails. Recompute now. |
| "The dataset on disk hasn't changed" | Has it? `git status` it, `ls -lR` it, hash it. Confirm. |

## After training

- Log dataset hash next to the resulting checkpoint name. This is what `policy-eval-protocol` will need.
- If you appended episodes mid-run: that's a different dataset. Re-run from scratch or document the boundary explicitly.
