# Handover Template (canonical)

The reference format every handover-writing skill in `claude-rudder` produces, and every handover-reading skill expects. **Single source of truth.** When you change this file, all consumers stay aligned automatically.

Skills that produce handovers in this format:
- `handover` (end-of-session menu)
- `handover-with-tasks` (task-list-heavy variant)
- `clipboard-handover` (compact YAML for clipboard)
- `context-gate` (orchestrator)

Skills that consume handovers in this format:
- `start-from-handover` (the receiving side — loads handover, resumes work)

## Structure

```markdown
---
schema: claude-rudder/handover/v1
agent_model: <e.g. claude-opus-4-7, claude-sonnet-4-6>
agent_harness: claude-code
written_at: <ISO-8601 with timezone, e.g. 2026-04-27T18:42:00+03:00>
session_started_at: <ISO-8601 — when the session being handed over began, if known>
repo: <absolute path or null>
branch: <branch name or null>
last_commit: <short sha or null>
---

# Handover — <YYYY-MM-DD HH:MM>

## Why this checkpoint
<one or two sentences — what triggered the handover>

## Original ask
<the user's request that started the session, verbatim or paraphrased>

## Done
- <bullets — completed work, with file paths or commit hashes where useful>
- <each bullet should be independently verifiable>

## Outstanding
- [ ] <bullet — remaining work item, priority order>
- [ ] <use markdown checkboxes so `start-from-handover` can parse a task list>

## Decisions made
- <key choices made during the session, with brief rationale>
- <call out any options that were considered and rejected>

## Open questions
- <things the new session should ask the user about before proceeding>
- <leave empty if there are none>

## Gotchas / pitfalls
- <surprises, dead ends, things the new session shouldn't waste time rediscovering>
- <anything fragile that requires care>

## Pointers
- Files touched this session: <list>
- Relevant docs/CLAUDE.md sections: <list>
- External resources consulted: <list>

## Resume command (suggested)
```
claude --message "Read this handover and continue from the Outstanding section: <absolute path to this file>"
```
```

## Field rules

### `agent_model`
The model that wrote the handover, in API-identifier form. Examples:

- `claude-opus-4-7`
- `claude-opus-4-6`
- `claude-sonnet-4-6`
- `claude-haiku-4-5-20251001`

The writing skill should read this from the active session's model identifier (typically available via `$CLAUDE_MODEL` env var or Claude's self-knowledge). If unknown, set to `unknown`.

### `agent_harness`
Almost always `claude-code` for these workflows. Other valid values: `claude.ai`, `claude-api`, `agent-sdk`. Distinguishes the runtime so the receiving skill can adjust expectations (e.g., a claude-code handover assumes filesystem access; a claude.ai one might not).

### `written_at`
ISO-8601 with timezone offset. Use the system timezone, not UTC unless the user prefers it. Minute resolution.

### `session_started_at`
Optional but recommended. Helps the new session understand how long the prior context ran.

### `Outstanding` checkboxes
Use proper markdown checkboxes (`- [ ]` for unchecked, `- [x]` for checked) so they can be parsed mechanically. Other bullet styles in this section are allowed but checkbox bullets are how `start-from-handover` extracts the task list.

### `Pointers`
Absolute paths preferred. Relative paths must be relative to `repo` (in the frontmatter).

## What NOT to include

- **Raw credential values.** API keys, tokens, passwords — never. Reference by env var name or `op://` path if the new session needs them.
- **Speculative next steps.** The handover documents what *was decided* and what *remains*. Don't predict what Claude should do; let the new session re-evaluate.
- **Full conversation transcripts.** Summaries only. The new session has the handover for context, not a replay.
- **Tool-call outputs that were noisy or irrelevant.** Trim to what's actually load-bearing.

## Versioning

The `schema` field in frontmatter is `claude-rudder/handover/v1`. If this template changes in a backwards-incompatible way, bump to `v2` and update both the writers and readers in lockstep.
