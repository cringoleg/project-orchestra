---
name: grill-protocol
description: Interview user one question at a time to resolve ambiguity in an epic. Each question includes recommended answer and confidence. Walks the design tree depth-first. Logs Q&A and distills decisions. Use when /epic-intake reaches grill phase, or when user says "grill me on X".
---

# grill-protocol

Interactive interview pattern. Walks design tree depth-first. Resolves dependencies between decisions one at a time.

## Inputs

The caller provides:
- `intake.md` path — research baseline.
- `grill.md` path — append-only Q&A transcript.
- `decisions.md` path — distilled resolved decisions.
- `termination` mode — `hybrid` (you propose done, user confirms).

## Loop rules

1. **One question at a time.** Strict.
2. Each question must include:
   - The question itself, scoped narrow.
   - Sub-bullets if dependent decisions exist.
   - **Recommended answer** with reasoning (1-2 lines).
   - **Confidence**: high / medium / low. Low confidence = expect pushback.
3. Walk the tree:
   - Start with the highest-impact unresolved root question (usually one named in intake.md "Open Questions").
   - As an answer comes in, branch into dependent sub-questions before moving to next root.
   - Skip branches that user marks "park" or "not now".
4. **Mid-grill research**: if you are not certain about a fact (e.g. existing API contract, prior decision), **always ask permission** before fetching. Format:
   > Need to verify <thing> via <Grep|Read|MCP>. OK to fetch?
   Do not fetch silently.
5. **Append every Q + answer to `grill.md`** with timestamp:
   ```
   ## <iso8601>

   Q: <question>
   Recommended: <answer> (confidence: <h|m|l>)
   A: <user response, verbatim>
   ```
6. After each branch resolves, append a **distilled decision** to `decisions.md`:
   ```
   - **<topic>**: <decision>. (<1-line rationale>)
   ```
   Decisions.md must read clean, no Q/A noise. Planner will read this file, not grill.md.

## Termination (hybrid mode)

When you believe all branches resolved:

1. Print:
   ```
   Grill done. Resolved: <count>. Unresolved/parked: <list or "none">.
   Decisions.md ready for /epic-plan. Confirm done or extend?
   ```
2. User says `done` → return control to caller.
   User says anything else → treat as new branch, continue.

## Style

- Caveman terseness. Drop articles, fragments OK.
- Recommended answer always present. Mark confidence honestly.
- No filler ("great question", "interesting"). Just the Q + recommendation.
- If user pushes back on recommendation, accept their answer, write decision, move on. Do not argue twice.

## Anti-patterns

- Asking 3 questions in one message.
- Skipping recommended answer.
- Silently fetching mid-grill.
- Writing prose paragraphs of decision context. One line per decision.
- Asking the same question in different words. If user said "park", park it.
- Declaring "grill done" without listing parked items.
