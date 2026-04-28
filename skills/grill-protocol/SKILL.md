---
name: grill-protocol
description: Interview user one question at a time to resolve ambiguity. Each Q includes recommended answer + confidence. Walks design tree depth-first. Logs Q&A and distills decisions. Used by main session at any grill gate (research grill, architect grill, blocker resolution).
---

# grill-protocol

Interactive interview pattern. Walks design tree depth-first. Resolves dependencies between decisions one at a time.

## Inputs

The caller provides:
- `intake_path` — research baseline (e.g. `intake.md`, `architect-summary.md`).
- `grill_path` — append-only Q&A transcript path.
- `decisions_path` — distilled resolved decisions path.
- `termination` — `hybrid` (you propose done, user confirms).

## Loop rules

1. **One question at a time.** Strict.
2. Each question must include:
   - The question itself, scoped narrow.
   - Sub-bullets if dependent decisions exist.
   - **Recommended answer** with reasoning (1-2 lines).
   - **Confidence**: high / medium / low. Low confidence = expect pushback.
3. Walk the tree:
   - Start with highest-impact unresolved root question (usually one named in `intake_path`'s `# Open Questions` or `# Open Decisions for Grill`).
   - As an answer comes in, branch into dependent sub-questions before moving to next root.
   - Skip branches user marks `park` or `not now`.
4. **Mid-grill research**: if uncertain about a fact (existing API contract, prior decision), **always ask permission** before fetching:
   > Need to verify <thing> via <Grep|Read>. OK to fetch?
   Do not fetch silently.
5. **Append every Q + answer** to `grill_path` with timestamp:
   ```
   ## <iso8601>

   Q: <question>
   Recommended: <answer> (confidence: <h|m|l>)
   A: <user response, verbatim>
   ```
6. After each branch resolves, append distilled decision to `decisions_path`:
   ```
   - **<topic>**: <decision>. (<1-line rationale>)
   ```
   `decisions_path` must read clean, no Q/A noise. Downstream agents read this file, not `grill_path`.

## Termination (hybrid mode)

When all branches resolved:

1. Print:
   ```
   Grill done. Resolved: <count>. Unresolved/parked: <list or "none">.
   <decisions_path> ready. Confirm done or extend?
   ```
2. User says `done` / `confirm` → return control to caller.
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
- Asking the same question in different words. If user said `park`, park it.
- Declaring "grill done" without listing parked items.

## When called

- After researcher writes `intake.md` (research grill).
- After architect writes `architect-summary.md` + story files (architect grill, holistic).
- On blocker escalation (`phase:blocked`) — single Q to confirm user's resolution path.
- Anywhere main session needs structured Q&A on a design tree.
