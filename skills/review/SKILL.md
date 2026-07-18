---
name: review
description: Review everything produced by the PRD → kanban → implementation workflow - verify the implemented code against the PRD requirements and each task's acceptance criteria, check code quality, run tests, and produce a written review report. Use this skill whenever the user asks to review, audit, check, validate, or QA the work, the implementation, the tasks, or the feature — e.g. "review all that was created", "check if everything matches the PRD", "audit the board" — even if they don't say "review" explicitly, such as "is this done correctly?" or "did we miss anything?".
---

# Review

Perform a full review of the work produced by the `write-prd` → `prd-to-kanban` → `implement-task` pipeline: does the implemented code actually satisfy the PRD, do the tasks' claims hold up, and is the code itself sound? Output is a written report file plus a verdict.

This is a **read-and-verify** skill: it inspects, runs, and reports. It does not fix code. If the user wants fixes, that becomes follow-up tasks (offer to add them to the board).

## Workflow

1. **Gather the artifacts.** Locate the PRD (`prd-*.md`), the board (`kanban-*/` with `BOARD.md` and task files), and the implementation (the files listed in each task's `files` field, plus `git log` for the `task(...)` commits). If some pieces are missing, review what exists and say what's missing.
2. **Review in three passes** (below): requirements traceability, task verification, code quality.
3. **Run the verification commands** — test suites, builds, linters — and record actual output. Never mark something verified based only on reading the code; if a criterion says "test passes", run the test.
4. **Write the report** to `kanban-<feature>/REVIEW.md` using the template.
5. **Summarize in chat**: verdict, top findings by severity, and the recommended next step.

## Pass 1 — Requirements traceability (PRD ↔ tasks ↔ code)

For every numbered requirement in the PRD (FR-x, NFRs):
- Which task(s) claimed it (`prd_refs`)? Requirements claimed by no task = **gap**.
- Does the implementation actually satisfy it, verified by running something where possible?
- Were the PRD's Non-Goals respected, or did scope creep in?
Also check the reverse direction: code changes not traceable to any task or requirement are flagged as unrequested work.

## Pass 2 — Task verification (board hygiene + honesty)

For every task in `done/`:
- Re-verify each acceptance criterion independently — actually run the command, click path, or assertion it implies. Checked boxes are claims to be audited, not facts.
- Confirm file ownership was respected: `git log`/diff should show the task's commits touching only its declared `files` (board files aside).
- Confirm the pinned interfaces were implemented exactly as specified in the task's "Interfaces you must conform to" section.
For tasks in `backlog/` or `in-progress/`: list them as unfinished work with their blockers, so the report reflects true completion status.

## Pass 3 — Code quality

Review the implementation as a senior engineer would a pull request:
- Correctness: edge cases, error handling, race conditions, off-by-one risks.
- Security: input validation, injection, authz checks (especially anything the PRD's NFRs called out).
- Tests: do they exist, do they pass, do they test behavior rather than implementation details? Run the full suite, not just task-local tests — parallel tasks can individually pass and jointly break.
- Consistency: naming, patterns, and style consistent with the surrounding codebase.
- Simplicity: flag over-engineering and dead code.

Report findings with severity: **critical** (broken/insecure/requirement unmet), **major** (works but shouldn't ship as-is), **minor** (style, polish, nice-to-have).

## REVIEW.md template

ALWAYS use this exact structure:

```markdown
# Review – [Feature Name]

Date: YYYY-MM-DD
PRD: [link]  ·  Board: [link]  ·  Commits reviewed: <range>

## Verdict
APPROVED | APPROVED WITH MINOR ISSUES | CHANGES REQUIRED
One-paragraph justification.

## Requirements traceability
| Req | Task(s) | Implemented | Verified how | Status |
|-----|---------|-------------|--------------|--------|
| FR-1 | 101, 201 | yes | ran `npm test csv` – pass | ✅ |
| FR-3 | – | no | – | ❌ gap |

## Task audit
| Task | Column | Criteria re-verified | File ownership | Notes |
|------|--------|----------------------|----------------|-------|
| 101 | done | 3/3 pass | clean | – |

## Test & build results
Actual commands run and their real output (summarized).

## Findings
### Critical
- [C1] file:line – description, why it matters, suggested fix.
### Major
- [M1] ...
### Minor
- [m1] ...

## Unrequested changes detected
Code changes with no corresponding task/requirement, if any.

## Recommended next steps
Ordered list. If fixes are needed, propose them as new board tasks
(id, title, wave) ready to hand to `prd-to-kanban`/`implement-task`.
```

## Hard rules

- **Verify by execution, not by vibes.** Every "✅" in the traceability and audit tables needs a stated verification method that was actually performed.
- **Be honest about severity.** Don't inflate minor style nits into blockers, and don't soften a missed requirement into a "minor". The verdict must follow mechanically: any critical finding or requirement gap ⇒ CHANGES REQUIRED.
- **Don't fix, propose.** Findings become suggested fixes and, on request, new task files — the review itself changes no implementation code.
- **Review the whole, not just the parts.** Independently-built parallel tasks can each be correct and still integrate badly; always run the full test suite and exercise at least one end-to-end path through the feature.

## Example

**Input:** "Review all that was created for the CSV export feature."

**Behavior:** Reads `docs/prd-reports-csv-export.md` and `docs/kanban-reports-csv-export/`, re-runs the acceptance checks of tasks 101–301, runs the full test suite and an end-to-end export, finds FR-4 (row limit) unimplemented and one unauthenticated route (critical), writes `REVIEW.md` with verdict CHANGES REQUIRED, and reports in chat: "2 findings (1 critical, 1 gap). I can add fix tasks 401 and 402 to the board — want me to?"
