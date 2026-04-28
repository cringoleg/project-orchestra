---
agent_type: <backend|frontend|devops|data-eng>
slug: <kebab>
gh_issue: ""
state: drafted
deps: []
last_commit_sha: ""
pushed_at: ""
last_synced_at: ""
---

# Goal

<one paragraph>

# Acceptance Criteria

(prefer test names where applicable; for devops, frontend-no-logic — prose OK)

- TestX_doesY
- TestZ_returnsErrOnW

# Affected Files

- <repo-relative path>

# Out of Scope

- ...

# Rule files to load

- <repo>/.claude/rules/<agent-type>.md
- <repo>/.claude/rules/<topic>.md

# Blocked by

(empty until sub-issues created; populated post-creation as `#N` refs)

# Package Structure

```
<path>/
  <file>            (new)
```

# Interfaces

```
// concrete signatures, language-idiomatic, no pseudo-code
```

# Verification Setup

(test files for backend / data-eng / frontend-w-logic; validator stubs for devops / frontend-no-logic)

```
// test files OR validator commands
```

# Verification Commands

- <lint cmd>
- <test or validator cmd>
- <build cmd if applicable>

# Migration

N/A — <reason>

(or fenced SQL block w/ schema diff)

# Estimate

<small | medium | large>

# --- agent fills below ---

# Status:
# Branch:
# Commit:
# Verification:
# Rules appended:
# Blockers:
