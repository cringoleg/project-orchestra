# epic-flow

Epic-to-story pipeline plugin. Manual gate every phase.

## Commands

- `/epic-intake <url|title>` — research + grill (Stage 1, implemented)
- `/epic-plan` — story breakdown + code-first plan (Stage 2, todo)
- `/epic-claim <story-id>` — worktree + branch (Stage 3, todo)
- `/epic-dispatch <story-id>` — coder subagent (Stage 4, todo)

## State

Per-epic dir at `<repo>/.claude/epics/<epic-id>/`. Gitignore it.

`epic-id` format:
- `sc-<n>` for existing Shortcut epics
- `draft-<slug>` for free-text drafts (no Shortcut record yet)

## See

[DESIGN.md](DESIGN.md) for full design.
