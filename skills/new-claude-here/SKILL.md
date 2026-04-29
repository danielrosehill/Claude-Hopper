---
name: new-claude-here
description: Open a new Konsole window at the current working directory running a fresh Claude Code CLI session, then close the current Konsole. Use when the user asks for a "new claude session here", "claude here", "jump to a new claude", or "new claude window". Pass "--keep" / "don't close this one" to skip the self-destruct.
---

# New Claude Session Here

Spawns a sibling Claude Code session in a new Konsole window at the current `$PWD`, then closes the current Konsole so the user lands cleanly in the fresh session.

## Why this lives in the session-transfer plugin

The common reason to jump to a fresh Claude session mid-task is that the current session can't pick up changes without a restart — most often after installing a new MCP server, editing settings, or hitting a stuck state. In those cases the user wants session continuity without losing their place. This skill is the "quick jump" path; `session-transfer:write-handover` / `session-transfer:start-from-handover` is the "pipe the full context across" path. Offer both when relevant.

## Steps (default: self-destruct on)

1. Snapshot existing Konsole PIDs: `before=$(pgrep -x konsole)`.
2. Spawn the new Konsole detached:
   ```bash
   setsid konsole --workdir "$PWD" -e claude >/dev/null 2>&1 &
   ```
   `setsid` + background keeps it running after the current Claude Code session exits.
3. Verify a new Konsole PID appeared (poll briefly if needed):
   ```bash
   after=$(pgrep -x konsole)
   comm -13 <(echo "$before" | sort) <(echo "$after" | sort)
   ```
   If no new PID, ABORT — do not close anything. Report the failure.
4. Warn the user once: "New Konsole verified at PID <n>. Closing this terminal in 3s — the current Claude session will die with it."
5. Close the current Konsole by killing its window process:
   ```bash
   (sleep 3 && kill -TERM $(ps -o ppid= -p $PPID | tr -d ' ')) &
   ```
   Rationale: `$PPID` inside a Claude tool call is the shell; its parent is the Konsole window. Killing the window tears down the shell and this Claude session cleanly.

## Keep-current variant

If the user says "keep this one open", "don't close this terminal", or passes `--keep`, do steps 1–3 only and skip the kill. Report the new session directory.

## Notes

- The new session is standalone — no shared memory with this one. If the user needs context carried over, suggest running `session-transfer:write-handover` first.
- Report the directory the new session was launched in before self-destructing.
