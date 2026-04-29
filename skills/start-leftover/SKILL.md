---
name: start-leftover
description: Pick one or more open items from `planning/leftover-tasks.md` and spawn a fresh Claude session focused on them. Counterpart to `capture-leftover`. Use when the user says "tackle a leftover", "pick up a parked task", "start on a leftover", "what's in the parking lot", or wants to clear deferred follow-ups without polluting the current session.
---

# Start Leftover

Picks open entries from `planning/leftover-tasks.md`, lets the user choose one or a batch to work on, and spawns a clean Claude session focused on just those items. The original session keeps its context untouched.

## When to use

- User wants to address a previously-deferred follow-up
- "What's parked?" / "let's clear some leftovers" / "tackle that auth refactor we noted"
- After `context-gate` writes a handover, user wants to tackle one parked item rather than resume the main thread

## Steps

### 1. Locate and read

```bash
repo_root=$(git rev-parse --show-toplevel 2>/dev/null) || repo_root="$PWD"
target="$repo_root/planning/leftover-tasks.md"
```

If `target` doesn't exist, tell the user there are no leftovers to start from and stop. Don't auto-scaffold here — there's nothing to scaffold for.

### 2. Parse open entries

Each entry begins with `## YYYY-MM-DD HH:MM — <summary>`. Read the file and extract entries where `Status: open`. Skip `in-progress` and `done`.

If zero open entries, print:

```
No open leftovers in planning/leftover-tasks.md. ✓
```

### 3. Present the list

Show open entries newest-first by capture timestamp:

```
Open leftovers (4):

1. 2026-04-27 18:53 — Refactor auth middleware to drop legacy session-token path
   Scope: medium  |  Captured 2h ago

2. 2026-04-26 14:10 — Investigate flaky CI run on linter step
   Scope: small  |  Captured 1d ago

3. 2026-04-25 09:30 — Add OG images to blog posts
   Scope: medium  |  Captured 2d ago

4. 2026-04-23 16:45 — Migrate utils.py to typed module structure
   Scope: large  |  Captured 4d ago
```

### 4. User picks

Ask:

> Pick one or more (e.g. `1`, `1,3`, or `all`). Or `cancel`.

- Single number → one task focus
- Comma-list → batch mode
- `all` → batch all opens
- `cancel` → exit cleanly

### 5. Mark in-progress

For each picked entry, change `**Status:** open` → `**Status:** in-progress` in the file. Append a marker line:

```
- **Started:** 2026-04-27 21:05 (session: <new-session-id-or-cwd-marker>)
```

This way the original session knows what's been claimed.

### 6. Build a focused-session prompt

Construct an initial message for the new session that includes:

- Which leftover(s) to tackle (full entry text from the file)
- The original repo context (path, branch)
- A one-line directive: "Tackle the items below. When complete, mark each as `Status: done` in `planning/leftover-tasks.md` and append a one-line outcome note."

Example payload:

```
Continue from these parked items in this repo:

[paste the full markdown entry blocks here]

Repo: /home/user/repos/myproject
Branch: main

When you finish each, mark `Status: done` in planning/leftover-tasks.md and append a one-line outcome below the entry. If any item turns out larger than the estimated scope, ask before continuing.
```

### 7. Spawn

Delegate to `new-claude-here` (same plugin) with the message pre-loaded. Default form:

```bash
# in a new Konsole window at $repo_root, opens claude with -m "<the payload>"
```

If `new-claude-here` isn't available or the user prefers manual launch, print the command + payload and let the user run it.

### 8. Return

Print one summary line in the original session:

```
✓ Started: "Refactor auth middleware..." in new session (1 picked, 3 still open)
```

## Batch behaviour

If the user picked multiple items, the new session gets all picked entries in one prompt. Tell it to handle them sequentially. The original session's `leftover-tasks.md` reflects all picked items as `in-progress`.

## Cleanup after completion

The new session is responsible for marking `Status: done` and appending an outcome line per task. If the user reports "I finished those" in the original session and the file still shows in-progress, offer to update it from this side — but the new session should normally do it.

## Edge cases

- **Same item picked twice across sessions** — second pick should detect `Status: in-progress`, warn the user, and let them either reclaim it (force) or pick something else.
- **Capture-while-in-flight** — user might capture new leftovers while the picked-up ones are running. That's fine — file is append-only at the entry level, and `Status` updates are line-level.
- **No `new-claude-here` available** — print the manual launch command instead. Don't fail the workflow.

## Hard rules

- **Don't reorder entries.** Newest-first is presentation only; the file stays in capture order.
- **Don't auto-pick.** Always require user confirmation, even if there's only one open item.
- **Don't mark `done` from this skill.** That's the new session's job — it ensures whoever did the work confirms it landed.
- **Don't spawn a session without the explicit prompt payload.** A blank new-claude-here loses the focus.

## Counterparts

- `capture-leftover` — adds entries that this skill consumes
- `scaffold-planning` — creates the file
- `new-claude-here` — actual session-spawning mechanic this delegates to
