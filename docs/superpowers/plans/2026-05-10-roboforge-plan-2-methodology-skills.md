# Roboforge Plan 2 of 5 — Methodology Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the six roboforge methodology skills that auto-trigger before risky robotics work — `safe-hardware-deployment`, `dataset-versioning`, `policy-eval-protocol`, `urdf-and-kinematics-changes`, `sim2real-checklist`, `ros2-node-discipline`. Each enforces a concrete checklist the agent must walk before proceeding.

**Architecture:** Each skill is a single `SKILL.md` under `roboforge/skills/<skill>/`, with a Superpowers-style trigger description in frontmatter and a checklist body. Each skill ships with a positive + negative pressure-test scaffold under `roboforge/tests/pressure-tests/<skill>/`. After all six skills are written, a single batch session-test run records PASS/FAIL across all twelve transcripts.

**Tech Stack:** Markdown only. No runtime code. Validation is the SessionStart hook smoke test plus manual session transcripts in fresh Claude Code sessions.

**Reference spec:** `docs/superpowers/specs/2026-05-10-roboforge-plugin-design.md`

**Depends on:** Plan 1 (foundation must already be installed and passing).

---

## File Structure

| Path | Responsibility |
|---|---|
| `roboforge/skills/safe-hardware-deployment/SKILL.md` | Pre-flight checklist before any code path that actuates physical motion |
| `roboforge/skills/dataset-versioning/SKILL.md` | Pre-flight checklist before training, retraining, or appending episodes |
| `roboforge/skills/policy-eval-protocol/SKILL.md` | Pre-flight checklist before claiming a policy works |
| `roboforge/skills/urdf-and-kinematics-changes/SKILL.md` | Pre-flight checklist before editing URDF / Xacro / SRDF |
| `roboforge/skills/sim2real-checklist/SKILL.md` | Pre-flight checklist before deploying a sim-trained policy on real hardware |
| `roboforge/skills/ros2-node-discipline/SKILL.md` | Pre-flight checklist when creating or modifying a ROS 2 node |
| `roboforge/tests/pressure-tests/<skill>/README.md` (×6) | How to run the skill's pressure test |
| `roboforge/tests/pressure-tests/<skill>/positive.md` (×6) | Transcript template — agent SHOULD invoke the skill |
| `roboforge/tests/pressure-tests/<skill>/negative.md` (×6) | Transcript template — agent should NOT invoke the skill |

No existing files modified. (`using-roboforge` already forward-declared all six skills in Plan 1, so no edit needed.)

---

## Task 1: `safe-hardware-deployment`

**Files:**
- Create: `roboforge/skills/safe-hardware-deployment/SKILL.md`
- Create: `roboforge/tests/pressure-tests/safe-hardware-deployment/README.md`
- Create: `roboforge/tests/pressure-tests/safe-hardware-deployment/positive.md`
- Create: `roboforge/tests/pressure-tests/safe-hardware-deployment/negative.md`

- [ ] **Step 1: Create skill directory and test directory**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/safe-hardware-deployment \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/safe-hardware-deployment
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/safe-hardware-deployment/SKILL.md` with this exact content:

````markdown
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
````

- [ ] **Step 3: Write the pressure-test README**

Create `roboforge/tests/pressure-tests/safe-hardware-deployment/README.md`:

```markdown
# Pressure tests — safe-hardware-deployment

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test below, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:safe-hardware-deployment` and walks the checklist before any motion code.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write the positive transcript template**

Create `roboforge/tests/pressure-tests/safe-hardware-deployment/positive.md`:

```markdown
# Positive test — safe-hardware-deployment

**User message:**

> I have a Franka arm. Let's run my trained ACT policy on it now to grab the cup.

**Pass criteria:**

1. The agent invokes `roboforge:safe-hardware-deployment` before writing or running any motion code.
2. The agent walks the checklist with the user (e-stop, limits, sim dry-run, logging, second person, slow first run, rollback plan).
3. The agent does NOT skip straight to a `ros2 run` / Python execution command.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write the negative transcript template**

Create `roboforge/tests/pressure-tests/safe-hardware-deployment/negative.md`:

