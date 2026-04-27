---
description: Research epic and grill user until requirements aligned
argument-hint: <url|title>
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, WebFetch
---

# /epic-intake

Arg: `$1` = Shortcut/Notion URL OR free-text title for new draft.

## Step 1 — Resolve epic-id and dir

Determine `epic-id`:
- If `$1` matches Shortcut URL or `sc-<n>`: extract numeric → `epic-id = sc-<n>`.
- If `$1` matches Notion URL: fetch page, derive `epic-id = notion-<page-id-short>` (first 8 chars of UUID).
- Otherwise treat `$1` as free text: `epic-id = draft-<slug>` where slug = kebab-case first 5 words.

Set `EPIC_DIR=<repo-root>/.claude/epics/<epic-id>`.

If `$EPIC_DIR/state.json` exists → **resume**: read phase, jump to Step 4 if phase ≠ `intake`, else continue grill at Step 5.

If new: create `$EPIC_DIR/`, write `state.json`:
```json
{ "phase": "intake", "epic_id": "<id>", "source": "<url-or-text>", "created_at": "<iso8601>", "updated_at": "<iso8601>" }
```

Ensure `.claude/epics/` is in repo `.gitignore`. If absent, append.

## Step 2 — Research (skip if `intake.md` already exists)

Spawn `epic-researcher` subagent with prompt:

> Research epic for `$EPIC_DIR/intake.md`. Source: `$1` (kind: `<shortcut|notion|draft>`).
>
> Budget: max 15 file reads, max 10 MCP/web fetches. Stop when budget hit.
>
> Output `intake.md` with sections: *Summary, Linked Docs, Affected Packages, Related Prior Work, Open Questions, Suspected Blockers*.
>
> For Shortcut epics: fetch via `mcp__...epics-get-by-id`, list linked stories, follow Notion links in description.
> For Notion: fetch via `mcp__...notion-fetch`. Follow inbound Shortcut/PR links.
> For drafts: write `Summary` from `$1`, leave Linked Docs empty, scan repo via Grep for keywords from title.
>
> Affected Packages: grep repo for keywords. List `api/internal/<pkg>/` paths only. Note canonical examples `api/internal/featureflags/` and `api/internal/giveaways/` if relevant.
>
> Related Prior Work: `git log --oneline --since=6.months --all` filtered by keyword. Top 5 only.
>
> Open Questions: bullet list. Each = a thing the planner will need answered to break this into stories. Aim 5–15.
>
> Suspected Blockers: cross-team deps, infra, missing schemas, ADR voids.

Wait for subagent completion. Print path + 5-line summary of `intake.md`.

## Step 3 — User reads intake

Stop. Tell user: "Read `<path-to-intake>`. Reply `continue` to start grill, or `edit` to revise intake first."

If `edit`: spawn researcher again with user's notes appended to original prompt. Loop.

If `continue`: proceed to Step 4.

## Step 4 — Init grill artifacts

Create if missing:
- `$EPIC_DIR/grill.md` — append-only Q/A log. Header: `# Grill log\n\n`.
- `$EPIC_DIR/decisions.md` — distilled decisions. Header: `# Decisions\n\n`.

## Step 5 — Grill loop

Invoke skill `grill-protocol` with context:
- intake.md path
- grill.md path (append every Q/A here)
- decisions.md path (write distilled resolved decisions here)
- termination = hybrid (you propose done with unresolved list, user confirms)

Grill protocol rules (enforced by skill):
- One question at a time.
- Each Q includes recommended answer + confidence (high/med/low).
- Mid-grill research: if not certain, **always ask permission before fetching**. Do not fetch silently.
- Append every Q + user answer to `grill.md` with timestamp.
- After each resolved branch, write a one-line decision to `decisions.md`.
- Walk dependency tree: ask root Qs first, drill into branches as answers come in.

When you (Claude) believe all branches resolved, propose:

> Grill done. Resolved: <count>. Unresolved/parked: <list or "none">. Confirm done or extend?

User confirms → Step 6. Else continue.

## Step 6 — Proposed epic body

Write `$EPIC_DIR/proposed-epic.md`. Sections:
- *Title*
- *Goal*
- *Scope*
- *Out of Scope*
- *Acceptance criteria* (epic-level, not story-level)
- *Open dependencies / blockers*

Print path + 5-line summary.

Tell user: "Paste `proposed-epic.md` into Shortcut/Notion manually. Reply `done` when posted to advance phase to `planning`, or `edit` to revise."

On `done`: update `state.json` phase → `planning`, `updated_at` → now. Print: "Phase → planning. Run `/epic-plan` next."

## Phase guard

If at start of command `state.json.phase` is anything other than `intake`, refuse with hint:
- `planning` → "Epic past intake. Run `/epic-plan`."
- `dispatch` → "Epic past planning. Run `/epic-claim <story-id>` or `/epic-dispatch <story-id>`."
- `done` → "Epic done. Edit `state.json` to reopen."

## Notes

- Working dir = repo root. All paths relative unless prefixed `<repo-root>`.
- Today's date for timestamps: shell `date -u +%Y-%m-%dT%H:%M:%SZ`.
- Never write to Shortcut/Notion. User pastes manually.
