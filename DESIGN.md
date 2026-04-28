# epic-flow — Design (v0.2)

GitHub-only plugin. Epic → grilled requirements → DDD+TDD story plans → claimed → coded by isolated background subagent in worktree. Manual gate at PR review. Push-only, no sync.

## Philosophy

- **GitHub = public truth.** Issue body + labels = epic/story state. Anyone with repo access sees current state without local context.
- **Local = private narrative.** Grill transcript, intake research, decisions stay local (often messy, often personal).
- **Push-only, lean calls.** Plugin pushes on transitions. No background poll. Targeted pulls only at intake/claim/done. ~3 fetches per story lifecycle.
- **Manual code review.** Coder writes + commits, opens PR. Human reviews + merges. `/epic-done` polls PR-merge once, then closes issue.

## Non-goals

- No Shortcut, no Notion, no any other PM tool.
- No MCP servers required.
- No background poll, no cron, no webhooks.
- No auto-merge.
- No multi-machine sync.

## Plugin location

`~/.claude/plugins/epic-flow/`. Personal install.

## Commands (8)

| Command | Purpose |
|---|---|
| `/epic-setup` | One-time per repo. Creates label taxonomy via `gh label create`. |
| `/epic-intake <url\|#N\|title>` | Auto-detect. URL/`#N` = fetch existing issue. Title = create new issue + draft. |
| `/epic-plan` | Story breakdown + per-story DDD code-first plan. Push child issues at end. |
| `/epic-claim <story-id>` | Worktree + branch + assign issue + `state:claimed`. |
| `/epic-dispatch <story-id>` | Background `story-coder` subagent. `state:in-progress`. |
| `/epic-done <story-id>` | Check PR merged. If yes → close issue + `state:done`. Else hint. |
| `/devops-check <topic>` | Ad-hoc devops-agent invocation. |
| `/data-search <query>` | Ad-hoc data-engineer invocation. |

## Subagents (6)

| Agent | When | Role |
|---|---|---|
| `epic-researcher` | `/epic-intake` | Bounded research → `intake.md`. Cap: 15 reads, 10 `gh`/web fetches. |
| `story-planner` | `/epic-plan` pass 1 | Story list with scope, points, priority, deps, `area:` hint label. |
| `story-architect` | `/epic-plan` pass 2 | DDD package tree, interfaces, failing tests. Must grill before spawning subagent. |
| `story-coder` | `/epic-dispatch` | Background subagent in worktree. TDD, commit + PR open with `Closes #<N>`. No merge. |
| `devops-agent` | architect-spawned or `/devops-check` | Consultative. Inspects deploy state (docker/k8s/terraform/CI). 10 reads / 5 bash. |
| `data-engineer` | architect-spawned or `/data-search` | Consultative. Searches sources, emits `data-spec-<name>.md`. 20 reads / 10 bash. |

Grill = main session, interactive Q&A via `grill-protocol` skill. Coder isolation = worktree (gitignored epic dir physically absent) + subagent prompt restricting reads to story file + listed rule files.

## State model

### GitHub side (public truth)

- **Epic** = parent issue, label `type:epic` + one `phase:*` label.
- **Story** = child issue, label `type:story` + one `state:*` label + `points:N` + `priority:X` + optional `area:*` + optional `blocked`.
- **Dependencies**: story body has `## Blocked by` section with `#N` refs. `blocked` label set when section non-empty + any ref open.
- **Epic ↔ stories link**: epic body task-list refs each story (`- [ ] #M description`). Auto-rendered cross-links by GitHub.
- **PR ↔ story link**: PR body has `Closes #<N>`. Coder always emits this.

### Local side (narrative + memory)

```
<repo>/.claude/epics/gh-<N>/
  intake.md
  grill.md
  decisions.md
  proposed-epic.md
  index.json          # story-id → issue-number map + last-sync timestamps
  data-specs/
    <name>.md         # data-engineer outputs
  stories/
    1-<slug>.md       # FrontMatter holds gh_issue, last_synced_at, etc.
    2-<slug>.md
```

`.claude/epics/` = gitignored. Worktree = fresh checkout, epic dir physically absent → coder forbidden by topology.

### `index.json` schema

```json
{
  "epic_id": "gh-123",
  "issue_number": 123,
  "stories": {
    "1": { "issue_number": 124, "slug": "rename-foo", "last_synced_at": "..." },
    "2": { "issue_number": 125, "slug": "split-bar", "last_synced_at": "..." }
  }
}
```