```markdown
# Negative test — safe-hardware-deployment

**User message:**

> Show me the inference code in my ACT policy.

**Pass criteria:**

1. The agent does NOT invoke `roboforge:safe-hardware-deployment` (no motion is being commanded; this is a code-reading task).
2. The agent answers the question directly, possibly invoking other appropriate skills (e.g. lerobot-conventions later in Plan 3).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test that session-start still loads cleanly**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'safe-hardware-deployment' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/safe-hardware-deployment/ \
        roboforge/tests/pressure-tests/safe-hardware-deployment/
git commit -m "roboforge: add safe-hardware-deployment skill + pressure tests"
```

---

## Task 2: `dataset-versioning`

**Files:**
- Create: `roboforge/skills/dataset-versioning/SKILL.md`
- Create: `roboforge/tests/pressure-tests/dataset-versioning/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/dataset-versioning \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/dataset-versioning
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/dataset-versioning/SKILL.md`:

````markdown
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
````

- [ ] **Step 3: Write pressure-test README**

Create `roboforge/tests/pressure-tests/dataset-versioning/README.md`:

```markdown
# Pressure tests — dataset-versioning

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:dataset-versioning` before any training command.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write positive transcript**

Create `roboforge/tests/pressure-tests/dataset-versioning/positive.md`:

```markdown
# Positive test — dataset-versioning

**User message:**

> Let's kick off training for ACT on the SO-100 cube-pickup dataset. Use 50 epochs.

**Pass criteria:**

1. The agent invokes `roboforge:dataset-versioning` before writing or running any training command.
2. The agent walks the checklist (version pin, episode validity, action space, split, normalization, stats storage).
3. The agent does NOT immediately produce a `python train.py` invocation.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write negative transcript**

Create `roboforge/tests/pressure-tests/dataset-versioning/negative.md`:

```markdown
# Negative test — dataset-versioning

**User message:**

> Plot the joint trajectories of episode 5 in my SO-100 dataset.

**Pass criteria:**

1. The agent does NOT invoke `roboforge:dataset-versioning` (visualization does not change model state).
2. The agent answers the visualization question directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'dataset-versioning' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/dataset-versioning/ \
        roboforge/tests/pressure-tests/dataset-versioning/
git commit -m "roboforge: add dataset-versioning skill + pressure tests"
```

---

## Task 3: `policy-eval-protocol`

**Files:**
- Create: `roboforge/skills/policy-eval-protocol/SKILL.md`
- Create: `roboforge/tests/pressure-tests/policy-eval-protocol/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/policy-eval-protocol \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/policy-eval-protocol
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/policy-eval-protocol/SKILL.md`:

````markdown
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
````

- [ ] **Step 3: Write pressure-test README**

Create `roboforge/tests/pressure-tests/policy-eval-protocol/README.md`:

```markdown
# Pressure tests — policy-eval-protocol

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:policy-eval-protocol` before producing the eval / claim.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write positive transcript**

Create `roboforge/tests/pressure-tests/policy-eval-protocol/positive.md`:

```markdown
# Positive test — policy-eval-protocol

**User message:**

> The diffusion policy I trained yesterday looks good. Let's confirm it works and then push it to the hub.

**Pass criteria:**

1. The agent invokes `roboforge:policy-eval-protocol` before agreeing the policy "works" or pushing.
2. The agent walks the checklist (frozen eval set, metrics, seeds, ID+OOD, sample size, sim-gate, provenance).
3. The agent does NOT immediately push.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write negative transcript**

Create `roboforge/tests/pressure-tests/policy-eval-protocol/negative.md`:

```markdown
# Negative test — policy-eval-protocol

**User message:**

> Show me the architecture diagram for diffusion policy.

**Pass criteria:**

1. The agent does NOT invoke `roboforge:policy-eval-protocol` (no claim of correctness, no eval, no deployment).
2. The agent answers the architecture question directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'policy-eval-protocol' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/policy-eval-protocol/ \
        roboforge/tests/pressure-tests/policy-eval-protocol/
git commit -m "roboforge: add policy-eval-protocol skill + pressure tests"
```

---

## Task 4: `urdf-and-kinematics-changes`

**Files:**
- Create: `roboforge/skills/urdf-and-kinematics-changes/SKILL.md`
- Create: `roboforge/tests/pressure-tests/urdf-and-kinematics-changes/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/urdf-and-kinematics-changes \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/urdf-and-kinematics-changes
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/urdf-and-kinematics-changes/SKILL.md`:

