---
description: Check PR merged for a story. If yes, close issue + flip state:done. If all stories done, close epic.
argument-hint: <story-id>
allowed-tools: Read, Edit, Bash
---

# /epic-done

Arg: `$1` = local `story_id`.

## Step 0 — Resolve

Detect current epic dir. Read `index.json` → `stories[$1]` for `issue_number`. Read story FrontMatter for `gh_pr`.

Refuse if FrontMatter `state` != `in-progress` or local marker indicates blocker.
Refuse if `gh_pr` empty (coder never opened PR — likely blocker; tell user to inspect story file).

## Step 1 — Targeted PR pull

```bash
gh pr view <gh_pr> --json state,mergedAt,mergeCommit
```

Cases:
- `state: MERGED` → proceed.
- `state: OPEN` → tell user "PR #<n> still open. Review + merge first." Exit.
- `state: CLOSED` (not merged) → tell user "PR #<n> closed without merge. Story stuck. Inspect manually." Exit.

## Step 2 — Close issue + flip state

```bash
gh issue close <story-issue> --reason completed
gh issue edit <story-issue> --remove-label state:in-progress --add-label state:done
```

Update story FrontMatter `state: done`, `last_synced_at: <iso8601>`.

## Step 3 — Epic-level rollup

Iterate all stories in `index.json`. For each, check label via `gh issue view <n> --json labels` (one fetch per story — cost = `O(stories)`).

**Optimization**: skip API call for stories already marked `state: done` in local FrontMatter (truth was synced when their `/epic-done` ran).

If all stories `state:done`:

```bash
gh issue edit <epic-issue> --remove-label phase:dispatch --add-label phase:done
gh issue close <epic-issue> --reason completed
```

Print: "Epic gh-<N> done. All stories merged."

Else: print "Story <id> done. <N>/<total> stories complete."

## Phase guard

If story label not `state:in-progress`: refuse with current state.
