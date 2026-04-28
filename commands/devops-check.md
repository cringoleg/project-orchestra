---
description: Ad-hoc devops-agent invocation. Inspects local + cloud deploy state for a topic. Returns notes file.
argument-hint: <topic>
allowed-tools: Read, Bash, Agent
---

# /devops-check

Arg: `$1` = topic / scope (e.g. "compose stack for api/", "k8s rollout for web", "GH Actions for release").

Standalone — no epic required.

## Step 1 — Grill (project rule: grill before subagent spawn)

Run `grill-protocol` skill briefly. One Q at a time. Confirm:
- Mode: `inspect` (default, read-only) or `apply` (edits configs).
- Scope: directory / service / file.
- Specific cloud target if relevant (none = auto-detect).

Each Q: recommended answer + confidence.

## Step 2 — Output path

Default: `<repo-root>/.claude/devops-checks/<iso8601>-<slug>.md`. Create dir if missing. Add `.claude/devops-checks/` to `.gitignore` if absent.

## Step 3 — Spawn devops-agent

`Agent` invocation, `subagent_type: devops-agent`, prompt:

> Inspect deploy state for: `<topic>`.
>
> Inputs:
> - scope: `<scope from grill>`
> - mode: `<inspect|apply>`
> - output_path: `<path>`
> - targets: `<list or empty>`
>
> Budget: 10 reads, 5 bash.
>
> Output sections: Stack, Required Env / Secrets, Build / Run Commands, Deploy Targets, Gaps / Risks. (Edits Applied if apply mode.)

Wait for return. Print path + Stack line + count of gaps.

## Out of scope

- No GitHub issue creation (not part of epic flow).
- No edit unless `mode: apply` confirmed in grill.