````markdown
---
name: urdf-and-kinematics-changes
description: Use when about to edit a URDF, Xacro, SRDF, or any robot description that downstream code (controllers, planners, MoveIt, sim) consumes. A bad URDF silently breaks IK, planning, collision checking, and dynamics. This checklist catches it before downstream burns hours.
---

URDF/Xacro errors rarely raise at edit time. They surface as "MoveIt can't find a plan", "the arm twitches at the same joint", or "sim and real disagree by 5 cm" — debugged downstream, attributed to the wrong cause. This skill enforces validation at the edit site.

## When this skill applies

Invoke this skill before:
- Adding, removing, or renaming a link or joint.
- Changing joint type, axis, origin, or limits.
- Updating mass, inertia, or center-of-mass.
- Changing visual or collision meshes / geometries.
- Editing a Xacro macro that expands into URDF.
- Editing the SRDF (planning groups, virtual joints, disabled collisions).
- Updating the URDF version pinned in a controller / planner config.

This skill does NOT apply to:
- Changing only `<material>` or visualization-only fields with no mesh/geometry change.
- Pure documentation edits.

## Required checklist

1. **Build the URDF.** Run the parser end-to-end:
   - `ros2 run xacro xacro <file>.urdf.xacro` (Xacro expansion).
   - `check_urdf <file>.urdf` (URDF parse).
   No errors before any other step.
2. **Visualize.** Open in RViz with `joint_state_publisher_gui`. Sweep every joint through its range. Confirm no link flips, no detached pieces, no obvious geometry errors.
3. **FK sanity at canonical configurations.** At the zero configuration and at one named pose (home, ready, etc.), compute end-effector pose. Confirm it matches expectation (millimeter-level for arms, centimeter-level for mobile bases).
4. **Joint limits respect actuator datasheet.** Position limits, velocity limits, effort limits — each ≤ datasheet value with margin. A URDF can claim 360° on a joint that physically can't.
5. **Mass and inertia plausible.** No zero masses on moving links. No 1e-9 inertias (use a non-degenerate inertia for any moving body — even a coarse box approximation). Total mass within ~10% of the real robot.
6. **Collision meshes simpler than visual meshes, AND present.** Missing collision = false-positive plan success. Overly detailed collision = planner timeouts. Use convex hulls or primitives.
7. **MoveIt SRDF synced (if applicable).** If you renamed a link/joint, the SRDF references must update. Re-run the MoveIt setup assistant or hand-edit consistently. Disabled-collision matrix re-checked.
8. **Downstream configs re-checked.** Controller YAMLs, ros2_control resources, sim launch files — anything that names links/joints. A rename in URDF without a rename downstream produces silent garbage.

## Red flags

| Thought | Reality |
|---|---|
| "It still loads" | URDF parsers tolerate a lot. Loading ≠ correct. |
| "MoveIt will figure it out" | MoveIt fails politely on a bad URDF. Hours later you discover why. |
| "Sim looks fine" | Sim uses sim's own tolerance for bad inertias. Real does not. |
| "Inertia doesn't matter for kinematic-only planning" | Until someone enables dynamics. |
| "I'll regenerate the SRDF later" | Disabled-collision matrix drift is the most-debugged 'why is the planner so slow' cause. |

## After the edit

- Commit the URDF and any downstream YAML in the same commit. They are coupled.
- Re-run any unit tests that exercise IK / FK on this robot.
- If the change affects a controller in production: bump a version tag in the controller config so it's obvious which URDF generation a saved trajectory was recorded against.
````

- [ ] **Step 3: Write pressure-test README**

Create `roboforge/tests/pressure-tests/urdf-and-kinematics-changes/README.md`:

```markdown
# Pressure tests — urdf-and-kinematics-changes

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:urdf-and-kinematics-changes` before editing the URDF.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write positive transcript**

Create `roboforge/tests/pressure-tests/urdf-and-kinematics-changes/positive.md`:

```markdown
# Positive test — urdf-and-kinematics-changes

**User message:**

> Add a new tool link with a 50mm offset to the end-effector in my Franka URDF.

**Pass criteria:**

1. The agent invokes `roboforge:urdf-and-kinematics-changes` before editing the URDF.
2. The agent walks the checklist (parse, visualize, FK sanity, limits, inertia, collision mesh, SRDF sync, downstream configs).
3. The agent does NOT silently edit and stop.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write negative transcript**

Create `roboforge/tests/pressure-tests/urdf-and-kinematics-changes/negative.md`:

```markdown
# Negative test — urdf-and-kinematics-changes

