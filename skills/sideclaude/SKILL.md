---
name: sideclaude
description: Capture a side-task/idea that surfaced mid-thread, save it as a plan file in the repo, and spawn a new Konsole running a fresh Claude Code session at the current cwd with instructions to execute the plan. Use when the user says "sideclaude", "side claude", "spin this off", "delegate this to a new claude", "fork this off without polluting context", or otherwise wants to hand a tangent to a parallel session while keeping the main thread clean.
---

# SideClaude

Spin off a tangent into its own parallel Claude Code session without contaminating the current thread's context.

## When to use

The user is working on something with you and surfaces a *secondary* idea — a refactor, a related fix, a new feature, an experiment — that they want done but don't want to do *here* because it would muddle the current task. Instead of switching gears, capture the idea, hand it to a fresh Claude in a new terminal, and keep the current session focused.

Distinct from neighboring skills:
- `new-claude-here` — fresh session, no plan, no task brief.
- `handover` / `handover-with-tasks` — *transferring* the current session's state forward.
- `sideclaude` — *forking off* a side-task while the current session continues.

## Steps

### 1. Capture the plan

Confirm the side-task with the user in one sentence ("spinning off: <one-line summary>"). If the brief is thin, ask a single clarifying question — otherwise proceed.

Write the plan to `planning/sideclaude/<YYYY-MM-DD>-<slug>.md` at the repo root (create the directory if missing). Structure:

```markdown
# SideClaude Task: <title>

- **Spawned**: <ISO 8601 timestamp>
- **Parent session cwd**: <$PWD>
- **Parent session was working on**: <one-line summary of the main thread's current task>

## Goal
<What the user wants done. One paragraph.>

## Context the side-session needs
<Anything from the current thread that's load-bearing for the side-task — file paths, prior decisions, constraints. Do NOT dump the whole conversation; include only what's directly relevant.>

## Plan
1. <Concrete first step>
2. <...>

## Out of scope
<Things the side-session should NOT do, especially anything the parent session is actively working on.>

## When done
<How the side-session should report back — commit + push, open a PR, write a HANDOVER.md, etc. Default: commit, push, and leave a note at the bottom of this file.>
```

If the repo isn't a git repo or has no obvious root, use `$PWD/planning/sideclaude/` and tell the user.

### 2. Spawn the side session as a split-left/right pane

Preferred path: open the side session as a **Split Left/Right** pane inside the *current* Konsole window — both panes visible, no new window, parent session keeps running on the left.

Konsole exports `$KONSOLE_DBUS_SERVICE`, `$KONSOLE_DBUS_WINDOW`, and `$KONSOLE_DBUS_SESSION` into every session it spawns, so the current window/session is addressable via D-Bus without guessing. The relevant methods:

- `org.kde.konsole.Window.currentSession() → int` — id of the focused view
- `org.kde.konsole.Window.createSplit(int viewId, bool horizontalSplit) → bool` — `horizontalSplit=true` is **Split Left/Right** (panes laid out horizontally, vertical divider between them); `false` is Split Top/Bottom
- `org.kde.konsole.Window.sessionList() → QStringList` — to find the new session id after the split
- `org.kde.konsole.Session.runCommand(QString)` — start `claude` in the new pane

Recipe:

```bash
PLAN_PATH="planning/sideclaude/<file>.md"
PROMPT="Read $PLAN_PATH and execute the plan. This is a sideclaude session — the plan file is your full brief. Append a status note at the bottom of the file when done."

SVC="$KONSOLE_DBUS_SERVICE"
WIN="$KONSOLE_DBUS_WINDOW"

if [ -z "$SVC" ] || [ -z "$WIN" ]; then
  echo "Not running inside a Konsole session — falling back to new window." >&2
  # See fallback below.
else
  CUR_VIEW=$(qdbus6 "$SVC" "$WIN" currentSession)
  BEFORE_IDS=$(qdbus6 "$SVC" "$WIN" sessionList)
  qdbus6 "$SVC" "$WIN" createSplit "$CUR_VIEW" true   # true = Split Left/Right
  AFTER_IDS=$(qdbus6 "$SVC" "$WIN" sessionList)
  NEW_ID=$(comm -13 <(echo "$BEFORE_IDS" | tr ' ' '\n' | sort) \
                    <(echo "$AFTER_IDS"  | tr ' ' '\n' | sort) | tail -1)
  if [ -z "$NEW_ID" ]; then
    echo "createSplit succeeded but couldn't find new session id — aborting." >&2
    exit 1
  fi
  # cd into the parent's $PWD, then launch claude with the prompt.
  qdbus6 "$SVC" "/Sessions/$NEW_ID" runCommand "cd $(printf '%q' "$PWD") && claude $(printf '%q' "$PROMPT")"
fi
```

Notes:
- `qdbus6` is the Plasma 6 binary; on older systems it's `qdbus`. Try `qdbus6` first, fall back to `qdbus`.
- `horizontalSplit=true` matches what the Konsole UI labels **Split View → Split Left/Right** (panes side by side). `false` is top/bottom.
- `runCommand` types into the new shell — quoting matters. Use `printf %q` to escape `$PWD` and the prompt safely.

### 3. Fallback: new window

If we're not inside a Konsole (`$KONSOLE_DBUS_SERVICE` empty — e.g. running under tmux, ssh, or a different terminal), spawn a detached window instead:

```bash
before=$(pgrep -x konsole)
setsid konsole --workdir "$PWD" -e claude "$PROMPT" >/dev/null 2>&1 &
after=$(pgrep -x konsole)
comm -13 <(echo "$before" | sort) <(echo "$after" | sort)
```

If no new PID appears, abort and report — do not retry blindly.

### 4. Stay put

Unlike `new-claude-here`, **do not close the current Konsole / current pane**. The whole point is that the parent session keeps going. Report back to the user:

- Plan file path
- Whether the side session was opened as a split pane or a new window (and the new session/PID)
- One-line reminder that the current session is unchanged and ready to continue the original task

## Notes

- The new session has no memory of this conversation — the plan file is its only brief. Write it accordingly: standalone, specific, with file paths the side-session can `cat` directly.
- If the user wants the side-task to land on a separate branch, include that in the **Plan** section ("create branch `sideclaude/<slug>` first"). Don't switch branches in the parent session.
- If `claude` isn't on PATH inside the spawned Konsole's login shell, fall back to spawning with `bash -lc 'claude "<prompt>"'` instead of `-e claude`.
