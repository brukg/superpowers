# Roboforge — Project Instructions

This plugin adds robotics-specific skills to your coding agent: ROS 2, VLA / LeRobot policies, Isaac Lab, manipulation, humanoid WBC, perception, and real-hardware deployment discipline.

Roboforge does **not** replace Superpowers core. Always run the relevant Superpowers process skill (brainstorming, TDD, debugging, planning) first; roboforge skills layer on top with robotics-specific guardrails.

If you are starting a session in this plugin's context, the SessionStart hook will load `using-roboforge` for you. If for any reason it didn't, invoke `roboforge:using-roboforge` manually before doing robotics work.
