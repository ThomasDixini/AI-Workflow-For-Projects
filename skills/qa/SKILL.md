---
name: qa
description: Generate a QA and code-review plan for a HUMAN to manually validate everything that was created in the PRD → kanban → implementation workflow, and to check that the code follows development best practices. Use this skill whenever the user asks for a QA plan, test plan, manual testing checklist, code-review checklist, validation plan, acceptance testing guide, or asks "how should I/my team verify this?" — even without the word "QA". This is different from the automated `review` skill: `review` is the AI verifying; `qa` produces a plan for humans to verify.
---

# QA

Produce a QA + code-review plan that a **human** (the user, a teammate, or a reviewer) can follow step by step to validate the feature and judge whether the code meets development best practices. The AI writes the plan; the human executes it and records results directly in the document.

The output is one file: `kanban-<feature>/QA-PLAN.md` (or `docs/QA-PLAN-<feature>.md` if there is no board). It must be executable by someone who did **not** build the feature: every step says exactly what to do, what to look at, and what "pass" looks like.

## Workflow

1. **Gather context.** Read the PRD, the board and task files, `REVIEW.md` if the `review` skill already ran (its findings become targeted QA items), and the implementation itself (to write accurate steps — real URLs, commands, file paths, seed data).
2. **Derive test scenarios** from the PRD's functional requirements and user stories — happy paths, edge cases, and failure paths. Every FR must appear in at least one manual test case.
3. **Derive the code-review checklist** from the actual code: point reviewers at the specific files/PRs/commits worth human eyes, with concrete questions per area.
4. **Tailor the best-practices audit** to the project's stack (detect it: language, framework, test runner, linter config) rather than generic advice.
5. **Write `QA-PLAN.md`** with the template below and tell the user where it is, how long execution should take, and which sections need which skill set (anyone vs. a developer).

This skill produces the plan only — it does not execute the tests (humans do) and does not fix anything.

## QA-PLAN.md template

ALWAYS use this exact structure:

```markdown
# QA & Code Review Plan – [Feature Name]

PRD: [link] · Board: [link] · AI review: [link to REVIEW.md, if any]
Estimated effort: ~X min manual QA (anyone) + ~Y min code review (developer)
Environment setup: exact commands to get a runnable state
(install, seed data, env vars, how to log in as each user role).

## Part 1 — Manual QA (no coding skills required)

### How to record results
Mark each step ✅ / ❌ and write what you saw in "Actual". Any ❌ = file it
in Part 4.

### TC-1: [Scenario name] (covers FR-1)
Preconditions: ...
| # | Step (exact action) | Expected result | Actual | ✅/❌ |
|---|---------------------|-----------------|--------|------|
| 1 | Go to /reports and click "Export CSV" | A file report.csv downloads | | |

(One TC per scenario. Include: happy paths, edge cases — empty data, huge
data, special characters —, failure paths — no permission, network error —,
and explicit checks of the PRD's Non-Functional Requirements: performance
feel, accessibility basics like keyboard navigation, mobile/responsive if
applicable.)

## Part 2 — Code review checklist (developer)

Where to look: list of key files/commits with one line on what each does.

For each area, concrete yes/no questions, e.g.:
### Correctness & error handling
- [ ] `src/lib/csv.ts`: are quotes/commas/newlines in cell values escaped?
- [ ] What happens when the report has zero rows? Is it handled?
### Security
- [ ] Is the export route behind the same authz check as the page?
- [ ] Any user input reaching queries/paths unvalidated?
### Tests
- [ ] Run `<real test command>` — all green? Coverage of the new code?
- [ ] Do tests assert behavior (output) rather than implementation?
### Readability & maintainability
- [ ] Would a new team member understand `<key file>` without help?
- [ ] Naming, dead code, TODOs left behind?

## Part 3 — Best-practices audit (stack-specific)

Checklist tailored to THIS project's stack and conventions, each item with
a command or file to check, e.g. linter passes (`<real lint command>`),
types check, commit history is task-scoped, no secrets committed
(`git log -p | grep -i ...` guidance), dependency additions justified,
error messages user-friendly, logging appropriate, docs/README updated.

## Part 4 — Findings log
| ID | Where found (TC / file) | Severity (critical/major/minor) | Description | Suggested owner |
|----|------------------------|--------------------------------|-------------|-----------------|

## Part 5 — Sign-off
- [ ] All test cases executed; failures logged above
- [ ] Code review checklist completed
- [ ] Best-practices audit completed
Verdict: SHIP / SHIP AFTER MINOR FIXES / DO NOT SHIP
Reviewer: ______  Date: ______
```

## Writing guidelines

- **Steps must be reproducible verbatim.** "Test the export" is not a step; "Log in as viewer@test.com / go to /reports / click Export CSV" is. Use real routes, commands, and seed data from the actual project — read the code to get them right.
- **Separate audiences.** Part 1 must be executable by a non-developer; Parts 2–3 assume a developer. Say so.
- **Convert AI-review findings into targeted checks.** If `REVIEW.md` flagged something as fixed or risky, add a manual test case that specifically re-validates it — humans double-check exactly where the AI had doubts.
- **Best practices means the project's practices first.** Match the existing lint/format/test conventions before reaching for generic industry advice; only add generic items (OWASP-style security basics, accessibility) where the project has no stated convention.
- **Right-size the plan.** A small feature deserves a 15-minute plan, not a 40-page one. Aim for the smallest set of checks that would catch what matters.

## Example

**Input:** "Create a QA plan for the CSV export feature."

**Output:** `docs/kanban-reports-csv-export/QA-PLAN.md` with 7 manual test cases (including "export with 0 rows", "cell containing commas and quotes", "user without permission gets 403"), a code-review checklist pointing at `csv.ts`, the API route and its tests, a stack-specific audit (`npm run lint`, `npm run typecheck`, task-scoped commits), findings log, and sign-off block. Chat summary: "~20 min for QA (anyone) + ~25 min code review (developer). TC-5 specifically re-checks the authz issue the AI review flagged."
