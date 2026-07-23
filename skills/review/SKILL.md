---
name: review
description: Review everything produced by the PRD → kanban → implementation workflow - verify the implemented code against the PRD requirements and each task's acceptance criteria, check code quality, run tests, and produce a written review report. Use this skill whenever the user asks to review, audit, check, validate, or QA the work, the implementation, the tasks, or the feature — e.g. "review all that was created", "check if everything matches the PRD", "audit the board" — even if they don't say "review" explicitly, such as "is this done correctly?" or "did we miss anything?".
---

# Review

Perform a full review of the work produced by the `write-prd` → `prd-to-kanban` → `implement-task` pipeline: does the implemented code actually satisfy the PRD, do the tasks' claims hold up, and is the code itself sound? Output is a written report file plus a verdict.

This is a **read-and-verify** skill: it inspects, runs, and reports. It does not fix code. If the user wants fixes, that becomes follow-up tasks (offer to add them to the board).

## Workflow

1. **Gather the artifacts.** Locate the PRD (`prd-*.md`), the board (`kanban-*/` with `BOARD.md` and task files), the build gate's `BUILD.md`, and the implementation (the files listed in each task's `files` field, plus `git log` for the `task(...)` commits). If some pieces are missing, review what exists and say what's missing.
2. **Review in three passes** (below): requirements traceability, task verification, code quality.
3. **Reuse the build gate — don't re-run it.** `implement-task` already built, tested, and linted the finished board once, and recorded it in `BUILD.md`. Treat that as this review's execution evidence. Re-run the commands yourself **only** if one of these holds, and say which:
   - `BUILD.md` is missing or the board never closed (then run the gate once yourself and write it);
   - code changed after the gate's commit — check `git log <gate-sha>..HEAD -- <source paths>`, not a hunch;
   - the gate is red (re-running a known-red build tells you nothing new — read its output instead);
   - a specific finding needs a *targeted* command the gate didn't cover (one focused run, e.g. a single e2e path, never the whole suite again).
4. **Write the report** to `kanban-<feature>/REVIEW.md` using the template.
5. **Summarize in chat**: verdict, top findings by severity, and the recommended next step. If the review surfaced durable lessons (a gotcha the tasks hit, a contract worth pinning), note them as candidates for `close-cycle` to record into module memory when the cycle closes.

## Pass 1 — Requirements traceability (PRD ↔ tasks ↔ code)

For every numbered requirement in the PRD (FR-x, NFRs):
- Which task(s) claimed it (`prd_refs`)? Requirements claimed by no task = **gap**.
- Does the implementation actually satisfy it, verified by running something where possible?
- Were the PRD's Non-Goals respected, or did scope creep in?
Also check the reverse direction: code changes not traceable to any task or requirement are flagged as unrequested work.

## Pass 2 — Task verification (board hygiene + honesty)

For every task in `done/`:
- Re-verify each acceptance criterion independently. Checked boxes are claims to be audited, not facts — but audit them **without re-running the build N times**. Criteria the task verified statically get re-checked by reading the code against the pinned contract; criteria marked `pending: build-gate` get settled by `BUILD.md`'s recorded result. If a criterion is checked off and neither the code nor `BUILD.md` supports it, that's a finding: an unverified claim.
- Confirm file ownership was respected: `git log`/diff should show the task's commits touching only its declared `files` (board files aside).
- Confirm the pinned interfaces were implemented exactly as specified in the task's "Interfaces you must conform to" section.
For tasks in `backlog/` or `in-progress/`: list them as unfinished work with their blockers, so the report reflects true completion status.

Also cross-check against **module memory**: for the units touched, load any `knowledge/` memory (from `close-cycle`) and flag changes that violated a recorded decision, broke a "stable" contract, or walked into a documented gotcha — those are findings, not nitpicks.

## Pass 3 — Code quality

Review the implementation as a senior engineer would a pull request:
- Correctness: edge cases, error handling, race conditions, off-by-one risks.
- Security: input validation, injection, authz checks (especially anything the PRD's NFRs called out).
- Tests: do they exist, do they cover the new code, do they test behavior rather than implementation details? Whether they *pass* is already answered by the build gate — read it, don't re-run it. (The gate is what catches "parallel tasks each pass alone and jointly break": it runs on the merged, finished board, which is the only tree where that question is meaningful.)
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
From `BUILD.md` (gate commit `<sha>`): summarize its result. Then state
whether you re-ran anything — if yes, which of step 3's conditions applied
and what the output was; if no, say "reused gate; no code changed since".

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

- **Verify by evidence, not by vibes.** Every "✅" needs a stated verification method that was actually performed — either code read against a pinned contract, or a real command's output. `BUILD.md` counts as that output; "looks fine" never does.
- **The build already ran. Reuse it.** This skill's job is judgment, not re-execution. Re-running the full suite/build to confirm what the gate just recorded is the single biggest waste in the pipeline. Re-run only under the four conditions in step 3, and name the condition in the report.
- **Be honest about severity.** Don't inflate minor style nits into blockers, and don't soften a missed requirement into a "minor". The verdict must follow mechanically: any critical finding or requirement gap ⇒ CHANGES REQUIRED.
- **Don't fix, propose.** Findings become suggested fixes and, on request, new task files — the review itself changes no implementation code.
- **Review the whole, not just the parts.** Independently-built parallel tasks can each be correct and still integrate badly. The gate's suite covers that mechanically; your job is to trace at least one end-to-end path through the feature *by reading it* — request → handler → data → response — and flag seams no test exercises.

## Example

**Input:** "Review all that was created for the CSV export feature."

**Behavior:** Reads `docs/prd-reports-csv-export.md`, `docs/kanban-reports-csv-export/` and its `BUILD.md` (gate green at `a1b2c3d`; `git log a1b2c3d..HEAD` shows no source changes, so nothing is re-run). Audits tasks 101–301 by reading each criterion against the code and against the gate's recorded output, traces the export path end to end, finds FR-4 (row limit) unimplemented and one unauthenticated route — the latter needs a targeted check, so it runs exactly one command (`curl` against the route) rather than the suite. Writes `REVIEW.md` with verdict CHANGES REQUIRED and reports: "2 findings (1 critical, 1 gap). Reused the build gate — only re-ran one targeted authz check. I can add fix tasks 401 and 402 to the board — want me to?"
