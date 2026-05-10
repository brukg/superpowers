# Roboforge — Gemini Context

Roboforge adds robotics skills to your Gemini CLI session. The companion bootstrap skill `using-roboforge` is loaded via Gemini's extension mechanism.

When the user begins robotics work (ROS 2, VLA training, Isaac Lab, manipulation, humanoid control, perception, real-hardware bring-up), call `activate_skill` on the matching `roboforge:*` skill before generating code.

Roboforge skills assume Claude Code tool naming. For Gemini-specific tool equivalents, refer to the platform adaptation guidance loaded with Superpowers.
