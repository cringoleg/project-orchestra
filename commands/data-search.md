---
description: Ad-hoc data-engineer invocation. Searches sources, emits data-spec.
argument-hint: <query>
allowed-tools: Read, Bash, Agent
---

# /data-search

Arg: `$1` = data query (e.g. "ride history last 90 days, ride id + driver id + timestamp + status").

Standalone — no epic required.

## Step 1 — Grill (project rule: grill before subagent spawn)

Run `grill-protocol` skill briefly. One Q at a time. Confirm:
- Sources: local repo only / specific connection string / warehouse / API.
- Prep required: yes (clean/transform/sample) or no (just discover schema).
- Output name: short slug for spec file.

Each Q: recommended answer + confidence.

## Step 2 — Output path

Default: `<repo-root>/.claude/data-specs/<slug>.md`. Create dir if missing. Add `.claude/data-specs/` to `.gitignore` if absent.

## Step 3 — Spawn data-engineer

`Agent` invocation, `subagent_type: data-engineer`, prompt:

> Search + spec data for query: `<query>`.
>
> Inputs:
> - query: `<query>`
> - sources: `<from grill — paths/conn/url or empty>`
> - output_path: `<path>`
> - prep_required: `<true|false>`
>
> Budget: 20 reads, 10 bash.
>
> Output sections: Schema, Sample Rows, Source Access, Prep Notes (if applicable), Adapter Hints, Gaps.
>
> Read-only on remote. Redact PII in samples. No passwords in spec — env-var refs only.

Wait for return. Print path + row_count + count of gaps.

## Out of scope

- No GitHub issue creation.
- No backend adapter code (caller / backend writes adapters from spec).
- No remote writes ever.
