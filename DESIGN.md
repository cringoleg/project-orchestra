# epic-flow — Design (v0.3)

GitHub-driven feature pipeline. Single command orchestrates research → architect → implementers → finalize → PR. One PR per epic. Sequential implementers in single epic worktree. Resumable on session death.

## Pipeline (canonical 13 steps)

1. User: `/epic <feature-desc>`.
2. Researcher writes `intake.md` with best-practices summary + repo affected packages + `Open Questions`.
3. Main session grills user via `grill-protocol`. User answers until conclusion.
4. Main session writes `proposed-epic.md`, pushes to GH issue body, flips `phase:research → phase:architect`. Spawns architect.
5. Architect writes per-story files (DDD/TDD skeleton + agent_type) + `architect-summary.md`.
6. Main session grills user holistically on architect's plan. User confirms.
7. Main session batch-creates GH sub-issues, updates epic body task-list, flips `phase:architect → phase:implementing`. Dispatches implementers in topo order, sequential.
8. Each implementer agent runs its pipeline (verification gate, code, distill rules, squash commit) in shared worktree until done.
9. Main session runs finalize: `index.json.verify` (lint → test → build) in worktree. On green: pushes branch, opens PR `Closes #<epic>` + per-story closes. Flip `phase:finalize → phase:review`.
10. PR contains all stories tagged via `Closes #<sub>` lines.
11. On any failure: 2 retries per incident, then escalate to user (`phase:blocked`).
12. User merges PR. CI auto-deploys. Plugin not involved.
13. `/epic-done` (or `/epic` resume detecting merge) closes epic issue + sub-issues, flips `phase:done`.

## Source of truth

GitHub. Issue body, labels, sub-issues, PR. Authoritative on conflict.

Local mirror = `<repo>/.claude/epics/gh-<N>/` narrative scratch. Reconciled to GH on every `/epic` invocation.

## Plugin location

`~/.claude/plugins/epic-flow/`. Personal install. Settings.json registers via plugins config.

## Commands (2)

| Command | Purpose |
|---|---|
| `/epic-setup` | One-time per repo. Labels + gitignore + rule file scaffolds. |
| `/epic [<desc>\|<#N>\|<url>]` | Entry + resume. No-arg from epic dir = resume current phase. |

All other behavior surfaces via in-flow gate verbs (`continue`/`retry`/`edit`/`skip`/`abort`).

## Agents (6)

| Agent | When | Role |
|---|---|---|
| `researcher` | research stage | Bounded research (repo + GH + ≤5 web). Writes `intake.md`. |
| `architect` | architect stage | DDD/TDD skeleton. Picks subset of implementer agents. Writes story files + `architect-summary.md`. |
| `backend` | implementing stage (when assigned) | Server logic, handlers, services, domain code. |
| `frontend` | implementing stage (when assigned) | UI, components, client wiring. |
| `devops` | implementing stage (when assigned) | CI, manifests, env wiring. NOT the finalize step (main session does that). |
| `data-eng` | implementing stage (when assigned) | Migrations, seeds, ETL scripts. Code only — no spec-only mode. |

Max 4 implementer stories per epic. One agent_type used at most once per epic. If work spans two backend chunks → fold into single backend story w/ multiple acceptance criteria.

## Skills (1)

| Skill | Used by |
|---|---|
| `grill-protocol` | Main session at research grill + architect grill. |

No language-specific skills bundled. Repo's `AGENTS.md` / `CLAUDE.md` / `.claude/rules/<topic>.md` carry stack conventions.

## State model

### GitHub side (public truth)

- **Epic** = parent issue. Labels: `type:epic` + one `phase:*`.
- **Story** = child issue. Labels: `type:story` + one `state:*` + one `agent:*`.
- **Epic ↔ stories** = epic body has `## Stories` task-list (`- [ ] #N <slug> (agent:<type>)`).
- **PR ↔ epic + stories** = PR body has `Closes #<epic>` + `Closes #<sub-issue>` per story.

### Local side (narrative)

