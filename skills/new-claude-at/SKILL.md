---
name: new-claude-at
description: Open a new Konsole window running a fresh Claude Code CLI session at an arbitrary path. Use when the user asks for "a new claude at <path>", "claude in <other repo>", "spawn claude over in <dir>", or similar. Unlike `new-claude-here`, this takes an explicit target directory. Supports optional seed prompt and self-destruct.
---

# New Claude Session At Path

Spawn a sibling Claude Code session in a new Konsole window at an **arbitrary** directory the user specifies. Companion to `new-claude-here` (which uses `$PWD`) — use this one when the target is somewhere else on disk.

## Inputs

- **Target path** — absolute path, or a path relative to the current repo. Required. If the user hasn't given one, ask. Expand `~` and resolve to an absolute path before launching.
- **Seed prompt** (optional) — an initial prompt or slash command to run in the new session (e.g. `/brainstorm-solutions:deep-research`). Passed as a positional argument to `claude`.
- **Self-destruct** (optional) — if the caller says "close this one", "self-destruct", or "hand off", close the current Konsole after the new one is verified. Default: keep current open.

## Steps

### 1. Resolve and verify target

```bash
target=$(readlink -f "<given-path>")
[ -d "$target" ] || { echo "not a directory: $target"; exit 1; }
```

### 2. Launch new session

IMPORTANT: When passing a seed prompt, wrap `claude` in `bash -c` so that Konsole sets the working directory correctly before Claude starts. Without this wrapper, Claude may open in the wrong directory.

If a seed prompt is provided:
```bash
setsid konsole --workdir "$target" -e bash -c 'claude "<seed-prompt>"' >/dev/null 2>&1 &
```

Otherwise:
```bash
setsid konsole --workdir "$target" -e claude >/dev/null 2>&1 &
```

`setsid` + background detaches so the new window survives the current session exiting.

### 3. Self-destruct (if requested)

If the caller requested self-destruct, follow the same pattern as `new-claude-here`:

1. Snapshot existing Konsole PIDs: `before=$(pgrep -x konsole)`.
2. Verify a new Konsole PID appeared after launch.
3. If verified, warn the user and close the current Konsole:
   ```bash
   (sleep 3 && kill -TERM $(ps -o ppid= -p $PPID | tr -d ' ')) &
   ```
4. If no new PID appeared, ABORT — do not close anything.

### 4. Report

Tell the user:
- The absolute path the new session was launched in
- Whether a seed prompt was passed
- If self-destruct is pending

## Notes

- The new session is standalone — no shared memory with this one.
- For full context transfer without Junction, run `session-transfer:write-handover` first.
- Do not block waiting on the Konsole process.
