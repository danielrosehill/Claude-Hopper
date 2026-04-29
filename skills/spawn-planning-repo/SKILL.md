---
name: spawn-planning-repo
description: Create a sibling planning repo for the current project, seed it with planning context moved out of the main repo, and push to GitHub. Use when the user says "create a planning repo alongside this", "spin off planning", or "split planning into its own repo".
---

# Spawn Planning Repo

Create a dedicated planning repository that lives alongside the current code repo. Planning material (roadmaps, notes, specs, brainstorms) is moved out of the main repo into the new one, keeping the code repo lean.

## Steps

1. Identify the current repo root and parent directory:
   ```bash
   root=$(git rev-parse --show-toplevel)
   name=$(basename "$root")
   parent=$(dirname "$root")
   ```
2. Propose a planning repo name (default: `<name>-planning`) and confirm with the user.
3. Survey the current repo for planning-flavored files/dirs — e.g. `planning/`, `docs/planning`, `ROADMAP.md`, `NOTES.md`, `TODO.md`, `ideas/`, brainstorm markdown. Present the list to the user and confirm what should move. Do not guess silently.
4. Create the new repo directory and initialize it:
   ```bash
   mkdir "$parent/<planning-name>"
   cd "$parent/<planning-name>"
   git init -b main
   ```
5. Seed it with:
   - A `README.md` naming the sibling code repo and explaining this is its planning companion.
   - A `CLAUDE.md` briefly stating: this repo holds planning/spec/roadmap context for `<name>`, which lives at `<path>` / `<github url>`.
   - The planning files moved (via `git mv` in the source repo, then copied over — or `mv` if they're untracked) from the main repo.
6. In the main repo: remove the moved files with `git rm` (or `rm` if untracked), commit with a message like `chore: move planning into sibling <planning-name> repo`. Do NOT push the main repo unless the user asks.
7. In the new planning repo: `git add -A`, initial commit, then create the GitHub repo and push:
   ```bash
   gh repo create danielrosehill/<planning-name> --private --source . --remote origin --push
   ```
8. Report: new repo path, GitHub URL, list of files moved, and the pending (uncommitted-pushed) state of the main repo.

## Notes

- Always confirm the file list before moving anything — planning material is subjective.
- Use `git mv` then physically move when possible so history is preserved in the source commit that removes them.
- The new repo is private by default; ask if the user wants public.
