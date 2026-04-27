# epic-flow — Design

Plugin. Epic → grilled requirements → DDD+TDD story plans → claimed → coded by isolated background subagent in worktree. Manual gate every phase.

## Non-goals

- No auto-push to Shortcut/Notion.
- No auto-PR.
- No multi-machine sync.
- No scheduled jobs.

## Plugin location

`~/.claude/plugins/epic-flow/`. Personal install.

## Commands (4)

| Command | Purpose |
|---|---|
| `/epic-intake <url\|title>` | URL = existing epic. Free text = local draft. Resumes if dir exists. |
| `/epic-plan` | Story breakdown + per-story code-first plan. |
| `/epic-claim <story-id>` | Mark + create worktree + branch. |
| `/epic-dispatch <story-id>` | Spawn background `story-coder` subagent in worktree. |

Future: `/epic-status`, `/epic-publish`, `/epic-unclaim`, `/epic-sync`.

## Subagents (4)

| Agent | When | Role |
|---|---|---|
| `epic-researcher` | `/epic-intake` | Bounded research → `intake.md`. Cap: 15 reads, 10 MCP fetches. |
| `story-planner` | `/epic-plan` | Scope + acceptance criteria. |
| `story-architect` | `/epic-plan`, after planner | Package tree, Go interfaces, failing tests. Code-first. Same story file. |
| `story-coder` | `/epic-dispatch` | Background subagent in worktree. Tight allowlist (Read/Edit/Write/Bash/Grep/Glob). No Agent, no MCP. TDD gate, commit-no-push. |

Grill = main session, interactive Q&A. Coder isolation = worktree (gitignored epic dir physically absent) + subagent prompt restricting reads to story file + listed rule files.

## State

### Layout (gitignored)

```
.claude/epics/<epic-id>/
  state.json
  intake.md
  grill.md
  decisions.md
  proposed-epic.md
  stories/
    1-<slug>.md
    2-<slug>.md
```

`epic-id` = `sc-<n>` or `draft-<slug>`. Add `.claude/epics/` to `.gitignore`.

### Epic states (4)

`intake` → `planning` → `dispatch` → `done`

### Story states (4)

`drafted` → `claimed` → `in-progress` → `done`

Blocker = stays `in-progress` + appended note.
Wrong-phase command = refuse with hint.

## Pipeline

### `/epic-intake <url|title>`

1. Detect URL vs free text. URL → fetch via Shortcut/Notion MCP. Text → local draft.
2. Resume if dir exists.
3. `epic-researcher` → `intake.md`.
4. Main session grills. One Q at a time. Recommended answer + confidence per Q. Mid-grill research = always ask permission. Append `grill.md`. Distill `decisions.md`.
5. Hybrid termination: agent proposes done with unresolved list. You confirm.
6. Write `proposed-epic.md`. You paste manually. Phase → `planning`.

### `/epic-plan`

1. Pass 1: `story-planner` writes `stories-proposal.md` table. Points (Fibonacci), Shortcut priority, deps, blocker notes. Soft cap 10.
2. You approve/edit/reject.
3. Pass 2 per story: planner expands → architect adds code (package tree, interfaces, failing tests). Same file. References `api/internal/featureflags/` + `api/internal/giveaways/` as DDD templates.
4. Batch review.
5. Approve all → stories `drafted`, epic `dispatch`.

### `/epic-claim <story-id>`

1. Refuse unless `drafted`.
2. Front matter: `claimed_by`, `claimed_at`.
3. Worktree `.claude/worktrees/sc-<id>-<slug>/`. Branch `sc-<id>-<slug>` off latest master.
4. State → `claimed`.

### `/epic-dispatch <story-id>`

1. Refuse unless `claimed`.
2. State → `in-progress`.
3. Spawn `story-coder` subagent in background, cwd = worktree. Prompt = absolute path to story file + behavior spec.
4. Parent session free. Notification on done.

Coder behavior:
- Read story file + listed rule files only.
- Write tests from *Test Scaffold* into worktree.
- `go test` → expect RED. Log to story file.
- `go tool mockery` if needed.
- Implement → GREEN.
- `task backend:lint`.
- `git add` + `git commit`. No push. Status `done`.
- Max 2 retries. Then blocker section + exit.

Status: coder writes directly to story file at absolute path (passed in spawn prompt). Worktree gitignored → epic dir physically absent → sibling artifacts unreadable.

## Story file template

```markdown
---
story_id: <n>
shortcut_id: <sc-id or empty>
slug: <kebab>
state: drafted
points: <fib>
priority: <shortcut value>
deps: []
external_blockers: []
claimed_by: ""
claimed_at: ""
branch: ""
---

# Goal
<paragraph>

# Acceptance Criteria
- TestX_doesY
- TestZ_returnsErrOnW

# Affected Packages / Files
- api/internal/<pkg>/...

# Out of Scope
- ...

# Rule files to load
- .claude/rules/<rule>.md

# Package Structure
\`\`\`
api/internal/<pkg>/
  handlers.go
  service.go
  internal/storage.go
\`\`\`

# Interfaces
\`\`\`go
\`\`\`

# Test Scaffold (failing)
\`\`\`go
\`\`\`

# Verification Commands
- task backend:lint -- ./api/internal/<pkg>/...
- go test -count=1 ./api/internal/<pkg>/...

# Migration
N/A — <reason>

# Estimate
<fib>

# --- coder fills below ---
# Status:
# Branch:
# Commits:
# Verification:
# Blockers:
```

## Plugin layout

```
~/.claude/plugins/epic-flow/
  DESIGN.md
  README.md
  commands/
    epic-intake.md
    epic-plan.md
    epic-claim.md
    epic-dispatch.md
  agents/
    epic-researcher.md
    story-planner.md
    story-architect.md
    story-coder.md
  skills/
    grill-protocol.md
  templates/
    intake.md
    story.md
    decisions.md
```

No hooks. No scheduled tasks. No auto-push.

## Build plan (vertical slice)

Stage 1 — `/epic-intake`
- `epic-researcher` agent
- grill protocol skill
- intake.md, grill.md, decisions.md, proposed-epic.md
- state.json + phase guards

Stage 2 — `/epic-plan`
- `story-planner` + `story-architect`
- stories-proposal.md
- per-story file
- batch review

Stage 3 — `/epic-claim`
- worktree + branch

Stage 4 — `/epic-dispatch`
- `story-coder` subagent (background, cwd = worktree)
- TDD gate, 2-retry, commit-no-push

Each stage = own session.

## Deferred

- `/epic-status`, `/epic-publish`, `/epic-unclaim`, `/epic-sync`
- Auto-push to Shortcut
- Shortcut comment with branch link on done
- Multi-machine sync
