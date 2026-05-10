# Roboforge

Robotics skills plugin for AI coding agents. Adds methodology, domain knowledge, and scaffolding for ROS 2, VLA / LeRobot, Isaac Lab, manipulation, humanoid WBC, perception, and real-hardware deployment.

Roboforge is a sibling plugin to [Superpowers](https://github.com/obra/superpowers). It does not replace Superpowers — it layers domain-specific guardrails on top.

## Status

`v0.1.0` — foundation only. Bootstrap skill (`using-roboforge`) is in place and the plugin loads cleanly on Claude Code. Domain skills (methodology, knowledge, scaffolding) ship in subsequent plans.

## Install (Claude Code, local)

The plugin ships its own dev marketplace at `roboforge/.claude-plugin/marketplace.json`. Register the marketplace, then install the plugin from it:

```text
/plugin marketplace add /home/phoenix/vla/superpowers/roboforge
/plugin install roboforge@roboforge-dev
```

After install, `/clear` (or open a fresh session) so the SessionStart hook fires and `using-roboforge` loads.

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
