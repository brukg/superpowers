# Roboforge Plan 1 of 5 — Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a working empty `roboforge` plugin that loads at SessionStart on Claude Code, exposes a `using-roboforge` bootstrap skill, and coexists cleanly with the `superpowers` plugin without name collisions.

**Architecture:** A nested sub-plugin under `/home/phoenix/vla/superpowers/roboforge/`. Mirrors Superpowers' multi-harness packaging exactly — per-harness manifests at the plugin root, hooks in `hooks/`, the canonical content under `skills/`. This plan only creates the foundation. Domain skills (methodology, knowledge, scaffolding) are scheduled for Plans 2–4. Cross-harness sync scripts are scheduled for Plan 5.

**Tech Stack:** Bash (hooks), JSON (manifests), Markdown (skills + context files). No Python or Node runtime. Targets Claude Code first; manifests for Codex / Cursor / Gemini / OpenCode are written but only Claude Code is validated in this plan.

**Reference spec:** `docs/superpowers/specs/2026-05-10-roboforge-plugin-design.md`

**Source plugin to mirror:** `/home/phoenix/vla/superpowers/` (same repo — the existing Superpowers core lives at the repo root; we're adding `roboforge/` as a sibling subtree).

---

## File Structure

Files to create in this plan:

| Path                                                       | Responsibility                                                          |
|------------------------------------------------------------|-------------------------------------------------------------------------|
| `roboforge/.claude-plugin/plugin.json`                     | Claude Code plugin manifest (`name: "roboforge"`).                      |
| `roboforge/.codex-plugin/plugin.json`                      | Codex CLI plugin manifest.                                              |
| `roboforge/.cursor-plugin/plugin.json`                     | Cursor plugin manifest.                                                 |
| `roboforge/gemini-extension.json`                          | Gemini CLI extension descriptor.                                        |
| `roboforge/package.json`                                   | OpenCode entry (also satisfies tools that probe for `package.json`).    |
| `roboforge/CLAUDE.md`                                      | Per-project instructions for Claude Code sessions.                      |
| `roboforge/AGENTS.md`                                      | Per-project instructions for Copilot CLI / Codex / Factory.             |
| `roboforge/GEMINI.md`                                      | Per-project instructions for Gemini CLI.                                |
| `roboforge/README.md`                                      | Human-facing description, install, status.                              |
| `roboforge/hooks/hooks.json`                               | Claude Code SessionStart hook config.                                   |
| `roboforge/hooks/hooks-cursor.json`                        | Cursor SessionStart hook config.                                        |
| `roboforge/hooks/run-hook.cmd`                             | Cross-platform polyglot wrapper (verbatim copy of Superpowers'). |
| `roboforge/hooks/session-start`                            | Bash script: emits `using-roboforge` content as SessionStart context.   |
| `roboforge/skills/using-roboforge/SKILL.md`                | Bootstrap skill — tells the agent roboforge skills exist and how to invoke. |
| `roboforge/tests/pressure-tests/bootstrap/README.md`       | How to run the bootstrap acceptance test.                               |
| `roboforge/tests/pressure-tests/bootstrap/acceptance.md`   | Recorded transcript of the acceptance test (filled in during Task 6).   |
| `roboforge/tests/pressure-tests/bootstrap/anti-collision.md` | Recorded transcript of the anti-collision test (filled in during Task 7). |

No existing files are modified.

---

## Task 1: Plugin directory scaffold + Claude Code manifest

**Files:**
- Create: `roboforge/.claude-plugin/plugin.json`

- [ ] **Step 1: Create the directory tree**

Run:
```bash
mkdir -p /home/phoenix/vla/superpowers/roboforge/.claude-plugin \
         /home/phoenix/vla/superpowers/roboforge/.codex-plugin \
         /home/phoenix/vla/superpowers/roboforge/.cursor-plugin \
         /home/phoenix/vla/superpowers/roboforge/hooks \
         /home/phoenix/vla/superpowers/roboforge/skills/using-roboforge \
         /home/phoenix/vla/superpowers/roboforge/tests/pressure-tests/bootstrap
```

Expected: no output, all directories created.

- [ ] **Step 2: Write the Claude Code manifest**

Create `roboforge/.claude-plugin/plugin.json` with this exact content:

```json
{
  "name": "roboforge",
  "description": "Robotics-engineer skills for AI coding agents: ROS 2, VLA / LeRobot policies, Isaac Lab, manipulation, humanoid WBC, perception, and real-hardware deployment discipline. Sibling to Superpowers core.",
  "version": "0.1.0",
  "author": {
    "name": "brukg",
    "email": "bruk@signalbotics.com"
  },
  "homepage": "https://github.com/brukg/superpowers",
  "repository": "https://github.com/brukg/superpowers",
  "license": "MIT",
  "keywords": [
    "robotics",
    "ros2",
    "vla",
    "lerobot",
    "isaac-lab",
    "manipulation",
    "humanoid",
    "perception",
    "skills"
  ]
}
```

- [ ] **Step 3: Verify the file is valid JSON**

Run: `python3 -m json.tool /home/phoenix/vla/superpowers/roboforge/.claude-plugin/plugin.json`
Expected: pretty-printed JSON, exit 0. (Any error means malformed JSON — fix and re-run.)

- [ ] **Step 4: Commit**

```bash
git add roboforge/.claude-plugin/plugin.json
git commit -m "roboforge: scaffold plugin tree and Claude Code manifest"
```

---

## Task 2: Manifests for the other harnesses

**Files:**
- Create: `roboforge/.codex-plugin/plugin.json`
- Create: `roboforge/.cursor-plugin/plugin.json`
- Create: `roboforge/gemini-extension.json`
- Create: `roboforge/package.json`

These are written now so the plugin can be installed across harnesses, but only Claude Code is validated end-to-end in this plan. Plan 5 covers per-harness validation and the sync script that keeps them aligned.

- [ ] **Step 1: Write the Codex CLI manifest**

Create `roboforge/.codex-plugin/plugin.json`:

```json
{
  "name": "roboforge",
  "version": "0.1.0",
  "description": "Robotics-engineer skills: ROS 2, VLA / LeRobot policies, Isaac Lab, manipulation, humanoid WBC, perception, real-hardware deployment discipline.",
  "author": {
    "name": "brukg",
    "email": "bruk@signalbotics.com",
    "url": "https://github.com/brukg"
  },
  "homepage": "https://github.com/brukg/superpowers",
  "repository": "https://github.com/brukg/superpowers",
  "license": "MIT",
  "keywords": [
    "robotics",
    "ros2",
    "vla",
    "lerobot",
    "isaac-lab",
    "skills"
  ],
  "skills": "./skills/",
  "interface": {
    "displayName": "Roboforge",
    "shortDescription": "Robotics skills for coding agents",
    "longDescription": "Roboforge surfaces robotics-specific skills: ROS 2 patterns, LeRobot/VLA conventions, Isaac Lab workflows, manipulation primitives, humanoid WBC, perception, and the discipline you want before deploying anything on real hardware.",
    "developerName": "brukg",
    "category": "Coding",
    "capabilities": [
      "Interactive",
      "Read",
      "Write"
    ],
    "defaultPrompt": [
      "Help me set up a ROS 2 node.",
      "I need to deploy this policy on real hardware."
    ],
    "websiteURL": "https://github.com/brukg/superpowers",
    "brandColor": "#0EA5E9"
  }
}
```

- [ ] **Step 2: Write the Cursor manifest**

Create `roboforge/.cursor-plugin/plugin.json`:

```json
{
  "name": "roboforge",
  "version": "0.1.0",
  "description": "Robotics skills for AI coding agents.",
  "author": {
    "name": "brukg",
    "email": "bruk@signalbotics.com"
  },
  "license": "MIT"
}
```

- [ ] **Step 3: Write the Gemini extension descriptor**

Create `roboforge/gemini-extension.json`:

```json
{
  "name": "roboforge",
  "description": "Robotics-engineer skills: ROS 2, VLA, Isaac Lab, manipulation, humanoid WBC, perception, real-hardware discipline.",
  "version": "0.1.0",
  "contextFileName": "GEMINI.md"
}
```

- [ ] **Step 4: Write package.json**

Create `roboforge/package.json`:

```json
{
  "name": "roboforge",
  "version": "0.1.0",
  "description": "Robotics skills plugin for AI coding agents.",
  "type": "module",
  "license": "MIT"
}
```

- [ ] **Step 5: Validate all four files are valid JSON**

Run:
```bash
for f in \
  /home/phoenix/vla/superpowers/roboforge/.codex-plugin/plugin.json \
  /home/phoenix/vla/superpowers/roboforge/.cursor-plugin/plugin.json \
  /home/phoenix/vla/superpowers/roboforge/gemini-extension.json \
  /home/phoenix/vla/superpowers/roboforge/package.json; do
  python3 -m json.tool "$f" > /dev/null && echo "OK: $f" || echo "FAIL: $f"
done
```

Expected: four `OK:` lines.

- [ ] **Step 6: Commit**

```bash
git add roboforge/.codex-plugin/plugin.json \
        roboforge/.cursor-plugin/plugin.json \
        roboforge/gemini-extension.json \
        roboforge/package.json
git commit -m "roboforge: add Codex/Cursor/Gemini/OpenCode manifests"
```

---

## Task 3: Hooks (run-hook wrapper, SessionStart configs, session-start script)

**Files:**
- Create: `roboforge/hooks/run-hook.cmd` (verbatim copy of `hooks/run-hook.cmd`)
- Create: `roboforge/hooks/hooks.json`
- Create: `roboforge/hooks/hooks-cursor.json`
- Create: `roboforge/hooks/session-start`

- [ ] **Step 1: Copy the cross-platform wrapper verbatim**

Run:
```bash
cp /home/phoenix/vla/superpowers/hooks/run-hook.cmd \
   /home/phoenix/vla/superpowers/roboforge/hooks/run-hook.cmd
```

- [ ] **Step 2: Confirm the copy is byte-identical**

Run:
```bash
diff /home/phoenix/vla/superpowers/hooks/run-hook.cmd \
     /home/phoenix/vla/superpowers/roboforge/hooks/run-hook.cmd
```

Expected: no output (files are identical), exit 0.

- [ ] **Step 3: Write the Claude Code SessionStart hook config**

Create `roboforge/hooks/hooks.json` with this exact content:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 4: Write the Cursor SessionStart hook config**

Create `roboforge/hooks/hooks-cursor.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CURSOR_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 5: Write the session-start bash script**

Create `roboforge/hooks/session-start` with this exact content:

```bash
#!/usr/bin/env bash
# SessionStart hook for roboforge plugin.
# Loads the using-roboforge bootstrap skill content and injects it as
# additionalContext so the agent learns roboforge skills exist.

set -euo pipefail

# Determine plugin root directory
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

# Read using-roboforge content
using_roboforge_content=$(cat "${PLUGIN_ROOT}/skills/using-roboforge/SKILL.md" 2>&1 || echo "Error reading using-roboforge skill")

# Escape string for JSON embedding using bash parameter substitution.
# Each ${s//old/new} is a single C-level pass.
escape_for_json() {
    local s="$1"
    s="${s//\\/\\\\}"
    s="${s//\"/\\\"}"
    s="${s//$'\n'/\\n}"
    s="${s//$'\r'/\\r}"
    s="${s//$'\t'/\\t}"
    printf '%s' "$s"
}

using_roboforge_escaped=$(escape_for_json "$using_roboforge_content")
session_context="<EXTREMELY_IMPORTANT>\nYou have roboforge \xe2\x80\x94 robotics-specific skills that complement Superpowers core. Superpowers handles process discipline (brainstorming, TDD, debugging). Roboforge adds robotics methodology, domain knowledge, and scaffolding.\n\n**Below is the full content of your 'roboforge:using-roboforge' skill - your introduction to roboforge. For all other roboforge skills, use the 'Skill' tool:**\n\n${using_roboforge_escaped}\n</EXTREMELY_IMPORTANT>"

# Output context injection as JSON.
# Cursor hooks expect additional_context (snake_case).
# Claude Code hooks expect hookSpecificOutput.additionalContext (nested).
# Copilot CLI and others expect additionalContext (top-level, SDK standard).
#
# Uses printf instead of heredoc to work around bash 5.3+ heredoc hang.
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
  printf '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi

exit 0
```

- [ ] **Step 6: Make the session-start script executable**

Run:
```bash
chmod +x /home/phoenix/vla/superpowers/roboforge/hooks/session-start
chmod +x /home/phoenix/vla/superpowers/roboforge/hooks/run-hook.cmd
```

- [ ] **Step 7: Smoke-test the script in isolation**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | head -3
```

Expected: JSON output that begins with `{` and includes `"hookSpecificOutput"`. (The `using-roboforge` skill doesn't exist yet, so the embedded content will say "Error reading using-roboforge skill" — that's expected at this stage. Real content arrives in Task 4.)

- [ ] **Step 8: Validate the two hook config JSON files**

Run:
```bash
for f in /home/phoenix/vla/superpowers/roboforge/hooks/hooks.json \
         /home/phoenix/vla/superpowers/roboforge/hooks/hooks-cursor.json; do
  python3 -m json.tool "$f" > /dev/null && echo "OK: $f" || echo "FAIL: $f"
done
```

Expected: two `OK:` lines.

- [ ] **Step 9: Commit**

```bash
git add roboforge/hooks/
git commit -m "roboforge: add SessionStart hooks for Claude Code and Cursor"
```

---

## Task 4: Bootstrap skill — `using-roboforge`

**Files:**
- Create: `roboforge/skills/using-roboforge/SKILL.md`

This is the load-bearing skill. Without it, none of the future roboforge skills will auto-trigger — they'll just be files on disk. The frontmatter `description` is what the harness reads to decide when to surface the skill, so phrasing matters.

- [ ] **Step 1: Write the bootstrap skill**

Create `roboforge/skills/using-roboforge/SKILL.md` with this exact content:

````markdown
---
name: using-roboforge
description: Use when starting any conversation involving robotics work — ROS 2, VLA / LeRobot policies, Isaac Lab, manipulation, humanoid control, perception pipelines, or real-hardware deployment. Establishes how to find and use roboforge skills, requiring Skill tool invocation before robotics-related actions.
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If a roboforge skill applies to your robotics task, you MUST invoke it before acting. Roboforge skills exist because robotics work has failure modes — physical, financial, scientific — that generic coding skills do not catch. Skipping them is not a productivity gain; it is the path to a damaged robot, a wasted training run, or a benchmark you cannot reproduce.
</EXTREMELY-IMPORTANT>

## Relationship to Superpowers core

Roboforge **adds to** Superpowers core. It does not replace it.

| Concern                                  | Owned by              |
|------------------------------------------|-----------------------|
| Brainstorming a feature/spec             | `superpowers:brainstorming` |
| Writing or executing an implementation plan | `superpowers:writing-plans`, `superpowers:executing-plans` |
| Test-driven development                  | `superpowers:test-driven-development` |
| Systematic debugging                     | `superpowers:systematic-debugging` |
| Code review                              | `superpowers:requesting-code-review`, `superpowers:receiving-code-review` |
| Robotics-specific methodology, knowledge, scaffolding | `roboforge:*` |

**Rule:** Always run the relevant Superpowers process skill first. Roboforge skills layer on top.

Examples:
- "Let's add a new manipulation skill." → `superpowers:brainstorming` first, then roboforge knowledge/scaffold skills as needed.
- "Fix this controller crash." → `superpowers:systematic-debugging` first, then roboforge skills for any hardware-deployment steps.
- "Train a diffusion policy on my SO-100 dataset." → `superpowers:brainstorming`, then `roboforge:dataset-versioning`, then `roboforge:policy-eval-protocol` before claiming success.

## When to invoke a roboforge skill

Use the `Skill` tool with `roboforge:<skill-name>` whenever any of the following are true:

- **About to send commands to real hardware** (a motor, gripper, mobile base, humanoid, IMU-driven actuator). Invoke `roboforge:safe-hardware-deployment` before any code path that actuates physical motion.
- **About to train, retrain, or append episodes to a dataset.** Invoke `roboforge:dataset-versioning`.
- **About to claim a policy "works".** Invoke `roboforge:policy-eval-protocol`.
- **Editing a URDF, Xacro, or SRDF.** Invoke `roboforge:urdf-and-kinematics-changes`.
- **Deploying a sim-trained policy on real hardware.** Invoke `roboforge:sim2real-checklist`.
- **Creating or modifying a ROS 2 node.** Invoke `roboforge:ros2-node-discipline`.
- **Working with LeRobot datasets, policies, or training loops.** Invoke `roboforge:lerobot-conventions`.
- **Picking a VLA / IL policy.** Invoke `roboforge:vla-policies-overview`.
- **Working in Isaac Sim or Isaac Lab.** Invoke `roboforge:isaac-lab-workflows`.
- **Writing manipulation code.** Invoke `roboforge:manipulation-primitives`.
- **Working on humanoid control or motion retargeting.** Invoke `roboforge:humanoid-wbc`.
- **Touching cameras, depth, or time-sync.** Invoke `roboforge:perception-stack`.
- **Editing launch files, params, namespaces, or DDS configs.** Invoke `roboforge:ros2-patterns`.
- **Asked to scaffold a ROS 2 package, LeRobot policy, Isaac task, training entrypoint, or teleop pipeline.** Invoke the matching `roboforge:scaffold-*` skill.

> **Note:** Domain skills are added in Plans 2–4 of the roboforge build. Until those land, this skill enumerates them as a forward declaration. If a skill listed here is not yet present when you try to invoke it, fall back to acting carefully and surface to the user that the skill is pending.

## Anti-collision

If you find yourself choosing between a Superpowers core skill and a roboforge skill that could both apply, **the core process skill wins for the planning step**, and the roboforge skill applies to the implementation step it governs.

Concretely: do not invoke `roboforge:*` to design a feature, write a spec, or run TDD. Those are core. Do invoke `roboforge:*` for the robotics-specific guardrails on top.

## Default tech stack

Roboforge skills assume the SOTA / commonly-used stack unless stated otherwise:

- **ROS 2** Humble or Jazzy.
- **PyTorch + HuggingFace Hub**, with **LeRobot** dataset/policy conventions and **OpenVLA-family** policies (OpenVLA, Octo, Pi0, RDT2, GR00T, ACT, diffusion policy).
- **Isaac Sim / Isaac Lab** for GPU sim and sim2real.
- **Real hardware**: Franka, UR, SO-100/101, Unitree, Aloha; RealSense / ZED cameras; IMUs; teleop (Quest, leader-follower).

If the user's stack diverges, ask before applying skill defaults.
````

- [ ] **Step 2: Verify the skill file exists and has frontmatter**

Run:
```bash
head -4 /home/phoenix/vla/superpowers/roboforge/skills/using-roboforge/SKILL.md
```

Expected: starts with `---`, contains `name: using-roboforge`, contains `description:`, ends second `---` block.

- [ ] **Step 3: Re-run the session-start script and confirm it picks up the skill**

Run:
```bash
CLAUDE_PLUGIN_ROOT=/home/phoenix/vla/superpowers/roboforge \
  /home/phoenix/vla/superpowers/roboforge/hooks/session-start | python3 -c "import sys, json; d=json.load(sys.stdin); ctx=d['hookSpecificOutput']['additionalContext']; print('OK' if 'using-roboforge' in ctx and 'safe-hardware-deployment' in ctx else 'FAIL'); print('len:', len(ctx))"
```

Expected: `OK` followed by a `len:` value greater than ~3000.

- [ ] **Step 4: Commit**

```bash
git add roboforge/skills/using-roboforge/SKILL.md
git commit -m "roboforge: add using-roboforge bootstrap skill"
```

---

## Task 5: Per-harness context files and README

**Files:**
- Create: `roboforge/CLAUDE.md`
- Create: `roboforge/AGENTS.md`
- Create: `roboforge/GEMINI.md`
- Create: `roboforge/README.md`

These files are the per-project bootstrap context that some harnesses read on session start (independent of the SessionStart hook). They reinforce the same message as `using-roboforge` for harnesses that don't run hooks.

- [ ] **Step 1: Write CLAUDE.md**

Create `roboforge/CLAUDE.md`:

```markdown
# Roboforge — Project Instructions

This plugin adds robotics-specific skills to your coding agent: ROS 2, VLA / LeRobot policies, Isaac Lab, manipulation, humanoid WBC, perception, and real-hardware deployment discipline.

Roboforge does **not** replace Superpowers core. Always run the relevant Superpowers process skill (brainstorming, TDD, debugging, planning) first; roboforge skills layer on top with robotics-specific guardrails.

If you are starting a session in this plugin's context, the SessionStart hook will load `using-roboforge` for you. If for any reason it didn't, invoke `roboforge:using-roboforge` manually before doing robotics work.
```

- [ ] **Step 2: Write AGENTS.md**

Create `roboforge/AGENTS.md` (Codex, Copilot CLI, Factory all read this):

```markdown
# Roboforge — Agent Instructions

Roboforge is a robotics skills plugin for AI coding agents. It complements Superpowers core (process skills) by adding domain-specific methodology, knowledge, and scaffolding for ROS 2, VLA / LeRobot, Isaac Lab, manipulation, humanoid control, perception, and real-hardware deployment.

**On session start:** invoke the `using-roboforge` skill before any robotics work. It enumerates the available roboforge skills and explains when to use each one.

**Skill priority:** Superpowers core process skills (brainstorming, TDD, debugging) always run first. Roboforge methodology skills run before the implementation step they govern.
```

- [ ] **Step 3: Write GEMINI.md**

Create `roboforge/GEMINI.md`:

```markdown
# Roboforge — Gemini Context

Roboforge adds robotics skills to your Gemini CLI session. The companion bootstrap skill `using-roboforge` is loaded via Gemini's extension mechanism.

When the user begins robotics work (ROS 2, VLA training, Isaac Lab, manipulation, humanoid control, perception, real-hardware bring-up), call `activate_skill` on the matching `roboforge:*` skill before generating code.

Roboforge skills assume Claude Code tool naming. For Gemini-specific tool equivalents, refer to the platform adaptation guidance loaded with Superpowers.
```

- [ ] **Step 4: Write README.md**

Create `roboforge/README.md`:

````markdown
# Roboforge

Robotics skills plugin for AI coding agents. Adds methodology, domain knowledge, and scaffolding for ROS 2, VLA / LeRobot, Isaac Lab, manipulation, humanoid WBC, perception, and real-hardware deployment.

Roboforge is a sibling plugin to [Superpowers](https://github.com/obra/superpowers). It does not replace Superpowers — it layers domain-specific guardrails on top.

## Status

`v0.1.0` — foundation only. Bootstrap skill (`using-roboforge`) is in place and the plugin loads cleanly on Claude Code. Domain skills (methodology, knowledge, scaffolding) ship in subsequent plans.

## Install (Claude Code, local)

From the repo root:

```bash
/plugin install /home/phoenix/vla/superpowers/roboforge
```

## What lives where

```
roboforge/
├── .claude-plugin/plugin.json   # Claude Code manifest
├── .codex-plugin/plugin.json    # Codex CLI manifest
├── .cursor-plugin/plugin.json   # Cursor manifest
├── gemini-extension.json        # Gemini CLI descriptor
├── package.json                 # OpenCode entry
├── hooks/                       # SessionStart wiring
└── skills/
    └── using-roboforge/         # bootstrap (others land in Plans 2–4)
```

## Coexistence

`roboforge` co-installs with `superpowers`. The bootstraps (`using-superpowers`, `using-roboforge`) explicitly defer to each other in their respective domains.
````

- [ ] **Step 5: Commit**

```bash
git add roboforge/CLAUDE.md \
        roboforge/AGENTS.md \
        roboforge/GEMINI.md \
        roboforge/README.md
git commit -m "roboforge: add per-harness context files and README"
```

---

## Task 6: Bootstrap acceptance test (Claude Code)

**Files:**
- Create: `roboforge/tests/pressure-tests/bootstrap/README.md`
- Create: `roboforge/tests/pressure-tests/bootstrap/acceptance.md`

This is a manual test executed in a fresh Claude Code session. It proves the SessionStart hook actually injects `using-roboforge` and the agent recognizes it.

- [ ] **Step 1: Write the test runner README**

Create `roboforge/tests/pressure-tests/bootstrap/README.md`:

```markdown
# Bootstrap Pressure Tests

These tests run manually in a fresh Claude Code session. They prove that the SessionStart hook successfully injects `using-roboforge` and that the agent invokes the right skills (or the right *kinds* of skills) when it sees robotics-related user messages.

## How to run

1. Open a clean Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm both `superpowers` and `roboforge` plugins are installed and enabled.
3. Run each test below by sending the listed user message.
4. Copy the agent's response into the matching transcript file.

## Tests

- `acceptance.md` — proves `using-roboforge` loads and the agent recognizes a robotics task.
- `anti-collision.md` — proves `superpowers:brainstorming` still wins for pure design questions.

## Pass criteria per test

See the front of each transcript file for the specific pass criteria.
```

- [ ] **Step 2: Write the acceptance test transcript template**

Create `roboforge/tests/pressure-tests/bootstrap/acceptance.md`:

```markdown
# Bootstrap acceptance test

**User message:**

> Let's train a diffusion policy on my SO-100 dataset.

**Pass criteria:**

1. The agent invokes `superpowers:brainstorming` first (per skill priority — design before implementation).
2. As the brainstorming progresses, the agent surfaces awareness of `roboforge:dataset-versioning` and `roboforge:policy-eval-protocol` as skills that will apply during implementation.
3. The agent does NOT skip straight to writing training code.

**Transcript:**

(Paste the actual session transcript here after running the test.)

**Result:** PASS / FAIL

**Notes:**

(Anything observed worth fixing in the bootstrap.)
```

- [ ] **Step 3: Run the acceptance test**

Manual:

1. Open a fresh Claude Code session in `/home/phoenix/vla/superpowers/`.
2. Confirm `roboforge` is installed and enabled (`/plugin` → list).
3. Send the user message exactly: `Let's train a diffusion policy on my SO-100 dataset.`
4. Save the agent's response.

- [ ] **Step 4: Record the result**

Edit `acceptance.md` and paste the agent's transcript into the "Transcript" section. Set `Result:` to `PASS` if all three pass criteria are met, otherwise `FAIL` and note what went wrong in `Notes`.

If FAIL: stop and fix the bootstrap — usually the description phrasing or the SessionStart context wrapper. Re-run the test before proceeding.

- [ ] **Step 5: Commit**

```bash
git add roboforge/tests/pressure-tests/bootstrap/README.md \
        roboforge/tests/pressure-tests/bootstrap/acceptance.md
git commit -m "roboforge: add bootstrap acceptance test (Claude Code)"
```

---

## Task 7: Anti-collision test

**Files:**
- Create: `roboforge/tests/pressure-tests/bootstrap/anti-collision.md`

Proves that pure design / process work still routes to Superpowers core, not to roboforge.

- [ ] **Step 1: Write the anti-collision test transcript template**

Create `roboforge/tests/pressure-tests/bootstrap/anti-collision.md`:

```markdown
# Anti-collision test

**User message:**

> Help me brainstorm a new manipulation skill.

**Pass criteria:**

1. The agent invokes `superpowers:brainstorming` (NOT a roboforge skill).
2. The agent does NOT invoke `roboforge:manipulation-primitives` for the design step.
3. The agent may *mention* that `roboforge:manipulation-primitives` will apply later, during implementation. That is correct behavior.

**Transcript:**

(Paste the actual session transcript here after running the test.)

**Result:** PASS / FAIL

**Notes:**

(Anything observed worth fixing in the bootstrap or in `using-roboforge`'s anti-collision section.)
```

- [ ] **Step 2: Run the anti-collision test**

Manual:

1. Open another fresh Claude Code session (or `/clear`) in `/home/phoenix/vla/superpowers/`.
2. Send: `Help me brainstorm a new manipulation skill.`
3. Save the agent's response.

- [ ] **Step 3: Record the result**

Edit `anti-collision.md`, paste the transcript, set `Result:` to `PASS` or `FAIL`.

If FAIL: the most likely cause is the `using-roboforge` description being too eager. Tighten the description (it should NOT match pure brainstorming). Re-run.

- [ ] **Step 4: Commit**

```bash
git add roboforge/tests/pressure-tests/bootstrap/anti-collision.md
git commit -m "roboforge: add bootstrap anti-collision test"
```

---

## Self-Review

Run this checklist after the plan's tasks are written, before handoff.

**1. Spec coverage** — Plan 1 implements:
- Plugin layout (Section "Architecture / Plugin Layout" of the spec) — Tasks 1–5.
- Multi-harness packaging skeleton (Section "Multi-Harness Packaging") — Task 2.
- Bootstrap (Section "Auto-Trigger Strategy / Bootstrap") — Tasks 3–4.
- Bootstrap acceptance test + anti-collision test (Section "Testing & Validation" items 2 and 3) — Tasks 6–7.

**Deferred to later plans (correctly):**
- The 18 domain skills (Plans 2–4).
- Per-skill pressure tests (Plans 2–4 alongside each skill).
- Cross-harness sync scripts and per-harness validation beyond Claude Code (Plan 5).

**2. Placeholder scan** — every step has either exact code, an exact command, or an explicit manual instruction with pass criteria. No "TBD" / "TODO" / "implement later" / "similar to Task N".

**3. Type / name consistency** — `roboforge` plugin name used throughout. Bootstrap skill name `using-roboforge` consistent in `session-start`, `SKILL.md`, README. Skill names referenced in `using-roboforge` (e.g. `safe-hardware-deployment`, `dataset-versioning`, `policy-eval-protocol`, `urdf-and-kinematics-changes`, `sim2real-checklist`, `ros2-node-discipline`, `lerobot-conventions`, `vla-policies-overview`, `isaac-lab-workflows`, `manipulation-primitives`, `humanoid-wbc`, `perception-stack`, `ros2-patterns`, `scaffold-*`) all match the spec's catalog exactly.

---

## Out of scope (explicitly)

- **Domain skills.** All 18 domain skills are forward-declared in `using-roboforge` but not implemented here. They land in Plans 2–4.
- **Sync script.** `scripts/sync-to-codex-plugin.sh` (and equivalents) is Plan 5.
- **Per-harness validation beyond Claude Code.** Plan 5.
- **Marketplace publication.** No `.claude-plugin/marketplace.json` is shipped in this plan; install is local-path until Plan 5.
- **CI for tests.** All pressure tests are manual / human-judged.
