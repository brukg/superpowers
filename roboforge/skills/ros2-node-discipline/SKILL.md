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