Tiny lookup file. Used to batch ops (iterate stories without grep) and to cache issue numbers without re-fetch.

### Phases (epic, label-mutex)

`phase:intake` → `phase:planning` → `phase:dispatch` → `phase:done`

### States (story, label-mutex)

`state:drafted` → `state:claimed` → `state:in-progress` → `state:done`

Blocker = stays `state:in-progress` + `blocked` label + `## Blockers` body section appended.
Wrong-phase command = refuse with hint.

## Label taxonomy

| Prefix | Values | Mutex? |
|---|---|---|
| `type:` | `epic`, `story` | yes |
| `phase:` | `intake`, `planning`, `dispatch`, `done` | yes (epic only) |
| `state:` | `drafted`, `claimed`, `in-progress`, `done` | yes (story only) |
| `points:` | `1`, `2`, `3`, `5`, `8`, `13` | yes |
| `priority:` | `highest`, `high`, `medium`, `low`, `none` | yes |
| `area:` | `devops`, `data-eng`, `backend`, `frontend`, … | additive |
| `blocked` | (boolean) | flag |

Mutex enforced by convention (plugin removes old prefix-mate before adding new).

## Pipeline

### `/epic-setup`

`gh label create` for each label in taxonomy. Idempotent (`--force` if exists). Run once per repo.

### `/epic-intake <url|#N|title>`

1. **Auto-detect arg**:
   - `https://github.com/.../issues/N` or `^#?\d+$` → fetch existing issue. `epic-id = gh-<N>`.
   - Else → create new issue: `gh issue create --title "<arg>" --label type:epic,phase:intake`. `epic-id = gh-<N>` from response.
2. **Resume** if `<repo>/.claude/epics/gh-<N>/` exists. Read current state (label `phase:*` on issue).
3. Spawn `epic-researcher` → `intake.md`.
4. Main session grills via `grill-protocol`. Append `grill.md`. Distill `decisions.md`.
5. Hybrid termination — agent proposes done with unresolved list. User confirms.
6. Write `proposed-epic.md`. Push to issue body via `gh issue edit <N> --body-file proposed-epic.md`. Flip label `phase:intake` → `phase:planning`. Print: "Phase → planning. Run `/epic-plan` next."

### `/epic-plan`

1. **Pass 1**: spawn `story-planner` (pass=1). Reads `intake.md`, `decisions.md`. Writes `stories-proposal.md` table (soft cap 10). Includes `area:` hint per story.
2. User approves/edits/rejects.
3. **Pass 2 per story**: spawn `story-planner` (pass=2) per row → fills story file scope/criteria. Then spawn `story-architect` per story → fills Package Structure, Interfaces, Test Scaffold.
   - **Architect grill rule**: before architect spawns `devops-agent` or `data-engineer`, it MUST run `grill-protocol` with user. Once grilled, spawns subagent with focused brief.
4. Batch review.
5. Approve all → for each story: `gh issue create --title "<slug>" --label type:story,state:drafted,points:N,priority:X[,area:Y]` → write `gh_issue` to story FrontMatter + `index.json`. Update epic body with task-list of child issues. Flip epic `phase:planning` → `phase:dispatch`.

### `/epic-claim <story-id>`

1. Refuse unless story label = `state:drafted`.
2. Targeted pull: `gh issue view <story-issue>` to confirm open + unassigned.
3. Worktree `<repo>/.claude/worktrees/story-gh-<N>-<slug>/`. Branch `story/gh-<N>-<slug>` off latest default branch.
4. Push: `gh issue edit <N> --add-assignee @me`, flip `state:drafted` → `state:claimed`.
5. Update story FrontMatter: `claimed_by`, `claimed_at`, `branch`.

### `/epic-dispatch <story-id>`

1. Refuse unless `state:claimed`.
2. Flip `state:claimed` → `state:in-progress`.
3. Spawn `story-coder` background subagent in worktree. Prompt = absolute story file path + behavior spec.
4. Parent session free. Notification on subagent done.

Coder behavior:
- Read story file + listed rule files only.
- Write tests from *Test Scaffold* into worktree.
- `<verify-test-cmd>` → expect RED. Append to story file under `# Verification`.
- Implement → GREEN.
- `<verify-lint-cmd>` clean.
- `git add` + `git commit`. Push branch. `gh pr create --title "<slug>" --body "Closes #<story-issue>\n\n<commit list>"`.
- Status `done` in story FrontMatter (local). Issue stays `state:in-progress` until `/epic-done`.
- Max 2 retries. Then `## Blockers` section + exit, `blocked` label pushed.

