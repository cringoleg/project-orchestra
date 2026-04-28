---
description: Create worktree + branch for a story. Assigns issue, flips state:claimed.
argument-hint: <story-id>
allowed-tools: Read, Edit, Bash
---

# /epic-claim

Arg: `$1` = local `story_id` (1, 2, ...).

## Step 0 — Resolve

Detect current epic dir. Read `<EPIC_DIR>/index.json` → `stories[$1].issue_number` and `slug`. Read story file `<EPIC_DIR>/stories/$1-<slug>.md` FrontMatter.

Refuse if FrontMatter `state` != `drafted`.

## Step 1 — Pull fresh issue state

```bash
gh issue view <story-issue> --json state,assignees,labels
```

Refuse if:
- issue closed
- already assigned to someone else
- label not `state:drafted` (drift — tell user to reconcile)

## Step 2 — Worktree + branch

```bash
WORKTREE="<repo-root>/.claude/worktrees/story-gh-<story-issue>-<slug>"
BRANCH="story/gh-<story-issue>-<slug>"
git fetch origin
git worktree add -b "$BRANCH" "$WORKTREE" origin/<default-branch>
```

(Default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.)

## Step 3 — Push state

```bash
gh issue edit <story-issue> --add-assignee @me
gh issue edit <story-issue> --remove-label state:drafted --add-label state:claimed
```

## Step 4 — Update local story file

Edit FrontMatter:
- `state: claimed`
- `claimed_by: <gh-username>` (from `gh api user -q .login`)
- `claimed_at: <iso8601>`
- `branch: <branch>`
- `last_synced_at: <iso8601>`

## Step 5 — Print

```
Claimed story <id> (#<story-issue>).
Worktree: <path>
Branch: <branch>
Run /epic-dispatch <id> to spawn coder.
```

## Phase guard

If story label not `state:drafted`: refuse with current state in error.
