---
name: claude-and-shell
description: Launch a single Konsole window at a target directory split left/right — left pane runs Claude Code, right pane is a raw shell at the same path (no Claude). Use when the user asks for "claude with a terminal", "claude + shell", "claude and a free konsole", "paired konsole", or wants Claude visible alongside a hands-on shell for the same project.
---

# Claude + Shell

Spawn one Konsole at a path with two side-by-side panes:
- **Left**: fresh `claude` session at the path.
- **Right**: raw login shell at the same path. No Claude. The user types here.

Useful when the user wants to watch Claude work *and* run their own commands (git, tests, scratch experiments) without taking over a Claude pane or opening a separate window.

## When to use

- "Open claude with a terminal next to it"
- "I want a shell beside this claude session"
- "Paired konsole at <path>"

Distinct from neighbors:
- `claude-grid` — multiple **Claude** sessions tiled together.
- `sideclaude` — fork a tangent into a side Claude pane with a plan brief.
- `terminal-here` — single raw Konsole, no Claude.

## Inputs

- `path` — target directory. Default `$PWD`. Resolve `~` and relative paths to absolute.

## Steps

### 1. Spawn the window with the Claude pane

```bash
TARGET="$1"  # absolute path

before_svcs=$(qdbus6 2>/dev/null | grep -E '^ org\.kde\.konsole-' | sort)
setsid konsole --separate --workdir "$TARGET" -e claude >/dev/null 2>&1 &
```

Poll for the new window's D-Bus service:

```bash
for i in $(seq 1 30); do
  sleep 0.1
  after_svcs=$(qdbus6 2>/dev/null | grep -E '^ org\.kde\.konsole-' | sort)
  NEW_SVC=$(comm -13 <(echo "$before_svcs") <(echo "$after_svcs") | tr -d ' ' | head -1)
  [ -n "$NEW_SVC" ] && break
done
[ -z "$NEW_SVC" ] && { echo "Could not find new Konsole D-Bus service — abort." >&2; exit 1; }

WIN="/Windows/1"
```

### 2. Add the raw-shell pane

`createSplit(viewId, true)` = Split Left/Right. The new session starts the user's default shell at the parent pane's cwd — no command needed.

```bash
A=$(qdbus6 "$NEW_SVC" "$WIN" currentSession)   # the claude pane
qdbus6 "$NEW_SVC" "$WIN" createSplit "$A" true
```

That's it. Do **not** `runCommand` anything on the new pane — leaving it untouched is the whole point.

### 3. Report back

- Path the window was launched in.
- D-Bus service of the new window (handy for follow-ups like `claude-grid`-style additions later).
- One-line note: left = Claude, right = shell.

## Notes

- `qdbus6` on Plasma 6; fall back to `qdbus` on older systems.
- New panes inherit cwd from the parent pane, so the right shell lands in `$TARGET` automatically.
- If the user later wants to swap which side is which, the layout is symmetric — they can drag panes in the Konsole UI, or we can re-run with the order reversed (spawn raw first, then split with `runCommand "claude"` on the new pane).
- For "claude on top, shell below" instead of left/right, change the boolean to `false` (Split Top/Bottom). Mention this if the user expresses a preference.