**User message:**

> What's the difference between URDF and SDF?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:urdf-and-kinematics-changes` (purely conceptual question).
2. The agent answers directly.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'urdf-and-kinematics-changes' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/urdf-and-kinematics-changes/ \
        roboforge/tests/pressure-tests/urdf-and-kinematics-changes/
git commit -m "roboforge: add urdf-and-kinematics-changes skill + pressure tests"
```

---

## Task 5: `sim2real-checklist`

**Files:**
- Create: `roboforge/skills/sim2real-checklist/SKILL.md`
- Create: `roboforge/tests/pressure-tests/sim2real-checklist/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/sim2real-checklist \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/sim2real-checklist
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/sim2real-checklist/SKILL.md`:

````markdown
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
````

- [ ] **Step 3: Write pressure-test README**

Create `roboforge/tests/pressure-tests/sim2real-checklist/README.md`:

```markdown
# Pressure tests — sim2real-checklist

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:sim2real-checklist` (and `safe-hardware-deployment`) before deploying.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write positive transcript**

Create `roboforge/tests/pressure-tests/sim2real-checklist/positive.md`:

```markdown
# Positive test — sim2real-checklist

**User message:**

> My RL policy trained in Isaac Lab for the cube-stacking task is converged. Let's deploy it on the real Franka.

**Pass criteria:**

1. The agent invokes `roboforge:sim2real-checklist` before any deployment code.
2. The agent walks domain randomization, action/obs match, control rate, latency, failure modes.
3. The agent ALSO references `roboforge:safe-hardware-deployment` for the actual motion run.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write negative transcript**

Create `roboforge/tests/pressure-tests/sim2real-checklist/negative.md`:

```markdown
# Negative test — sim2real-checklist

**User message:**

> Tune the friction coefficient in my Isaac Lab task to better match real.

**Pass criteria:**

1. The agent does NOT invoke `roboforge:sim2real-checklist` (this is a sim parameter tweak, no deployment).
2. The agent answers the friction-tuning question (will likely invoke `isaac-lab-workflows` once Plan 3 ships).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'sim2real-checklist' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/sim2real-checklist/ \
        roboforge/tests/pressure-tests/sim2real-checklist/
git commit -m "roboforge: add sim2real-checklist skill + pressure tests"
```

---

## Task 6: `ros2-node-discipline`

**Files:**
- Create: `roboforge/skills/ros2-node-discipline/SKILL.md`
- Create: `roboforge/tests/pressure-tests/ros2-node-discipline/{README.md,positive.md,negative.md}`

- [ ] **Step 1: Create directories**

```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/skills/ros2-node-discipline \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/ros2-node-discipline
```

- [ ] **Step 2: Write the skill**

Create `roboforge/skills/ros2-node-discipline/SKILL.md`:

````markdown
---
name: ros2-node-discipline
description: Use when creating a new ROS 2 node, adding publishers/subscribers, or changing QoS, lifecycle, parameters, or the choice between actions/services/topics. Catches the design errors that produce "the controller drops messages" and "my parameter changes don't take effect" downstream.
---

Most "flaky ROS 2 system" complaints are not flakiness — they're QoS mismatches, missing parameter declarations, or the wrong primitive (topic where it should be a service, service where it should be an action). This skill enforces the design choice up front.

## When this skill applies

Invoke this skill when:
- Creating a new ROS 2 node (Python or C++).
- Adding a publisher, subscriber, service, or action server/client.
- Changing a QoS profile.
- Converting a node to lifecycle.
- Adding or modifying parameters.
- Refactoring across the action/service/topic boundary.

This skill does NOT apply to:
- Pure colcon build / dependency edits without code changes.
- Editing launch files only (use `ros2-patterns` once Plan 3 ships).

## Required checklist

1. **QoS matches the topic class.** Use the right profile for the data:
   - **Sensor data** (camera, lidar, IMU, joint_states): `BEST_EFFORT`, `KEEP_LAST` with depth ~5–10, `VOLATILE` durability. Late subscribers don't get history.
   - **Control commands** (`cmd_vel`, joint commands, action goals): `RELIABLE`, `KEEP_LAST(1)`, `VOLATILE`. Newest matters; deliver it.
   - **Maps, configs, latched state** (occupancy_grid, robot_description, lifecycle state): `RELIABLE`, `KEEP_LAST(1)`, `TRANSIENT_LOCAL`. Latched — late subscribers receive the most recent.
   - **Diagnostics**: `RELIABLE`, `KEEP_LAST(10)`, `VOLATILE`.
   Mismatch = silently dropped messages. Pub and sub QoS must be compatible — not identical, compatible. Check the ROS 2 QoS compatibility table.
