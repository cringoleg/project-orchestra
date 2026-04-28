---
description: Story breakdown + per-story DDD code-first plan. Pushes child issues at end.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---

# /epic-plan

No args. Operates on the current epic dir.

## Step 0 — Resolve current epic

Detect epic via cwd context — find newest `<repo-root>/.claude/epics/gh-<N>/` or accept user pointer if multiple. Read `index.json` → `issue_number`.

Phase guard: `gh issue view <N> --json labels | jq -r '.labels[].name'` must include `phase:planning`. Else refuse with hint.

## Step 1 — Pass 1 (story list)

Spawn `story-planner` (pass=1) with prompt:

> Epic dir: `<EPIC_DIR>`. Pass: `1`.
>
> Read `intake.md` + `decisions.md`. Decompose into stories. Soft cap 10.
>
> Output: `<EPIC_DIR>/stories-proposal.md` table — `# | Slug | Goal | Points | Priority | Area | Deps | External Blockers`.
>
> Points: Fibonacci 1/2/3/5/8/13. Priority: highest/high/medium/low/none. Area: devops/data-eng/backend/frontend or empty.

Wait. Print summary.

## Step 2 — User reviews proposal

Tell user: "Read `<path>`. Reply `approve`, `edit`, or `reject`."

If `edit`: user edits file directly, then says continue. Re-read.
If `reject`: re-spawn planner with user notes.
If `approve`: proceed.

## Step 3 — Pass 2 (per-story expand)

For each row in proposal:

1. Spawn `story-planner` (pass=2) with target file path. Fills scope, criteria, deps, etc.
2. Spawn `story-architect` with target file path. Architect:
   - May run `grill-protocol` with user (mid-step!) before spawning `devops-agent` or `data-engineer`. **Always grill before subagent spawn** (project rule).
   - Fills Package Structure, Interfaces, Test Scaffold.
   - May fold `Deploy Notes` / `Data Spec` sections if subagent ran.

Run sequentially per story (architect grilling = interactive).

## Step 4 — Batch review

Print: "All stories drafted. Review files in `<EPIC_DIR>/stories/`. Reply `approve`, `revise <n>`, or `reject all`."

`revise <n>`: respawn planner+architect for that story.
`approve`: proceed.

## Step 5 — Push child issues

For each story file (in order):

1. Read FrontMatter (`slug`, `points`, `priority`, `area`).
2. Build labels: `type:story,state:drafted,points:<n>,priority:<x>` + `area:<y>` if set.
3. Build issue body: copy story file content from `# Goal` through `# Estimate` (skip FrontMatter; skip coder-fills section).
4. Create:
   ```bash
   gh issue create --title "<slug>" --label "<labels>" --body-file <body-tmp>
   ```
5. Capture `<story-issue-N>`. Update story FrontMatter `gh_issue: <N>`. Update `index.json`.
6. If story has `deps:` → after all child issues exist, edit each story body to populate `# Blocked by` section with `#<dep-issue>` refs (`gh issue edit <N> --body-file ...`). If any open dep → add `blocked` label.

## Step 6 — Update epic body + flip phase

Append `## Stories` task-list to `proposed-epic.md`:

```
## Stories
- [ ] #<issue1> <slug1>
- [ ] #<issue2> <slug2>
```

Push:

```bash
gh issue edit <epic-issue-N> --body-file "$EPIC_DIR/proposed-epic.md"
gh issue edit <epic-issue-N> --remove-label phase:planning --add-label phase:dispatch
```

Print: "Phase → dispatch. <count> stories created. Run `/epic-claim <story-id>` to start."

## Notes

- `<story-id>` for downstream commands = local `story_id` (1, 2, ...) per FrontMatter, NOT issue number. Plugin resolves via `index.json`.
- If user re-runs `/epic-plan` (e.g. epic re-opened): refuse if `phase:dispatch` already set. User must manually flip back via `gh issue edit`.
