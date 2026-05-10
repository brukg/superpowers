---
name: policy-eval-protocol
description: Use when about to claim a learned policy "works", "is ready", "passes", or before deploying it on real hardware. Defines the evaluation contract — episode set, metrics, seeds, statistical power, ID/OOD coverage — so success claims are reproducible and not cherry-picked.
---

Most VLA / IL "it works!" claims fall apart on closer inspection because the evaluation wasn't pre-registered. This skill enforces a pre-registered eval before the run, so success or failure is unambiguous.

## When this skill applies

Invoke this skill when:
- The user says a policy "works" / "passes" / "is ready" / "is good".
- About to run a final evaluation pass.
- About to deploy a checkpoint to real hardware (combine with `safe-hardware-deployment`).
- About to publish results, push a checkpoint to the hub, or close out a training run.

This skill does NOT apply to:
- Quick sanity rollouts during training (call those out as such, not as "works").
- Inference-speed benchmarks unrelated to task success.

## Required checklist

1. **Eval episode set is defined and frozen.** A list of (env config, initial state, goal) tuples that does not change between runs. Save the list (or a hash of it) with results.
2. **Success metric is unambiguous.** Per-task definition. Examples: "object in target zone within 3 cm for 0.5 s", "all 4 cubes stacked", "trajectory completed within 30 s without collision". Edge cases stated.
3. **Random seeds enumerated and pinned.** Per-episode seed list, not a single global seed. Same seed list → same eval.
4. **Coverage includes ID and OOD splits.** ID = same conditions as training. OOD = at least one varied factor (object pose, lighting, distractor count, gripper jaw width, robot pose). Report both separately.
5. **Statistical power is sufficient.** ≥10 evals per condition cell as a minimum, ≥30 for confident comparisons. Report success rate AND number of trials.
6. **Sim eval gate before real eval.** If the policy was trained with a sim-trained component, sim eval must clear an agreed bar before any real-eval. Real-eval is the most expensive evidence; don't burn it on a checkpoint that fails sim.
7. **Full provenance recorded.** Checkpoint hash, dataset version (from `dataset-versioning`), training commit, eval set hash, seed list, metrics, environment versions. One JSON per eval run.
8. **No comparing apples to oranges.** If you want to compare two checkpoints, run them on the same eval set with the same seed list. Reuse, don't re-roll.

## Red flags

| Thought | Reality |
|---|---|
| "I ran it a few times and it worked" | "A few" is not a sample size. Run the full eval. |
| "It works on this object" | One object isn't a distribution. Define the set. |
| "Looks good in sim, ship it" | Sim ≠ real. Real-eval before claiming hardware success. |
| "Different seed but same task" | Different seed = different evaluation. Don't compare. |
| "The success rate is 80%, that's great" | Out of 10 trials, that's 6.5%–94.5% with 95% confidence. Run more. |
| "Just one OOD test" | One OOD condition isn't OOD coverage. |

## When you can claim it works

A policy "works" if:
- Sim eval ≥ X% on the frozen ID set (where X was set before training).
- Real eval ≥ Y% on the frozen ID set (where Y was set before training).
- OOD eval explicitly reported (no implicit claim of robustness).
- Provenance JSON exists and is committed.

Anything less is a claim about the run, not the policy.
