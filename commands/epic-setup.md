---
description: One-time per repo. Bootstraps GitHub label taxonomy + .gitignore + rule file scaffolds.
allowed-tools: Bash, Write, Read
---

# /epic-setup

Idempotent. Safe to re-run. Required before first `/epic <desc>`.

## Step 1 — Preflight

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If error: print "current dir not in a GitHub repo OR `gh auth login` not done. Aborting." Exit.

## Step 2 — Create labels (17 total)

For each row, run:

```bash
gh label create "<name>" --color "<hex>" --description "<desc>" --force
```

`--force` updates existing label color/description rather than failing.

| Name | Color | Description |
|---|---|---|
| `type:epic` | `5319e7` | epic-flow: parent issue |
| `type:story` | `1d76db` | epic-flow: child story issue |
| `phase:research` | `fbca04` | epic-flow: epic in research/grill |
| `phase:architect` | `f9d0c4` | epic-flow: epic awaiting architect skeleton |
| `phase:implementing` | `0e8a16` | epic-flow: implementer agents running |
| `phase:finalize` | `c5def5` | epic-flow: integration build + push + PR open |
| `phase:review` | `bfdadc` | epic-flow: PR open, awaiting human merge |
| `phase:blocked` | `b60205` | epic-flow: gate-pending, user input required |
| `phase:done` | `cccccc` | epic-flow: PR merged, epic closed |
| `state:drafted` | `c5def5` | epic-flow: story planned, not started |
| `state:in-progress` | `0e8a16` | epic-flow: implementer running |
| `state:done` | `cccccc` | epic-flow: implementer's commits + verify clean |
| `state:blocked` | `b60205` | epic-flow: implementer retries exhausted |
| `agent:backend` | `0e8a16` | epic-flow: backend story |
| `agent:frontend` | `c5def5` | epic-flow: frontend story |
| `agent:devops` | `5319e7` | epic-flow: devops story |
| `agent:data-eng` | `1d76db` | epic-flow: data-eng story |

## Step 3 — Append `.gitignore` (idempotent)

If `<repo>/.gitignore` missing, create it. Append entries only if absent:

```
.claude/epics/
.claude/worktrees/
```

Detect via:
```bash
grep -qxF '.claude/epics/' .gitignore || echo '.claude/epics/' >> .gitignore
grep -qxF '.claude/worktrees/' .gitignore || echo '.claude/worktrees/' >> .gitignore
```

## Step 4 — Pre-create rule file scaffolds

Create if absent (never overwrite):

- `<repo>/AGENTS.md`:
  ```
  # AGENTS.md

  Project-wide conventions accumulated by epic-flow. Append-only by main session post-grill.
  ```

- `<repo>/.claude/rules/backend.md`:
  ```
  # backend rules
  ```
- `<repo>/.claude/rules/frontend.md`:
  ```
  # frontend rules
  ```
- `<repo>/.claude/rules/devops.md`:
  ```
  # devops rules
  ```
- `<repo>/.claude/rules/data-eng.md`:
  ```
  # data-eng rules
  ```

(`mkdir -p .claude/rules` first.)

## Step 5 — Verify + summary

```bash
gh label list --limit 200 | grep -E '^(type|phase|state|agent):' | wc -l
```

Print:
- Labels: `<n>/17` epic-flow labels found.
- `.gitignore`: entries `<count-added|already-present>`.
- Rule files: `<count-created|already-present>`.

Warn if labels < 17.

## Notes

- Re-run safe.
- To remove labels later: `gh label delete <name> --yes`.
- Out of scope: `verify` config (architect populates per-epic in `<epic-dir>/index.json`).
