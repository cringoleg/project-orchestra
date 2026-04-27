---
story_id: <n>
shortcut_id: ""
slug: <kebab>
state: drafted
points: <fib>
priority: <highest|high|medium|low|none>
deps: []
external_blockers: []
claimed_by: ""
claimed_at: ""
branch: ""
---

# Goal

<one paragraph>

# Acceptance Criteria

(prefer test names; prose only when test name insufficient)

- TestX_doesY
- TestZ_returnsErrOnW

# Affected Packages / Files

- api/internal/<pkg>/...

# Out of Scope

- ...

# Rule files to load

- .claude/rules/<rule>.md

# Package Structure

```
api/internal/<pkg>/
  handlers.go         (new)
  service.go          (new)
  internal/storage.go (new)
```

# Interfaces

```go
// concrete Go interface declarations
```

# Test Scaffold (failing — TDD seed)

```go
// failing unit tests, table-driven where applicable
```

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
