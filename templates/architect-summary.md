---
epic_id: gh-<N>
generated_at: <iso8601>
agents_picked: [<list of agent_types>]
verify_lint: <inline cmd list>
verify_test: <inline cmd list>
verify_build: <inline cmd list>
---

# Plan Overview

<one paragraph: what gets built, in what order, why this slicing.>

# Stories (topo order)

- `data-eng`: <one-line goal>. Deps: <none|list>.
- `backend`: <one-line goal>. Deps: data-eng.
- `frontend`: <one-line goal>. Deps: backend.
- `devops`: <one-line goal>. Deps: backend, frontend.

(Only picked agent_types listed. Subset of {backend, frontend, devops, data-eng}.)

# Verify

- lint: <cmd list>
- test: <cmd list>
- build: <cmd list>

# Open Decisions for Grill

- <ambiguity 1>
- <ambiguity 2>

# Cycles

(empty if none; else list cycles + suggested break point)
