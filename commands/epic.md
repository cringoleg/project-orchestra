---
description: Entry + resume for full epic pipeline. Auto-chains research → architect → implementers → finalize → PR. Resumable on session death.
argument-hint: [<feature-desc>|#N|<issue-url>]  (no arg = resume)
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---

# /epic

Single command for the whole pipeline. Resumes on re-invoke if active epic exists.

Argument forms:

- **Free-text** — create new epic. Title = arg.
- **`#N`** or **issue URL** — fetch existing epic issue, treat as fresh-from-existing.
- **No arg, run from inside repo with active `<repo>/.claude/epics/gh-<N>/`** — resume current phase.

Requires `/epic-setup` already run in this repo. Refuse if labels missing.

## Step 0 — Preflight

```bash
gh repo view --json nameWithOwner -q .nameWithOwner       # confirm repo + auth
gh label list --limit 200 | grep -c '^phase:research'      # confirm /epic-setup ran
```

If labels missing: print "Run `/epic-setup` first." Exit.

## Step 1 — Resolve arg → epic context

Detect arg shape:

- **URL** matching `https://github.com/<owner>/<repo>/issues/<N>` → `gh_issue=N`. Existing.
- **`#?\d+`** → `gh_issue=<N>`. Existing.
- **Free text** → create new issue:
  ```bash
  gh issue create --title "$1" --label type:epic,phase:research --body "Draft — populated by /epic"
  ```
  Capture `<N>` from response URL.
- **No arg** → find active epic dir under `<repo-root>/.claude/epics/`. If multiple → list, ask user. If none → print usage.

`epic-id = gh-<N>`. `EPIC_DIR = <repo-root>/.claude/epics/gh-<N>`.

## Step 2 — Single-active-epic guard (only on new epic)

```bash
gh issue list --label type:epic --state open --json number,labels | \
  jq '[.[] | select(.labels[].name | startswith("phase:") and . != "phase:done")] | length'
```

If count > 0 and current isn't one of them: refuse. Print existing open epic + offer `abort` or `resume <N>`.

## Step 3 — Init or resume

If `$EPIC_DIR/` absent: create it. Write seed `index.json` from `templates/index.json`.

If present: read `index.json` + `gh issue view <N> --json labels`. Reconcile:

- If GH `phase:*` ≠ local `phase` → trust GH, rewrite local `phase`.
- If `gate_pending` set → re-print gate prompt (Step 4 onwards).
- If `active_story` set + no PR + no commit on branch in last 10 min → orphan implementer; re-spawn at Step 6.

## Step 4 — Phase: research

If `intake.md` absent:

Spawn `researcher` (foreground subagent):

> Research epic.
>
> Inputs:
> - epic_id: `gh-<N>`
> - gh_issue_number: `<N>`
> - source_arg: `$1`
> - output_path: `$EPIC_DIR/intake.md`
> - rule_files: `<repo>/AGENTS.md`, `<repo>/CLAUDE.md` (if exist)
>
> Budget: 15 reads, 10 gh fetches, 5 web fetches.
>
> Sections in `intake.md`: *Summary, Linked Issues / PRs, Affected Packages, Best Practices, Open Questions (5–10), Suspected Blockers*.

Wait for return. Print path + 5-line summary.

Initialize grill artifacts if missing:
- `$EPIC_DIR/grill.md` — `# Grill log\n\n`
- `$EPIC_DIR/decisions.md` — `# Decisions\n\n`

Invoke skill `grill-protocol` with:
- intake_path: `$EPIC_DIR/intake.md`
- grill_path: `$EPIC_DIR/grill.md`
- decisions_path: `$EPIC_DIR/decisions.md`
- termination: hybrid

On grill confirm:

1. Distill ≤3 project-wide rules from `decisions.md` into `<repo>/AGENTS.md` (append, dedup by line semantics).
2. Write `$EPIC_DIR/proposed-epic.md` from `templates/proposed-epic.md`. Sections: Goal, Scope, Out of Scope, Acceptance Criteria, Open Dependencies / Blockers. (`## Stories` left empty until architect phase.)
3. Print path + summary. Ask: "Reply `confirm` to push to GH issue + advance, `edit` to revise."
4. On `confirm`:
   ```bash
   gh issue edit <N> --body-file "$EPIC_DIR/proposed-epic.md"
   gh issue edit <N> --remove-label phase:research --add-label phase:architect
   ```
   Update `index.json.phase = "architect"`, `last_transition`. Continue to Step 5.
5. On `edit`: print "Edit `$EPIC_DIR/proposed-epic.md` then reply `continue`." Yield.

## Step 5 — Phase: architect

If no story files in `$EPIC_DIR/stories/`:

Spawn `architect` (foreground subagent):

> Plan epic skeleton.
>
> Inputs:
> - epic_dir: `$EPIC_DIR`
> - intake_path: `$EPIC_DIR/intake.md`
> - decisions_path: `$EPIC_DIR/decisions.md`
> - agents_md_path: `<repo>/AGENTS.md`
> - rules_dir: `<repo>/.claude/rules/`
> - output_summary: `$EPIC_DIR/architect-summary.md`
> - story_dir: `$EPIC_DIR/stories/`
> - state_file: `$EPIC_DIR/index.json`
>
> Pick subset of {backend, frontend, devops, data-eng} needed. Max 1 of each. Write one `<agent_type>.md` story file per pick (template: `templates/story.md`). Topo-sort by deps; write ordered list to `index.json.stories`. Detect verify cmds → write `index.json.verify`.

Wait. Print summary + path.

Invoke skill `grill-protocol` (holistic — single grill across entire plan):
- intake_path: `$EPIC_DIR/architect-summary.md`
- grill_path: `$EPIC_DIR/grill.md` (append; same file)
- decisions_path: `$EPIC_DIR/decisions.md` (append; same file)
- termination: hybrid

User can `revise <agent-type>` mid-grill — rewrite that one story file then re-grill that branch.

On grill confirm:

1. Distill ≤3 project-wide rules from new `decisions.md` deltas → append `<repo>/AGENTS.md`.
2. Batch-create sub-issues. For each story (in topo order):
   ```bash
   gh issue create \
     --title "<epic-slug>: <agent-type> — <story-slug>" \
     --label "type:story,state:drafted,agent:<agent-type>" \
     --body-file <body-tmp>
   ```
   Body shape (lean): Goal, Acceptance Criteria, Affected Files, Out of Scope, Verification Commands, Blocked by. Skip Package Structure / Interfaces / Verification Setup (those stay local to story file).
   Capture `<sub-issue-N>`. Update story FrontMatter `gh_issue: <N>`. Update `index.json.stories[i].issue_number`.
3. Update epic body: append `## Stories` task list (`- [ ] #<N> <slug> (agent:<type>)`). Push:
   ```bash
   gh issue edit <epic-issue> --body-file "$EPIC_DIR/proposed-epic.md"
   ```
4. Flip phase:
   ```bash
   gh issue edit <epic-issue> --remove-label phase:architect --add-label phase:implementing
   ```
5. Update `index.json.phase = "implementing"`, `last_transition`.
6. Create worktree + branch:
   ```bash
   DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
   git fetch origin
   git worktree add -b "epic/gh-<N>-<slug>" \
       "<repo>/.claude/worktrees/epic-gh-<N>-<slug>" \
       "origin/$DEFAULT_BRANCH"
   ```

## Step 6 — Phase: implementing

For each story in `index.json.stories` (in topo order):

If `story.state == "done"` → skip.

Update labels + state:

```bash
gh issue edit <story-issue> --remove-label state:drafted --add-label state:in-progress --add-assignee @me
```

Set `index.json.active_story = <agent-type>`, story FrontMatter `state: in-progress`, write `started_at`.

Spawn corresponding agent (`backend` / `frontend` / `devops` / `data-eng`) **background**:

> Implement story.
>
> Inputs:
> - story_file: `$EPIC_DIR/stories/<agent-type>.md` (absolute)
> - worktree: `<repo>/.claude/worktrees/epic-gh-<N>-<slug>` (absolute, cwd)
> - branch: `epic/gh-<N>-<slug>`
> - agent_type: `<agent-type>`
> - gh_issue: `<sub-issue-N>`
> - rule_files: <list from story FrontMatter>
> - own_rule_file: `<repo>/.claude/rules/<agent-type>.md`
>
> Pipeline per agent definition.

Monitor for completion. On agent return:

- **Success**: agent committed + appended its rule file. Update `gh issue edit <story-issue> --remove-label state:in-progress --add-label state:done`. Update `index.json.stories[i].state = "done"`, story FrontMatter `state: done`. Continue to next story.
- **Failure (incident)**: read story's `# Blockers` section. Increment incident counter. If counter ≤ 2 → re-spawn same agent w/ failure context appended. If > 2 → escalate (Step 8).

Mid-implement abort: user types `abort` → kill subagent → set story `state:blocked`, epic `phase:blocked`, write `gate_pending`, yield.

When all stories `state:done`:

Flip phase:
```bash
gh issue edit <epic-issue> --remove-label phase:implementing --add-label phase:finalize
```
Update `index.json.phase = "finalize"`. Continue to Step 7.

## Step 7 — Phase: finalize

Run integration verify in worktree, in order:

```bash
cd <worktree>
for cmd in $(jq -r '.verify.lint[]'  <epic-dir>/index.json); do $cmd || FAIL=lint; done
for cmd in $(jq -r '.verify.test[]'  <epic-dir>/index.json); do $cmd || FAIL=test; done
for cmd in $(jq -r '.verify.build[]' <epic-dir>/index.json); do $cmd || FAIL=build; done
```

(Pseudo. Real impl iterates stages, captures stdout+stderr, stops on first fail.)

**On fail**:

1. Extract candidate file paths from stderr via regex (Go: `*.go:LINE`, TS/JS: `at *.ts:LINE`, Python: `File "*.py", line N`, generic: `<path>:N:M`).
2. Cross-ref each path against each story's `# Affected Files` in FrontMatter.
3. Single match → re-dispatch that story's agent. Append last 80 lines of failure to story `# Blockers`. Increment counter. Counter > 2 → escalate.
4. Multi match → escalate. Print: "Build failed touching files in stories <a>, <b>. Reply `retry <a>` / `retry <b>` / `edit`."
5. No match → escalate. Print: "Build failed in unowned files: <list>. Reply `retry all` / `edit` / `abort`."

**On all green**:

```bash
git push -u origin epic/gh-<N>-<slug>
gh pr create \
  --title "<epic-slug>" \
  --body-file <pr-body-tmp>
```

PR body template (`templates/proposed-epic.md`-derived; see DESIGN.md PR shape).

Capture `<pr-num>`. Update `index.json.pr_number = <pr-num>`. Flip phase:
```bash
gh issue edit <epic-issue> --remove-label phase:finalize --add-label phase:review
```

Print: "PR #<pr-num> opened. Review + merge to advance."

## Step 8 — Phase: review

```bash
gh pr view <pr-num> --json state,mergedAt
```

Cases:
- `MERGED` → close epic + sub-issues:
  ```bash
  for n in <epic-N> <sub-issues...>; do gh issue close $n --reason completed; done
  gh issue edit <epic-issue> --remove-label phase:review --add-label phase:done
  ```
  Update `index.json.phase = "done"`. Cleanup per matrix (Step 10).
- `OPEN` → "PR #<n> still open. Merge to advance."
- `CLOSED` (not merged) → "PR #<n> closed without merge. Reply `retry` / `abort`."

## Step 9 — Escalation (`phase:blocked`)

On any retry-exhausted failure:

1. Set `phase:blocked` on epic. Set `state:blocked` on offending story.
2. Append failure context to story's `# Blockers`.
3. Set `index.json.gate_pending = { story_id, reason, last_error_tail }`.
4. Print blocker context to user.
5. Wait for verb:
   - `continue` — re-check current state. If green → advance. Else re-print.
   - `retry` — re-spawn agent w/ fresh story. Reset incident counter.
   - `edit` — print files to edit. User replies `continue` / `retry` / `abort` after.
   - `skip` — mark story `state:done` manually. Warn first: "Story not implemented. Future stories may break. Type `skip-confirm` to proceed."
   - `abort` — exit cleanly. Worktree preserved.

## Step 10 — Cleanup matrix

| Terminal | Local epic dir | Worktree | Branch |
|---|---|---|---|
| `phase:done` | delete (default) / archive on user `archive` | `git worktree remove` | keep |
| `phase:blocked + abort` | keep | `git worktree remove` | keep |

## Notes

- All paths in this command absolute. No relative path traps.
- `gh` CLI is the only network tool. No curl + token.
- `date -u +%Y-%m-%dT%H:%M:%SZ` for timestamps.
- Resume detection in Step 3 is the only place we check for orphans (>10 min stale active_story).
- Phase guard: each phase block above checks `index.json.phase` matches before acting. Mismatch → reconcile + re-route.
