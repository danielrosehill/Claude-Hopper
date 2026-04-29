---
name: handover-with-tasks
description: Write a handover document that includes an open task list for the next agent. Use when switching sessions mid-way through a multi-task workload, not just a single in-progress job. Triggers on phrases like "handover with tasks", "task handover", "hand off the task list", "context cleanup with tasks", "pass on the work queue".
---

# Handover With Task List

A 2-in-1 handover skill: writes a structured `HANDOVER.md` that captures both the current state AND an actionable task list for the next agent. Use this instead of the regular handover when the session has an open queue of work items rather than a single focused task.

## When to use this vs regular handover

- **Regular handover** (`session-transfer:handover`): One task in progress, next agent picks up where you left off
- **This skill**: Multiple tasks, some done, some in progress, some not started. The next agent needs a prioritized work queue, not just resumption instructions

## Investigation Process

Before writing, gather ALL of the following:

1. **Git status** — staged/unstaged changes, current branch, recent commits (`git log --oneline -20`)
2. **Git diff** — what's changed since last commit
3. **Read CLAUDE.md** if present — understand project conventions
4. **Scan for existing plans/tasks** — TODO files, task lists, issues, any planning docs
5. **Review conversation context** — what tasks were discussed, requested, completed, deferred
6. **Identify the active work area** — which files were most recently modified

If `$ARGUMENTS` contains additional context about the task list, incorporate it.

## Output

Write `HANDOVER.md` at the repository root with the following structure:

```markdown
# Agent Handover Document

## Metadata
- **Created**: [ISO 8601 timestamp]
- **Creating agent**: [Agent type and model]
- **Repository**: [repo name]
- **Branch**: [current branch]
- **Last commit**: [short hash + message]
- **Handover type**: Task list transition

## Context
[Brief summary of what this session was doing and why the handover is happening. Include any relevant background the next agent needs to understand the overall objective.]

## Task List

### Completed
- [x] [Task description] — [brief note on outcome/location of work]

### In Progress
- [ ] [Task description] — [current state, what's been done, what remains]
  - **Files touched**: [list]
  - **Current state**: [where exactly this was left]
  - **Next action**: [specific next step]

### Not Started
- [ ] [Task description] — [any context or prerequisites]

### Deferred / Out of Scope
- [ ] [Task description] — [why deferred, any notes for future]

## Current Repository State
- **Build status**: [passing/failing/unknown]
- **Uncommitted changes**: [yes/no — summarize if yes]
- **Working tree clean**: [yes/no]

## Key Decisions Made
- [Decision]: [Rationale]

## Failed Approaches (Do Not Repeat)
- [What was tried]: [Why it failed]

## Files of Interest
| File | Why |
|------|-----|
| [path] | [reason this file matters] |

## Resumption Instructions
[Tell the next agent to start by reviewing the Task List above and begin with the highest-priority incomplete item. Include any ordering preferences, dependencies between tasks, or constraints.]

## Context the Next Agent Needs
[Environment variables, running services, non-obvious codebase quirks, etc.]
```

## After Writing

1. Confirm HANDOVER.md was written successfully
2. Give the user a summary: N completed, N in progress, N remaining
3. Ask if they want to spawn a new Claude session to pick up from the handover (using `session-transfer:new-claude-here` or `session-transfer:handover` spawn pattern)
