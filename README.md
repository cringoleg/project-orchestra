# epic-flow

GitHub-driven feature pipeline. One slash command, one PR per epic.

## Pipeline

```
/epic <feature-desc>
  → researcher (best-practices + repo scan)
  → grill (you answer until conclusion)
  → architect (DDD/TDD skeleton, picks 1-4 implementers)
  → grill (you answer until conclusion)
  → implementers run sequentially in one worktree
       data-eng → backend → frontend → devops  (default topo)
       each writes verification + commits + appends rules
  → finalize (build + push + PR opening, all stories closed by PR)
  → you merge → CI deploys
```

## Setup

Once per repo:

```
/epic-setup
```

Bootstraps GH labels, appends `.gitignore`, pre-creates `AGENTS.md` + `.claude/rules/<agent-type>.md`.

Requires `gh` CLI authenticated.

## Use

```
/epic <feature description>
```

Resume on dead session / crash:

```
/epic
```

(no args, run from inside repo with active epic dir → reads state, picks up).

### Gate verbs

At any user gate, reply one of:

- `continue` — advance current phase
- `retry` — re-spawn current agent with refreshed story
- `edit` — edit files yourself, then `continue` / `retry`
- `skip` — mark current story `done` manually (warns first; dangerous)
- `abort` — stop epic, set `phase:blocked`, preserve worktree

## Constraints

- One active epic per repo at a time. Hard-refused otherwise.
- Max 4 stories per epic (one per agent_type: `backend`, `frontend`, `devops`, `data-eng`).
- Sequential agents — no parallelism, no merge conflicts.
- Per-story green gate (working state per story). Finalize gate = full integration build.
- 2 retries per failure incident, then escalate.

## Source of truth

- **GitHub** — issue body, labels, comments, sub-issues, PR. Public, durable.
- **Local** (gitignored) — `<repo>/.claude/epics/gh-<N>/` narrative scratch (intake, decisions, story files, `index.json`).
- **Committed conventions** — `<repo>/AGENTS.md` + `<repo>/.claude/rules/<agent-type>.md` accumulate cross-epic.

## External tools

- `gh` CLI required.
- Implementers may invoke whatever Bash tools the repo's stack provides (docker, kubectl, terraform, psql, etc.).
- No MCP, no webhooks, no cron, no auto-merge.

## See

[DESIGN.md](DESIGN.md) for full spec.
