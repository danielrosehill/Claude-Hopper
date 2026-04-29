---
name: start-from-handover
description: Open the current Claude session against a previously-written handover file and continue from the Outstanding section. Use at the start of a fresh session when a prior session was checkpointed via context-gate, handover, or handover-with-tasks. Triggers on "start from handover", "load handover", "resume from handover", "pick up where the last session left off", "continue from the plan".
---

# Start From Handover

The receiving end of the context-gate workflow. The previous session wrote a handover plan; this skill loads it, validates the format, summarises what's there, and shifts the new session into "continue from Outstanding" mode.

This skill matches the canonical handover format defined in `references/handover-template.md` (schema `claude-rudder/handover/v1`).

## When to use

- The user just opened a new Claude session and references a handover file
- The user says "start from handover", "load handover", "resume from <path>", "pick up the plan"
- A previous session ended with `context-gate` and the user is now in the spawned session

## Inputs

- `--path <file>` (optional) — path to the handover file. If omitted, discover.

## Discovery

If no path is given, look for handover files in this order:

1. `<repo-root>/planning/handovers/handover-*.md` (sorted newest-first by mtime)
2. `<repo-root>/.claude/handovers/handover-*.md` (legacy location, accept but warn)
3. `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/claude-rudder/handovers/handover-*.md`

Where `<repo-root>` is `git rev-parse --show-toplevel` if cwd is in a git repo, else cwd.

If multiple handovers are found, list the top 5 (newest first) with their `written_at` timestamp and reason from the frontmatter:

```
1. handover-2026-04-27-1842.md  (15min ago) — "context cluttered after migration debug"
2. handover-2026-04-25-0930.md  (2 days ago) — "end of work session"
...
```

Ask the user to pick. If only one exists, default to it after confirming.

## Steps

### 1. Read and validate

Read the picked handover file. Verify:

- Frontmatter is YAML and includes `schema: claude-rudder/handover/v1` (warn if absent or mismatched, but proceed)
- Required sections exist: `Why this checkpoint`, `Original ask`, `Done`, `Outstanding`, `Pointers`
- The `repo` in frontmatter matches the current repo (warn if mismatch — the user might be intentionally resuming in a different repo, but flag it)

### 2. Show a session-orientation summary

Print this to the user before doing any work:

```
=== Resuming from handover ===
File: <absolute path>
Written: <written_at> by <agent_model> (<harness>)
Reason: <Why this checkpoint section, first sentence>

Original ask:
  <Original ask section, condensed to ~2 lines>

Done in prior session (<n> items):
  - <first 3-5 bullets>
  ...

Outstanding (<n> items):
  - [ ] <each open checkbox bullet>

Open questions for you:
  - <each question, if any>

Gotchas to remember:
  - <each, if any>
=============================
```

If `Open questions` is non-empty, **do not proceed** — answer them with the user first. The prior session deliberately deferred these.

### 3. Confirm and shift mode

Ask the user:

> Pick up from the Outstanding list? (yes / pick specific items / start fresh)

- **yes** → treat the Outstanding list as the new TODO. Work through items in the listed order unless the user redirects.
- **pick specific items** → user names which Outstanding bullets to tackle. The rest stay open.
- **start fresh** → user wants context but a different focus. Ack the handover, drop the resume-mode, await new instructions.

### 4. Execute

Once the user confirms, work through the Outstanding items. As each completes:

- Optionally update the handover file: change `- [ ]` to `- [x]` in the Outstanding section, append a one-line note to the bottom of the `Done` section.
- Don't rewrite the handover wholesale — append/checkbox-toggle only. The handover is a frozen snapshot of the prior session; this session's work belongs in its own future handover.

### 5. New decisions or scope changes

If during execution the user makes a new decision that contradicts something in `Decisions made`, surface it explicitly:

> The handover noted: "<decision>". You're now choosing differently. Confirm I should proceed with the new approach?

Then update only this session's working notes — don't retroactively edit the handover.

## Hard rules

- **Treat the handover as authoritative for prior context.** Don't second-guess the `Done` list or re-do completed work.
- **Surface Open questions before doing anything.** That's the point of the field.
- **Don't write to the handover file beyond checkbox toggles and Done-section appends.** The frozen snapshot rule preserves audit trail.
- **If validation fails** (no schema, no Outstanding section, malformed frontmatter): show the file as-is and ask the user how to proceed. Don't try to "fix" the handover automatically.

## Counterparts in claude-rudder

- `context-gate` — orchestrates the handover-write side (mirror of this skill)
- `handover` — end-of-session menu, can also write the file consumed here
- `handover-with-tasks` — task-list-heavy variant, also produces compatible files
- `references/handover-template.md` — canonical format spec
