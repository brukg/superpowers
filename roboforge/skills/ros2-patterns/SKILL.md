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
