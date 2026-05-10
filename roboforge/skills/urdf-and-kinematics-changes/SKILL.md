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