### `/epic-done <story-id>`

1. Targeted pull: `gh pr view <pr-number> --json state,mergedAt`. PR number stored in story FrontMatter at dispatch time.
2. If `MERGED`: close story issue (`gh issue close <N> --reason completed`), flip `state:in-progress` → `state:done`. If all stories `state:done`: epic `phase:dispatch` → `phase:done`, close epic issue.
3. Else: print PR state, hint user to review/merge.

## Story file template

```markdown
---
story_id: <n>
gh_issue: <number-or-empty>
gh_pr: <number-or-empty>
slug: <kebab>
state: drafted
points: <fib>
priority: <highest|high|medium|low|none>
area: <devops|data-eng|backend|frontend|""> 
deps: []
external_blockers: []
claimed_by: ""
claimed_at: ""
branch: ""
last_synced_at: ""
---

# Goal
<paragraph>

# Acceptance Criteria
- TestX_doesY

# Affected Packages / Files
- <path>

# Out of Scope
- ...

# Rule files to load
- .claude/rules/<rule>.md

# Blocked by
- (empty | list of #N refs)

# Package Structure
\`\`\`
<tree>
\`\`\`

# Interfaces
\`\`\`go
\`\`\`

# Test Scaffold (failing)
\`\`\`go
\`\`\`

# Verification Commands
- <lint cmd>
- <test cmd>

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
  .claude-plugin/plugin.json
  commands/
    epic-setup.md
    epic-intake.md
    epic-plan.md
    epic-claim.md
    epic-dispatch.md
    epic-done.md
    devops-check.md
    data-search.md
  agents/
    epic-researcher.md
    story-planner.md
    story-architect.md
    story-coder.md
    devops-agent.md
    data-engineer.md
  skills/
    grill-protocol/SKILL.md
  templates/
    intake.md
    story.md
    proposed-epic.md
    decisions.md
    data-spec.md
    index.json
```

No hooks. No scheduled tasks. No background workers.

## Call budget per epic lifecycle (typical)

| Event | `gh` calls |
|---|---|
| `/epic-setup` | ~12 (one-time, label creates) |
| `/epic-intake` (new) | 1 (issue create) + 1 (body push at phase flip) + 1 (label flip) = 3 |
| `/epic-intake` (existing) | 1 (issue view) + 2 (push + flip) = 3 |
| `/epic-plan` | N (issue create per story) + 1 (epic body update) + 1 (phase flip) ≈ N+2 |
| `/epic-claim <story>` | 1 (view) + 1 (assign) + 1 (label flip) = 3 |
| `/epic-dispatch <story>` | 1 (label flip) |
| coder PR open | 1 (`gh pr create`) |
| `/epic-done <story>` | 1 (PR view) + 1 (issue close) + 1 (label flip) = 3 |

For 5-story epic: ~12 (setup, amortized) + 3 + 7 + (3+1+3)·5 = 22 + 35 = ~57 calls total. Pre-migration estimate (full Q6 auto + per-commit comments) was ~150+. ~3× reduction.

## Build plan (vertical slice)

Stage 1 — `/epic-setup` + `/epic-intake`
- `epic-researcher` (gh-CLI rewrite)
- grill protocol skill (unchanged)
- intake.md, grill.md, decisions.md, proposed-epic.md
- index.json + epic-id resolution
- label bootstrap

Stage 2 — `/epic-plan`
- `story-planner` + `story-architect` (with grill-before-spawn)
- `devops-agent` + `data-engineer` (consultative)
- stories-proposal.md
- per-story file
- batch issue creation

Stage 3 — `/epic-claim`
- worktree + branch
- assign + label flip

Stage 4 — `/epic-dispatch` + `/epic-done`
- `story-coder` subagent (background, cwd = worktree)
- TDD gate, 2-retry, commit + PR open
- `/epic-done` PR-merge probe + close

Side commands — `/devops-check`, `/data-search`
- thin wrappers around respective consultative subagents

## Deferred

- `/epic-status` (list epics + phases via `gh issue list --label type:epic`)
- `/epic-unclaim` (reverse claim)
- Multi-repo epics
- GitHub Projects (v2) integration as optional view layer
