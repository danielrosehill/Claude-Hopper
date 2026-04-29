---
name: handover
description: End-of-session menu — wrap up this Claude session. Logging, YAML handovers, spawning new instances, clipboard. Triggers on "handover", "end session", "wrap up", "hand off", "session transfer".
---

# Handover Menu

Single entry point for all end-of-session workflows.

## When to use

- User says "handover", "hand off", "wrap up", "end session", "session transfer", "pass the baton"
- Context getting long, switching task/repo

## Procedure

### 1. Present menu

Display exactly:

```
How do you want to wrap up?

1. Full handover + new session — investigate repo, write YAML HANDOVER.md, spawn new Claude
2. Quick handover + new session — minimal YAML HANDOVER.md, spawn new Claude
3. Log + full handover + new session — archive to Agent-Logs, then full handover + spawn
4. Write handover only — create YAML HANDOVER.md, stay in this session
5. Log only — record to Agent-Logs, no handover, no spawn
6. Log + spawn (no handover) — archive log, spawn fresh Claude with no context
7. Clipboard handover — short YAML summary in chat + copied to clipboard, no files, no spawn
```

### 2. Wait for choice

Accept 1-7 or keyword ("quick", "log only", "full", "clipboard").

### 3. Execute

#### Choice 1: Full handover + new session
Run `/session-transfer:handover`:
1. **handover-writer** subagent creates YAML `HANDOVER.md`
2. Verify exists
3. `konsole --workdir "$(git rev-parse --show-toplevel)" -e claude "/session-transfer:start-from-handover" &`
4. Report

#### Choice 2: Quick handover + new session
Run `/session-transfer:quick-handover`:
1. Capture git state
2. Write minimal YAML HANDOVER.md
3. Commit and push
4. `konsole --workdir "$(git rev-parse --show-toplevel)" -e claude "/session-transfer:start-from-handover" &`
5. Report

#### Choice 3: Log + full handover + new session
Run `/session-transfer:log-and-handover`:
1. Write JSON log to Agent-Logs (check memory for path, fall back `~/repos/github/my-repos/Agent-Logs/`)
2. Commit, push, run `python3 scripts/ingest.py` if exists
3. **handover-writer** subagent creates YAML `HANDOVER.md`
4. Spawn + report

#### Choice 4: Write handover only
Run `/session-transfer:write-handover`:
1. **handover-writer** subagent creates YAML `HANDOVER.md`
2. Confirm — do NOT spawn
3. Remind: can run `/session-transfer:start-from-handover` later

#### Choice 5: Log only
1. Find Agent-Logs path (memory `reference_agent_logs.md`, fall back `~/repos/github/my-repos/Agent-Logs/`)
2. If missing, suggest `/session-transfer:setup-agent-logs`. Stop.
3. Write JSON log:
   - Folder: `DDMMYY/`, File: `HHMMSS_model-name.json`
   - Schema:
   ```json
   {"created":"ISO8601","model":"ID","agent_type":"Claude Code session","working_directory":"/path","repository":"name","branch":"name","last_commit":"hash msg","blocked":false,"user_input_needed":false,"debugging_required":false,"pending_tasks":0,"goal":"summary","completed":[],"in_progress":[],"not_started":[],"blockers":[],"resumption_summary":"next steps"}
   ```
4. Commit, push, run `python3 scripts/ingest.py` if exists
5. Report: "Session logged. No handover, no spawn."

#### Choice 6: Log + spawn (no handover)
1. Write log (same as Choice 5)
2. `konsole --workdir "$(git rev-parse --show-toplevel)" -e claude &`
3. Report: "Logged + spawned. No handover — fresh session."

#### Choice 7: Clipboard handover
Run `/session-transfer:clipboard-handover`:
1. Gather git state
2. Compose short YAML summary from conversation + git state
3. Display in chat as code block
4. Copy via `wl-copy` (fall back `xclip`)
5. Confirm: "Copied to clipboard."
6. No files, no spawn

## Notes

- `/handover quick` or `/handover 2` skips menu, goes direct
- Individual commands still work as shortcuts
- Missing Agent-Logs: suggest `/session-transfer:setup-agent-logs`, don't fail silently
