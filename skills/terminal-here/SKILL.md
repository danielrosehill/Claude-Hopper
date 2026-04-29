---
name: terminal-here
description: Open a new Konsole terminal window at the current working directory. Use when the user asks to "open a terminal here", "new terminal here", or "konsole here".
---

# Open Terminal Here

Launch a new Konsole window with its working directory set to the current shell's `$PWD`. Useful as a lightweight companion to the session-transfer workflow — e.g. after installing a new MCP server, when the user needs a fresh shell at this directory to start a sibling session.

## Command

```bash
setsid konsole --workdir "$PWD" >/dev/null 2>&1 &
```

`setsid` + background detaches the new window so it survives after the current Claude Code session exits.

## Notes

- Do not block waiting on the process.
- Report which directory was opened.
