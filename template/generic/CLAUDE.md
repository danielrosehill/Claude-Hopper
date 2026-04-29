# Meta-Tools Workspace

This workspace pairs with the `meta-tools` Claude Code plugin. The plugin ships primitives (commands + skills) that operate on Claude Code itself — context-break evaluation, CLAUDE.md chunking, MCP command generation, primitive selection (what-thing), and claude-code issue filing.

This workspace is **data only**. The primitives live in the plugin and are available globally regardless of cwd.

## Variant

`<variant>` — one of:

- `context-management` — CLAUDE.md chunking, context-break evaluation, context-health checks.
- `mcp-integration` — MCP server installation helpers and syntax reference.
- `feedback-filing` — file well-formed issues against `anthropics/claude-code`.
- `primitive-selection` — choose which Claude Code primitive fits a workflow (what-thing).
- `generic` — all of the above.

## Suggested layout

- `notes/` — free-form notes about meta-tooling decisions.
- `context/` — scratch context the primitives can read.
- `logs/` — session logs, feedback drafts, MCP command history.

## Relevant plugin primitives

**Commands** (`/meta-tools:<name>`):

- `chunk-claude` — prune bloated CLAUDE.md into `agent-context/` chunks.
- `should-i-start-fresh` — evaluate whether to begin a new conversation.
- `context-health-check` — audit current conversation context load.
- `add-mcp-server` — generate a `claude mcp add` command from plain English.
- `list-mcp-servers` — show configured MCP servers.
- `mcp-command-help` — MCP CLI syntax help.

**Skills** (auto-invoked by description):

- `context-break-evaluator`
- `mcp-syntax-reference`
- `what-thing-pick`, `what-thing-help`, `what-thing-set-context`, `what-thing-refresh-docs`
- `feedback-bug`, `feedback-feature-request`, `feedback-model-behavior`, `feedback-docs`, `feedback-refresh-templates`

## Notes

- Workspace updates never conflict with plugin updates. Plugin primitives are global; this repo holds only user data.
- See the `meta-tools` plugin README for full primitive catalog.
