# epic-flow

GitHub-only epic-to-story pipeline plugin. Push-only, no sync.

## Setup

Run once per repo:

```
/epic-setup
```

Creates labels (`type:*`, `phase:*`, `state:*`, `points:*`, `priority:*`, `area:*`, `blocked`) via `gh label create`.

## Commands

- `/epic-intake <issue-url|#N|title>` — research + grill. Auto-creates GitHub issue if title.
- `/epic-plan` — story breakdown + DDD code-first plan. Pushes child issues at end.
- `/epic-claim <story-id>` — worktree + branch. Assigns issue, flips `state:claimed`.
- `/epic-dispatch <story-id>` — background `story-coder` subagent. Flips `state:in-progress`.
- `/epic-done <story-id>` — checks PR merged, closes issue, flips `state:done`.
- `/devops-check <topic>` — ad-hoc consult on local + cloud deploy state.
- `/data-search <query>` — ad-hoc data discovery + spec emission.

## State

GitHub = truth for public artifacts (issue body, labels, comments, child issues).
Local `<repo>/.claude/epics/<gh-N>/` = narrative (intake.md, grill.md, decisions.md) + slim `index.json`.

`epic-id` = `gh-<n>` always. Auto-created at intake.

## External tools

- `gh` CLI required.
- Devops-agent may invoke infra tools (docker, kubectl, terraform, cloud SDKs) per repo.
- No Shortcut, no Notion, no MCP.

## See

[DESIGN.md](DESIGN.md) for full design.