```
<repo>/.claude/epics/gh-<N>/
  intake.md            # researcher output
  grill.md             # append-only Q&A log
  decisions.md         # distilled grill output
  proposed-epic.md     # epic body draft
  architect-summary.md # architect's plan overview
  index.json           # state mirror + verify cmds + story manifest
  stories/
    backend.md         # one file per assigned implementer
    frontend.md
    devops.md
    data-eng.md
```

`.claude/epics/` is gitignored.

### Worktree

```
<repo>/.claude/worktrees/epic-gh-<N>-<slug>/
```

Single worktree per epic. Branch `epic/gh-<N>-<slug>` off latest default branch. Sequential implementers commit linearly.

`.claude/worktrees/` is gitignored.

### `index.json` schema

```json
{
  "epic_id": "gh-123",
  "issue_number": 123,
  "slug": "ride-history",
  "phase": "implementing",
  "active_story": "backend",
  "gate_pending": null,
  "last_transition": "2026-04-29T08:00:00Z",
  "verify": {
    "lint":  ["go vet ./...", "golangci-lint run"],
    "test":  ["go test ./..."],
    "build": ["go build ./..."]
  },
  "stories": [
    { "agent_type": "data-eng", "issue_number": 124, "slug": "ride-history-schema",  "state": "done",        "deps": [] },
    { "agent_type": "backend",  "issue_number": 125, "slug": "ride-history-handler", "state": "in-progress", "deps": ["data-eng"] },
    { "agent_type": "frontend", "issue_number": 126, "slug": "ride-history-page",    "state": "drafted",     "deps": ["backend"] }
  ]
}
```

Story array ordered = topo dispatch order.

### Phases (epic, label-mutex)

`phase:research` → `phase:architect` → `phase:implementing` → `phase:finalize` → `phase:review` → `phase:done`

Plus `phase:blocked` (any phase can transition to it; resume returns to last phase).

### States (story, label-mutex)

`state:drafted` → `state:in-progress` → `state:done`

Plus `state:blocked` (replaces `state:in-progress` when retries exhausted).

## Label taxonomy (17 total)

| Prefix | Values | Mutex |
|---|---|---|
| `type:` | `epic`, `story` | yes |
| `phase:` | `research`, `architect`, `implementing`, `finalize`, `review`, `blocked`, `done` | yes (epic) |
| `state:` | `drafted`, `in-progress`, `done`, `blocked` | yes (story) |
| `agent:` | `backend`, `frontend`, `devops`, `data-eng` | yes (story) |

Mutex enforced by convention — plugin removes old prefix-mate before adding new.

## Verification

Two-level:

1. **Story-level gate** — each implementer's story FrontMatter has `# Verification Commands`. Must pass clean before agent commits.
2. **Epic-level finalize gate** — main session runs `index.json.verify` (`lint` → `test` → `build`) in worktree. Stop on first fail. On fail: regex-extract file paths from stderr → match story `Affected Files` → re-dispatch single matched agent. Multi-match / no-match → escalate.

Architect detects + writes `index.json.verify` during architect pass. User edits at architect grill if wrong.

## Retry + escalation

Per-incident counter. Same agent fails on same root cause: 2 retries → escalate.

Different failure on same agent later in epic = fresh 2 retries.

Escalate path: set `phase:blocked` (epic) + `state:blocked` (story) + write story's `# Blockers` section + `index.json.gate_pending = { story_id, reason, last_error }`. Yield to user with context. User replies one of:

| Verb | Effect |
|---|---|
| `continue` | Re-check current state. User fixed in place. Advance if green. |
| `retry` | Re-spawn agent w/ refreshed story file. Counter resets. |
| `edit` | User edits files outside session. Reply `continue` / `retry` / `abort` after. |
| `skip` | Mark story `done` manually. Warns first. Dangerous. |
| `abort` | Stop epic. Preserve worktree + epic dir. Clean exit. |

## Convention accumulation

Each cycle, agents append rules to persistent files. Files version-controlled with epic PR.

| File | Owner | Scope |
|---|---|---|
| `<repo>/AGENTS.md` | main session (post-grill, both grills) | project-wide cross-cutting |
| `<repo>/.claude/rules/<agent-type>.md` | implementer (pre-commit, ≤5 rules, dedup) | per-agent specialized |
| `<repo>/.claude/rules/<topic>.md` (e.g. `migrations.md`, `error-handling.md`) | architect (when applicable) | topical fine-grained |