2. **Lifecycle decision.** Is this node part of a larger startup sequence (planner, controller, sensor driver)? If yes, make it a `LifecycleNode`. Implement `on_configure`, `on_activate`, `on_deactivate`, `on_cleanup`. Don't touch resources outside the matching state.
3. **Parameters declared, typed, and validated.** Every parameter goes through `declare_parameter(name, default, descriptor)`. Set the type. Set min/max where bounded. Otherwise a string parameter silently passes a bad value through to a float consumer.
4. **Action vs service vs topic.** Pick deliberately:
   - **Topic** — continuous data flow. No reply. No cancel. (`/cmd_vel`, `/joint_states`, `/scan`.)
   - **Service** — request/response, fast, no progress feedback, no cancel. (`/get_parameters`, `/spawn_entity`.)
   - **Action** — request/response, long-running, with feedback and cancel. (`/navigate_to_pose`, `/follow_joint_trajectory`, `/move_group/move_action`.)
   The wrong primitive is a recurring source of "this hangs", "I can't cancel it", "I lose feedback during long ops".
5. **Namespacing planned for multi-robot.** Even single-robot today, prefix with a parameterized namespace (`{ns}/joint_states`, not `/joint_states`). Free; saves a refactor later.
6. **Clean shutdown.** Cancel running actions on shutdown. Release hardware claims. Don't just rely on the destructor.
7. **Threading model explicit.** Single-threaded executor by default. Switch to multi-threaded only with reason (callbacks blocking each other, periodic timers + slow callbacks). Mark callback groups (Reentrant vs MutuallyExclusive) when you do.

## Red flags

| Thought | Reality |
|---|---|
| "Default QoS is fine" | The default isn't a default; it's "Reliable, KeepLast(10), Volatile". That fights with sensor publishers. |
| "I'll make it a topic and ignore old ones" | If "ignore old" means latency, you wanted KeepLast(1). If it means cancel, you wanted an action. |
| "I'll set the parameter at runtime" | Without `declare_parameter`, runtime sets are silently dropped. |
| "Lifecycle is overkill for now" | Until startup ordering bites, then a refactor. Decide now. |
| "Single namespace forever" | The day you need a second robot, you'll regret it. |

## After the change

- Run `ros2 topic list` / `ros2 topic info -v` to confirm advertised QoS matches intended.
- For a publisher: `ros2 topic echo` from a subscriber with the *intended* subscriber QoS to confirm compatibility.
- For parameters: `ros2 param list <node>` and `ros2 param describe <node> <param>`.
- For lifecycle: `ros2 lifecycle list` / `ros2 lifecycle set`.
````

- [ ] **Step 3: Write pressure-test README**

Create `roboforge/tests/pressure-tests/ros2-node-discipline/README.md`:

```markdown
# Pressure tests — ros2-node-discipline

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled.
3. For each test, send the listed user message in a fresh `/clear`-ed session.
4. Paste the agent's response into the matching transcript file.

## Pass criteria

- `positive.md` — agent invokes `roboforge:ros2-node-discipline` before writing node code.
- `negative.md` — agent does NOT invoke the skill.
```

- [ ] **Step 4: Write positive transcript**

Create `roboforge/tests/pressure-tests/ros2-node-discipline/positive.md`:

```markdown
# Positive test — ros2-node-discipline

**User message:**

> Write me a ROS 2 Python node that subscribes to /scan and republishes a downsampled version on /scan_low.

**Pass criteria:**

1. The agent invokes `roboforge:ros2-node-discipline` before producing the node code.
2. The agent walks QoS choice (sensor data → BEST_EFFORT, KEEP_LAST), parameter declaration (downsampling factor), and namespacing.
3. The resulting node uses sensor-appropriate QoS, not the rclpy default.

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 5: Write negative transcript**

Create `roboforge/tests/pressure-tests/ros2-node-discipline/negative.md`:

```markdown
# Negative test — ros2-node-discipline

