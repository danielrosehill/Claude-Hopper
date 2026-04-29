---
name: new-workspace
description: Provision a minimal meta-tools workspace. The meta-tools plugin ships primitives (commands + skills) that operate on Claude Code itself — this provisioning skill only creates a thin scaffold for per-workspace notes. Accepts a workspace name and optional variant (context-management | mcp-integration | feedback-filing | primitive-selection | generic).
disable-model-invocation: true
allowed-tools: Bash(mkdir *), Bash(cp *), Bash(cat *), Bash(git init *), Bash(git add *), Bash(git commit *), Bash(gh repo create *), Bash(gh auth status), Bash(git push *), Read
---

# Provision Meta-Tools Workspace

The `meta-tools` plugin is **primitive-only** — its value is in the global commands and skills it installs (context-break evaluator, CLAUDE.md chunker, MCP command generator, what-thing picker, claude-code feedback filer). This skill provisions an optional thin scaffold for users who want a repo to collect their own notes, logs, and local config related to meta-tooling.

## Arguments

`$ARGUMENTS` is parsed as:

- **First positional**: workspace name (kebab-case). Required.
- **Second positional** (optional): target parent path. Defaults to `~/repos/github/my-repos`.
- **`--variant=<name>`** (optional): `context-management` | `mcp-integration` | `feedback-filing` | `primitive-selection` | `generic` (default).
- **`--local-only`** (optional): skip GitHub repo creation.
- **`--private`** (optional): create the GitHub repo as private. Default: public.

## Procedure

### 1. Parse arguments

Extract workspace name, target parent path, variant, flags. If workspace name missing, ask.

### 2. Resolve the scaffold path

The bundled scaffold lives at `${CLAUDE_SKILL_DIR}/../../template/generic/`. There is a single shared `generic/` variant — the `--variant` flag only flavors the generated CLAUDE.md header and hints, not the on-disk layout.

### 3. Read ambient facts

Read `~/.claude/CLAUDE.md` if it exists. Extract OS, locale, timezone, user identity for CLAUDE.md personalisation.

### 4. Create the workspace directory

```bash
mkdir -p <target-parent>/<workspace-name>
cp -r ${CLAUDE_SKILL_DIR}/../../template/generic/. <target-parent>/<workspace-name>/
```

Do **not** copy any `.claude/` tree. Plugin primitives are global.

### 5. Personalise CLAUDE.md

Edit the workspace's `CLAUDE.md`:

- Insert workspace name and chosen variant at the top.
- If variant is `context-management`, list the context-management primitives (`/meta-tools:chunk-claude`, `/meta-tools:should-i-start-fresh`, `/meta-tools:context-health-check`) and the `context-break-evaluator` skill.
- If variant is `mcp-integration`, list the MCP primitives (`/meta-tools:add-mcp-server`, `/meta-tools:list-mcp-servers`, `/meta-tools:mcp-command-help`) and the `mcp-syntax-reference` skill.
- If variant is `feedback-filing`, list the `feedback-bug`, `feedback-feature-request`, `feedback-model-behavior`, `feedback-docs`, `feedback-refresh-templates` skills.
- If variant is `primitive-selection`, list the `what-thing-pick`, `what-thing-help`, `what-thing-set-context`, `what-thing-refresh-docs` skills.
- If variant is `generic`, list all of the above briefly.
- Embed ambient OS/locale facts from step 3 so downstream commands skip re-asking.

### 6. Initialise git and (optionally) publish

```bash
cd <target-parent>/<workspace-name>
git init
git add .
git commit -m "Initial workspace from meta-tools plugin"
```

Unless `--local-only`:

```bash
gh repo create <workspace-name> --<public|private> --source=. --push
```

### 7. Print next steps

Tell the user:

- Workspace path.
- Chosen variant and which plugin primitives are most relevant.
- Reminder: the workspace is **data** — plugin primitives remain globally available regardless of cwd.

## Notes

- Resolve the scaffold via `${CLAUDE_SKILL_DIR}/../../template/` (not `${CLAUDE_PLUGIN_ROOT}` — that isn't exported in skill bash injection).
- Never copy `.claude/commands/`, `.claude/agents/`, or `.claude/skills/` into the workspace.
- No personal paths or identifiers in the scaffold — everything comes from user memory or prompts.