Rule format (one per line):

```
- <rule one line>. (gh-<N>:<agent-type> <source: grill|architect|self>)
```

Implementer's commit includes code + rule file changes atomically.

Rule file bloat: cap watch (~50 rules per file). Future maintenance pass for compression. No auto-prune now.

## File topology + read-scope per implementer

Worktree = epic branch checkout. `.claude/epics/` gitignored → physically absent in worktree.

Each implementer subagent receives spawn prompt with:
- Worktree absolute path (cwd)
- Absolute path to own story file (in epic dir, outside worktree)
- Absolute paths to rule files listed in story FrontMatter
- `gh_issue` (story sub-issue number), `branch`, `agent_type`

**ALLOWED**:
- Worktree contents (cwd)
- Own story file (absolute path)
- Listed rule files (absolute paths)
- Repo-root `AGENTS.md` + `CLAUDE.md` (auto-loaded by Claude Code)
- Own `<repo>/.claude/rules/<agent-type>.md` (rule append target)

**FORBIDDEN**:
- Sibling story files in epic dir
- `intake.md`, `grill.md`, `decisions.md`, `proposed-epic.md`, `architect-summary.md`
- Other epic dirs
- `list-traverse` parent of own story file

Predecessor's work visible via committed code in worktree, not via predecessor's story file.

## PR shape

**Title**: `<epic-slug>` (matches epic issue title).

**Body**:

```markdown
Closes #<epic-issue>

Closes #<backend-sub>
Closes #<frontend-sub>
Closes #<devops-sub>
Closes #<data-eng-sub>

## Stories
- `backend` (#<n>): <one-line summary>
- `frontend` (#<n>): <one-line summary>
- `devops` (#<n>): <one-line summary>
- `data-eng` (#<n>): <one-line summary>

## Verification
- lint: clean
- test: clean
- build: clean

## Conventions delta
- AGENTS.md: <count> new rules
- .claude/rules/backend.md: <count> new rules

🤖 epic-flow
```

## Commit format per agent

Conventional Commits, agent_type as scope:

```
<type>(<agent>): <subject ≤50 chars>

<optional body>
```

`type` ∈ {`feat`, `fix`, `refactor`, `test`, `chore`, `docs`, `ci`}.

Squash-at-end per agent. One commit per agent_type. PR has ≤4 commits.

Repo's `CLAUDE.md` / `AGENTS.md` may declare different convention — agents respect when present.

## Resume + session-death contract

1. **Disk-only state.** Every transition writes `index.json` + GH label before yielding.
2. **Pre-yield commit.** Before any user gate or subagent spawn, write state.
3. **Implementer resumability.** Per-TDD-cycle commits during agent work. On orphan, re-spawn re-reads story + last commit log.
4. **Resume detection.** `/epic` no-arg from inside epic dir:
   - Read `index.json`. Read GH labels.
   - If GH `phase:*` ≠ local → trust GH, rewrite local.
   - If `gate_pending` set → re-print gate prompt.
   - If `active_story` set + no PR + no recent commit (>10 min on branch) → orphan. Re-spawn implementer.
   - Else advance from current phase.
5. **Idempotent transitions.** Phase flips check current state first. Already-advanced = no-op + warn.

## Cleanup matrix

| Terminal state | Local epic dir | Worktree | Branch |
|---|---|---|---|
| `phase:done` (PR merged) | delete (default) / archive on user `archive` | `git worktree remove` | keep (already merged) |
| `phase:blocked` + `abort` | keep (debug context) | `git worktree remove` | keep (user can resurrect) |
| `phase:blocked` + `continue`/`retry` (resume) | keep | keep | keep |

## `/epic-setup` (one-time per repo)

1. **Preflight** — `gh repo view` to verify cwd is GH repo + `gh` authed. Abort with hint if not.
2. **Create labels** — 17 labels via `gh label create --force` (idempotent).
3. **Append `.gitignore`** (idempotent):
   ```
   .claude/epics/
   .claude/worktrees/
   ```