**User message:**

> What ROS 2 distribution should I use on Ubuntu 24.04?

**Pass criteria:**

1. The agent does NOT invoke `roboforge:ros2-node-discipline` (no node is being created or modified).
2. The agent answers directly (Jazzy is the LTS for 24.04).

**Transcript:**

(Paste actual session transcript here.)

**Result:** PASS / FAIL

**Notes:**
```

- [ ] **Step 6: Smoke-test session-start**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'ros2-node-discipline' in ctx else 'FAIL')"
```
Expected: `OK`.

- [ ] **Step 7: Commit**

```bash
git add roboforge/skills/ros2-node-discipline/ \
        roboforge/tests/pressure-tests/ros2-node-discipline/
git commit -m "roboforge: add ros2-node-discipline skill + pressure tests"
```

---

## Task 7: Manual session-test sweep

This is a single batch verification pass. The agent (in this session) cannot run fresh Claude Code sessions, so the human partner runs the twelve transcripts (6 positive + 6 negative) and records results.

- [ ] **Step 1: Reinstall the plugin so new skills are loaded**

In a Claude Code session:

```text
/plugin marketplace remove roboforge-dev
/plugin marketplace add /home/phoenix/vla/superpowers/roboforge
/plugin install roboforge@roboforge-dev
/clear
```

- [ ] **Step 2: Run all 12 pressure tests**

For each of the 6 skills, in a fresh `/clear`-ed session:

1. Send the **positive** user message from `roboforge/tests/pressure-tests/<skill>/positive.md`. Confirm the agent invokes the skill and walks the checklist. Paste transcript into the file. Set `Result:` to `PASS` or `FAIL`.
2. `/clear`. Send the **negative** user message. Confirm the agent does NOT invoke the skill. Paste transcript. Set `Result:`.

The skills, in order:

1. `safe-hardware-deployment`
2. `dataset-versioning`
3. `policy-eval-protocol`
4. `urdf-and-kinematics-changes`
5. `sim2real-checklist`
6. `ros2-node-discipline`

- [ ] **Step 3: Triage failures (if any)**

A FAIL on a positive test means the description in the skill's frontmatter isn't triggering. Tighten or expand the description's "Use when…" clause.

A FAIL on a negative test means the description is over-triggering. Constrain it.

Re-run any failed test after editing.

- [ ] **Step 4: Commit transcripts**

```bash
git add roboforge/tests/pressure-tests/
git commit -m "roboforge: record pressure-test results for methodology skills"
```

---

## Self-Review

**1. Spec coverage** — Plan 2 implements the methodology row of the spec's skill catalog (6 skills). Each skill in the catalog (`safe-hardware-deployment`, `dataset-versioning`, `policy-eval-protocol`, `urdf-and-kinematics-changes`, `sim2real-checklist`, `ros2-node-discipline`) has a dedicated task. Knowledge skills (Plan 3) and scaffold skills (Plan 4) are correctly out of scope.

**2. Placeholder scan** — every step has either complete file content, an exact command, or an explicit manual instruction with pass criteria. No "TBD" / "TODO" / "implement later" / "fill in details". The transcript-paste sections in test files use the explicit `(Paste actual session transcript here.)` placeholder which is the intended template behavior, not a plan-level placeholder.

**3. Type / name consistency** — All 6 skill names match the spec's catalog and `using-roboforge`'s forward declaration exactly:
- `safe-hardware-deployment`
- `dataset-versioning`
- `policy-eval-protocol`
- `urdf-and-kinematics-changes`
- `sim2real-checklist`
- `ros2-node-discipline`

Cross-references between skills (`sim2real-checklist` references `safe-hardware-deployment`; `policy-eval-protocol` references `dataset-versioning`) use the same names consistently.

---

## Out of scope (explicitly)

- **Knowledge skills** (`ros2-patterns`, `lerobot-conventions`, `vla-policies-overview`, `isaac-lab-workflows`, `manipulation-primitives`, `humanoid-wbc`, `perception-stack`) — Plan 3.
- **Scaffold skills** (`scaffold-ros2-package`, `scaffold-lerobot-policy`, `scaffold-isaac-task`, `scaffold-training-entrypoint`, `scaffold-teleop-pipeline`) — Plan 4.
- **Multi-harness validation beyond Claude Code** — Plan 5.
- **CI for transcripts** — manual / human-judged for v1.
