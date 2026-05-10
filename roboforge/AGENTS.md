# Roboforge — Agent Instructions

Roboforge is a robotics skills plugin for AI coding agents. It complements Superpowers core (process skills) by adding domain-specific methodology, knowledge, and scaffolding for ROS 2, VLA / LeRobot, Isaac Lab, manipulation, humanoid control, perception, and real-hardware deployment.

**On session start:** invoke the `using-roboforge` skill before any robotics work. It enumerates the available roboforge skills and explains when to use each one.

**Skill priority:** Superpowers core process skills (brainstorming, TDD, debugging) always run first. Roboforge methodology skills run before the implementation step they govern.
