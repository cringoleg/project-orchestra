---
description: Research epic and grill user until requirements aligned. Auto-creates GitHub issue if title given. Resumes if existing.
argument-hint: <issue-url|#N|title>
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---

# /epic-intake

Arg: `$1` = GitHub issue URL, `#N` / numeric issue ref, OR free-text title for new epic.

Requires `gh` CLI authenticated. Run `/epic-setup` first if labels not bootstrapped.

## Step 1 — Resolve epic-id and dir

Detect arg shape:

- **URL** matching `https://github.com/<owner>/<repo>/issues/<N>` → `gh_issue=N`. Existing.
- **`#?\d+`** → `gh_issue=<N>`. Existing.
- **Else** → free-text title. Create new issue:
  ```bash
  gh issue create --title "$1" --label type:epic,phase:intake --body "Draft — populated by /epic-intake."
  ```
  Capture `<N>` from output URL.

`epic-id = gh-<N>`.

`EPIC_DIR=<repo-root>/.claude/epics/gh-<N>`.

If `$EPIC_DIR/` exists → **resume**: read `$EPIC_DIR/index.json` if present. Read current phase from issue label (`gh issue view <N> --json labels`). If `phase:intake` → continue grill. If past intake → refuse with hint.

If new dir: create `$EPIC_DIR/`. Write `$EPIC_DIR/index.json`:

```json
{
  "epic_id": "gh-<N>",
  "issue_number": <N>,
  "stories": {}
}
```

Ensure `.claude/epics/` is in repo `.gitignore`. If absent, append.

## Step 2 — Research (skip if `intake.md` already exists)

Spawn `epic-researcher` subagent with prompt:

> Research epic. Output: `$EPIC_DIR/intake.md`.
>
> Inputs:
> - epic-id: `gh-<N>`
> - gh_issue_number: `<N>`
> - source_arg: `$1`
> - output_path: `$EPIC_DIR/intake.md`
>
> Budget: 15 reads, 10 fetches. Stop when hit.
>
> Steps: fetch issue body+comments via `gh issue view <N> --json title,body,comments,labels,assignees,url`; follow primary `#N` refs in body; grep repo for keywords; `git log` since 6 months filtered; `gh search issues` for closed related.
>
> Sections: *Summary, Linked Issues / PRs, Affected Packages, Related Prior Work, Open Questions, Suspected Blockers*. Aim 5–15 Open Questions.

Wait for completion. Print path + 5-line summary.

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

## Step 6 — Proposed epic body + push

Write `$EPIC_DIR/proposed-epic.md` from `templates/proposed-epic.md`. Sections:
- *Title* (matches issue title; rename via `gh issue edit <N> --title` if user changed it during grill)
- *Goal*
- *Scope*
- *Out of Scope*
- *Acceptance criteria* (epic-level, not story-level)
- *Open dependencies / blockers*
- *Stories* (empty placeholder — populated at end of `/epic-plan`)

Print path + 5-line summary.

Tell user: "Reply `confirm` to push to GitHub issue body + advance phase. Reply `edit` to revise."

On `confirm`:
1. Push body:
   ```bash
   gh issue edit <N> --body-file "$EPIC_DIR/proposed-epic.md"
   ```
2. Flip label:
   ```bash
   gh issue edit <N> --remove-label phase:intake --add-label phase:planning
   ```
3. Print: "Phase → planning. Issue: `<url>`. Run `/epic-plan` next."

## Phase guard

At start of command, if issue label is anything other than `phase:intake`, refuse with hint:
- `phase:planning` → "Epic past intake. Run `/epic-plan`."
- `phase:dispatch` → "Epic past planning. Run `/epic-claim <story-id>` or `/epic-dispatch <story-id>`."
- `phase:done` → "Epic done. Re-open issue + flip label to reopen."

## Notes

- Working dir = repo root. All paths relative unless prefixed `<repo-root>`.
- Today's date for timestamps: shell `date -u +%Y-%m-%dT%H:%M:%SZ`.
- Never bypass `gh` (no curl + token). User authenticates via `gh auth login`.
