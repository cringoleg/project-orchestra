---
description: One-time per repo. Bootstraps GitHub label taxonomy via gh CLI.
allowed-tools: Bash
---

# /epic-setup

Creates the epic-flow label taxonomy in current repo. Idempotent — safe to re-run.

Requires `gh` CLI authenticated and current dir inside target repo.

## Step 1 — Verify repo

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If error: tell user "current dir is not in a GitHub repo, or `gh auth login` not done. Aborting."

## Step 2 — Create labels

For each label below, run:

```bash
gh label create "<name>" --color "<hex>" --description "<desc>" --force
```

`--force` updates existing label color/description rather than failing.

### Labels (create all)

| Name | Color | Description |
|---|---|---|
| `type:epic` | `5319e7` | epic-flow: parent issue |
| `type:story` | `1d76db` | epic-flow: child story issue |
| `phase:intake` | `fbca04` | epic-flow: epic in research/grill |
| `phase:planning` | `f9d0c4` | epic-flow: epic awaiting story breakdown |
| `phase:dispatch` | `0e8a16` | epic-flow: stories created, ready for claim |
| `phase:done` | `cccccc` | epic-flow: all stories merged |
| `state:drafted` | `c5def5` | epic-flow: story planned, not claimed |
| `state:claimed` | `bfdadc` | epic-flow: story claimed, worktree created |
| `state:in-progress` | `0e8a16` | epic-flow: coder working / PR open |
| `state:done` | `cccccc` | epic-flow: PR merged, issue closed |
| `points:1` | `ededed` | epic-flow: 1 point |
| `points:2` | `ededed` | epic-flow: 2 points |
| `points:3` | `ededed` | epic-flow: 3 points |
| `points:5` | `ededed` | epic-flow: 5 points |
| `points:8` | `ededed` | epic-flow: 8 points |
| `points:13` | `ededed` | epic-flow: 13 points |
| `priority:highest` | `b60205` | epic-flow: priority highest |
| `priority:high` | `d93f0b` | epic-flow: priority high |
| `priority:medium` | `fbca04` | epic-flow: priority medium |
| `priority:low` | `0e8a16` | epic-flow: priority low |
| `priority:none` | `ededed` | epic-flow: priority none |
| `area:devops` | `5319e7` | epic-flow: devops/infra story |
| `area:data-eng` | `1d76db` | epic-flow: data-eng story |
| `area:backend` | `0e8a16` | epic-flow: backend story |
| `area:frontend` | `c5def5` | epic-flow: frontend story |
| `blocked` | `b60205` | epic-flow: story blocked by dep or external |

## Step 3 — Verify

```bash
gh label list --limit 200 | grep -E '^(type|phase|state|points|priority|area):|^blocked'
```

Print count of epic-flow labels found. If <26: warn user.

## Notes

- Add `area:` values per project as needed (e.g. `area:mobile`). Plugin convention: prefix matches handler logic in commands.
- To remove labels later: `gh label delete <name> --yes`.
- Run once per repo. Re-run safe (idempotent).
