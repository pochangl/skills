---
name: describe
description: Listen to the user's description and write it down when they are done. Only triggered via the /describe command.
user_invocable: true
---

# Describe Skill

## Procedure

1. When triggered, enter listening mode. Acknowledge that you are ready and wait for the user to describe what they want captured.
2. As the user provides information across multiple messages, acknowledge each message briefly (e.g. "Got it.", "Noted.", "Continue.") without writing anything yet.
3. When the user indicates they are done (e.g. "done", "that's it", "finished"), compile everything they described into a well-structured Markdown file.
4. Write the output to a `.md` file in an appropriate location.
