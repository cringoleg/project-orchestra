---
description: Spawn background story-coder subagent in worktree. Flips state:in-progress.
argument-hint: <story-id>
allowed-tools: Read, Edit, Bash, Agent
---

# /epic-dispatch

Arg: `$1` = local `story_id`.

## Step 0 — Resolve

Detect current epic dir. Read `index.json` → `stories[$1]` for `issue_number` + `slug`. Read story file FrontMatter for `branch`, `gh_issue`. Refuse if `state` != `claimed`.

## Step 1 — Push state

```bash
gh issue edit <story-issue> --remove-label state:claimed --add-label state:in-progress
```

Update story FrontMatter `state: in-progress`, `last_synced_at: <iso8601>`.

## Step 2 — Spawn coder

`Agent` invocation, `subagent_type: story-coder`, `run_in_background: true`. Prompt:

> Implement story `gh-<story-issue>` TDD-style.
>
> Inputs:
> - story_file: `<absolute path to .claude/epics/gh-<N>/stories/<id>-<slug>.md>`
> - worktree: `<absolute path>`
> - branch: `<branch-name>`
> - gh_issue: `<story-issue>`
> - rule_files: `<list from story's "Rule files to load" section>`
>
> Pipeline: load context → write failing tests → RED gate → mocks (if needed) → implement → GREEN gate → lint → commit → push branch → `gh pr create --title "<slug>" --body "Closes #<gh_issue>\n\n<commits>"`.
>
> Status: write `state: done` (local-only) and `gh_pr: <pr-num>` to FrontMatter once PR opens.
>
> Manual review gate per project policy: do NOT merge or close issue. `/epic-done` handles post-merge.
>
> Failure handling: 2 retries max. Then `# Blockers:` section + exit cleanly.

## Step 3 — Print

```
Dispatched story <id>. Coder running in background.
Worktree: <path>
Notify on completion. Run /epic-done <id> after you merge the PR.
```

## Phase guard

If story label not `state:claimed`: refuse.
