---
name: clipboard-handover
description: Short YAML handover displayed in chat and copied to clipboard. No files, no spawn. Use when user says "clipboard handover", "copy handover", "handover to clipboard".
---

# Clipboard Handover

Generate a terse YAML handover, display in chat, copy to clipboard. Nothing written to disk.

## When to use

- User wants a portable summary to paste into Slack, a ticket, or another Claude session
- Quick context capture without file clutter

## Procedure

### 1. Gather context

Run in parallel:
- `git rev-parse --show-toplevel`
- `git branch --show-current`
- `git log --oneline -3`
- `git status --short`
- `git diff --stat`

### 2. Compose handover

Review conversation context and git state. Use this exact format, terse telegraphic English, no articles/filler:

```yaml
# Clipboard Handover
repo: name
branch: name
last_commit: hash msg
goal: one sentence
done: comma-separated items or ~
next: single most important step — be specific
watch_out: one warning/gotcha or ~
uncommitted: yes/no + brief desc
```

Keep under 12 lines.

### 3. Display and copy

1. Display the YAML in chat as a code block.
2. Copy to clipboard:
   ```bash
   echo "<the handover text>" | wl-copy
   ```
   Fall back to `xclip -selection clipboard` if `wl-copy` fails.
3. Confirm: "Copied to clipboard."

## Notes

- No files created, no session spawned
- For file-based handover use `/session-transfer:write-handover` or `/session-transfer:quick-handover`
- If user passes args (e.g. `/clipboard-handover focus on API changes`), incorporate into summary