4. **Pre-create rule files** — `<repo>/AGENTS.md` (header only if absent), `<repo>/.claude/rules/{backend,frontend,devops,data-eng}.md` (each: `# <agent_type> rules\n\n`).
5. **Print summary** — labels created/updated count, gitignore status, rule files initialized.

Out of scope for setup: `verify` config (architect populates per-epic in `index.json`).

## `/epic <arg>` (entry + resume)

### Argument forms

- **Free-text** → create new epic issue. Title = arg. `gh issue create --title "<arg>" --label type:epic,phase:research --body "Draft"`. Capture `<N>`. Init `<repo>/.claude/epics/gh-<N>/`.
- **`#N`** or **`https://github.com/.../issues/<N>`** → fetch existing issue. Treat as fresh-from-existing if `phase:research`. Resume otherwise.
- **No arg, inside epic dir** → resume current phase.

### Single-active-epic guard

Before any new-epic action: `gh issue list --label type:epic --state open --json number,labels`. If any has `phase:* != phase:done` → refuse, print existing epic + offer abort/resume.

### Pipeline (orchestrator pseudocode)

```
1. Resolve arg → epic-id, gh_issue_number, epic-dir.
2. Single-active-epic guard (if new epic).
3. Read state (index.json + GH labels). Reconcile if drift.
4. Switch on phase:
     phase:research:
       if no intake.md → spawn researcher (foreground subagent).
       invoke grill-protocol on intake's Open Questions.
       on grill confirm:
         distill ≤3 project-wide rules from decisions.md → append AGENTS.md.
         write proposed-epic.md.
         user confirms → push epic body, flip phase:research → phase:architect.
     phase:architect:
       if no story files → spawn architect (foreground subagent).
       invoke grill-protocol on architect-summary.md (holistic).
       on grill confirm:
         distill ≤3 project-wide rules → append AGENTS.md.
         batch-create sub-issues. Update epic body task-list. Flip phase.
         create worktree + branch.
     phase:implementing:
       for story in index.json.stories (topo order):
         if story.state == done → skip.
         spawn implementer subagent (background).
         monitor → on done, mark state:done.
         on fail: retry (≤2). On 2nd fail: escalate.
       all done → flip phase:implementing → phase:finalize.
     phase:finalize:
       run index.json.verify (lint → test → build).
       on green → push branch, open PR (Closes ...). Flip phase:finalize → phase:review.
       on fail → diagnose → re-dispatch matching agent (counts as retry).
     phase:review:
       gh pr view <pr> --json state,mergedAt.
       if MERGED → close epic + sub-issues, flip phase:review → phase:done. Cleanup per matrix.
       else → print "PR #<n> still open. Merge to advance."
     phase:done:
       print "epic done."
     phase:blocked:
       print blocker + gate_pending. Wait for verb.
```

### Gate verbs (any phase)

`continue`, `retry`, `edit`, `skip`, `abort`. Behaviors per Retry + escalation section.

### Mid-implement abort

User types `abort` (or `/epic abort`) anytime. Main session monitors → on user message during background subagent → kill subagent → revert in-progress story to `state:blocked` → set epic `phase:blocked` → yield. Worktree changes preserved.

## Templates

`templates/`:
- `intake.md` — researcher output shape
- `decisions.md` — grill distill shape
- `proposed-epic.md` — epic body shape
- `story.md` — per-agent story file shape
- `architect-summary.md` — architect plan overview shape
- `index.json` — state schema seed

## Plugin layout

```
~/.claude/plugins/epic-flow/
  README.md
  DESIGN.md
  .claude-plugin/plugin.json
  commands/
    epic-setup.md
    epic.md
  agents/
    researcher.md
    architect.md
    backend.md
    frontend.md
    devops.md
    data-eng.md
  skills/
    grill-protocol/SKILL.md
  templates/
    intake.md
    decisions.md
    proposed-epic.md
    story.md
    architect-summary.md
    index.json
```

No hooks. No scheduled tasks. No background workers beyond per-epic implementer subagents.

## Non-goals

- No multi-epic concurrency.
- No multi-PR per epic.
- No auto-merge.
- No webhooks, no cron.
- No PM-tool integrations (Shortcut/Notion/Jira).
- No MCP servers required.
- No multi-machine sync.
