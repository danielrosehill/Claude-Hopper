---
name: claude-grid
description: Launch a single Konsole window at a given path containing 2 or 4 split panes, each running a fresh Claude Code session in that path. Use when the user asks for a "claude grid", "2-pane claude", "4-pane claude", "split konsole with claudes", "quad claude", or similar — i.e. they want multiple parallel Claude sessions in the same path, visible side-by-side. Default count is 2; pass `4` (or "quad") for the 2×2 grid.
---

# Claude Grid

Spawn one Konsole window at a given path with N parallel Claude sessions tiled inside it.

- **2 panes**: Split Left/Right (vertical divider).
- **4 panes**: 2×2 grid (Split Left/Right, then Split Top/Bottom on each side).

All panes start in the same `cwd`. Each runs an independent `claude` — no shared memory between them.

## When to use

The user wants several parallel Claude sessions on the same project visible at once — typically for fan-out work (try N approaches in parallel, batch up independent tasks, or just have a "main + scratch" layout). Distinct from:

- `sideclaude` — one tangent into a single side pane, with a plan file as brief.
- `new-claude-here` / `new-claude-at` — single fresh session in a *new window*.

## Inputs

- `path` — directory to launch in. Default `$PWD`. Resolve `~` and relative paths to absolute.
- `count` — `2` or `4`. Default `2`. Reject other values with a one-line note.

## Steps

### 1. Spawn the window with pane 1

Spawn a fresh Konsole running `claude` at `path`. Force a separate process so the new window's D-Bus service is independent and findable:

```bash
TARGET="$1"      # absolute path
COUNT="${2:-2}"  # 2 or 4

before_svcs=$(qdbus6 2>/dev/null | grep -E '^ org\.kde\.konsole-' | sort)
setsid konsole --separate --workdir "$TARGET" -e claude >/dev/null 2>&1 &
```

Poll briefly (up to ~3s) for a new D-Bus service to appear:

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

### 2. Add the splits

`createSplit(viewId, horizontalSplit)`:
- `horizontalSplit=true` → Split **Left/Right** (panes laid out horizontally, vertical divider).
- `horizontalSplit=false` → Split **Top/Bottom**.

Helper to find the newest session id after a split:

```bash
new_session_after() {
  local svc="$1" win="$2" before="$3"
  local after
  after=$(qdbus6 "$svc" "$win" sessionList | tr ' ' '\n' | sort -n)
  comm -13 <(echo "$before") <(echo "$after") | tail -1
}

run_claude_in() {
  local svc="$1" sid="$2"
  qdbus6 "$svc" "/Sessions/$sid" runCommand "claude"
}
```

#### count = 2

```bash
A=$(qdbus6 "$NEW_SVC" "$WIN" currentSession)
before=$(qdbus6 "$NEW_SVC" "$WIN" sessionList | tr ' ' '\n' | sort -n)
qdbus6 "$NEW_SVC" "$WIN" createSplit "$A" true   # Split Left/Right
B=$(new_session_after "$NEW_SVC" "$WIN" "$before")
run_claude_in "$NEW_SVC" "$B"
```

#### count = 4

```bash
A=$(qdbus6 "$NEW_SVC" "$WIN" currentSession)

before=$(qdbus6 "$NEW_SVC" "$WIN" sessionList | tr ' ' '\n' | sort -n)
qdbus6 "$NEW_SVC" "$WIN" createSplit "$A" true    # Split L/R → A | B
B=$(new_session_after "$NEW_SVC" "$WIN" "$before")

before=$(qdbus6 "$NEW_SVC" "$WIN" sessionList | tr ' ' '\n' | sort -n)
qdbus6 "$NEW_SVC" "$WIN" createSplit "$A" false   # Split T/B on A → A / C
C=$(new_session_after "$NEW_SVC" "$WIN" "$before")

before=$(qdbus6 "$NEW_SVC" "$WIN" sessionList | tr ' ' '\n' | sort -n)
qdbus6 "$NEW_SVC" "$WIN" createSplit "$B" false   # Split T/B on B → B / D
D=$(new_session_after "$NEW_SVC" "$WIN" "$before")

run_claude_in "$NEW_SVC" "$B"
run_claude_in "$NEW_SVC" "$C"
run_claude_in "$NEW_SVC" "$D"
```

### 3. Report back

Tell the user:
- Path the grid was launched in.
- Number of panes (2 or 4).
- D-Bus service of the new window (so a follow-up skill can target it).

## Notes

- New panes from `createSplit` inherit the parent pane's `cwd`, so all sessions land in `$TARGET` without an explicit `cd`.
- `qdbus6` on Plasma 6; fall back to `qdbus` on older systems.
- The first pane was launched with `-e claude` so it's already running. Only the *added* panes need `runCommand "claude"`.
- Konsole's `--tabs-from-file` can launch multiple tabs but does **not** support splits — that's why this skill uses the D-Bus path.
- If the user later wants to broadcast the same input to all panes, that's a Konsole built-in (`Edit → Copy Input To → All Tabs / All Panes in Window`); not scripted here.
- For counts other than 2 or 4 (e.g. 3, 6), reject with a note rather than guessing a layout — the geometry is ambiguous and worth a one-line clarifying question.
