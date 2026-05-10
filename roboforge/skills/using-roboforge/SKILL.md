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
