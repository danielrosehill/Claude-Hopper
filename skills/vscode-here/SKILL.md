---
name: vscode-here
description: Open VS Code at the current working directory. Use when the user asks to "open in vs code", "open in vscode", "vscode here", or "code .".
---

# Open in VS Code

Launch VS Code with the current shell's `$PWD` as the workspace root. Companion to `terminal-here` — useful when the user wants to jump from a Claude session into the editor at the same directory.

## Command

```bash
setsid code "$PWD" >/dev/null 2>&1 &
```

`setsid` + background detaches the editor process so it survives after the current Claude Code session exits.

## Notes

- Do not block waiting on the process.
- Report which directory was opened.
- If `code` is not on `$PATH`, fall back to `codium` or report the missing binary rather than guessing a path.
