---
name: quick-task
description: Fast path for a small, self-contained change that doesn't justify the full PRD → kanban → implement pipeline — a cosmetic tweak, a copy/text change, a tiny localized bugfix, moving or restyling one element. Use this skill when the user asks for a single obvious change that touches one or two files with no design decision, no new module/API/schema, and no need for parallel agents — e.g. "move this button", "change the header color", "fix this label's text", "bump the timeout", "tweak the spacing". Escalate to write-prd / prd-to-kanban the moment it turns out bigger than that.
---

# Quick Task (small-change fast path)

The `write-prd → prd-to-kanban → implement-task → review → qa` pipeline exists to make **big, parallelizable** work safe. For a one-line change it's pure overhead. This skill is the short path: do the change well, verify it, commit it — without a PRD, a board, waves, or a review plan.

Its discipline is a single gate: **prove the task is actually small, and bail out the instant it isn't.**

## Eligibility — a task qualifies only if ALL of these hold

- **One obvious change, no design decision.** You know exactly what to do and where; there's no architecture to choose.
- **≤ 1–2 files, localized.** No new module, and no change to a schema, an API, or a shared contract/interface.
- **No parallelism needed.** It's a single unit of work — nothing to split into waves or coordinate file ownership over.
- **Low blast radius / reversible.** Cosmetic, copy, or a small localized fix; not touching auth or other security-sensitive surface.
- **No new dependency, no data migration.**
- **Directly verifiable.** A quick check settles it: the build passes, a test goes green, or the visual result is right.

If **any** item fails, this is not a quick task. Stop and route it to `write-prd` (if it needs scoping) or `prd-to-kanban` (if the PRD already exists). Forcing a real feature through the fast path is exactly the failure this skill guards against.

Examples that qualify: move the "New Task" button to the header's right and use the accent color; change a heading's copy; fix an off-by-one in a single pure helper; adjust a component's padding token. Examples that do **not**: anything touching an API route, the DB, auth, or multiple modules; anything where "where" or "how" isn't already obvious.

## What it skips vs keeps

- **Skips:** the PRD, the board, waves, task files, the file-ownership map, the multi-pass AI `review`, and the human `qa` plan. No `kanban-*/` directory is created.
- **Keeps (the irreducible core):** consult module memory → make the change → verify for real → scoped commit → record any durable lesson. In one line: **memory → change → verify → commit → (maybe) note.**

## Workflow

1. **Check eligibility** against the list above. If any item fails, say so plainly and hand off to the full pipeline — don't proceed here.
2. **Consult module memory.** For the file(s) you'll touch, search `knowledge/INDEX.md` (the `close-cycle` memory). Load any matching entry and respect its decisions, gotchas, patterns, and contracts. *Small does not mean blind* — a one-line edit can trip a recorded footgun.
3. **Make the change**, strictly within the identified file(s). If you discover you must touch another file, change a contract, or make a design call → **STOP and escalate**; the eligibility gate was wrong.
4. **Verify for real.** Run the build/test the change implies, or check the visual result (screenshot / preview). Never report done on a change you didn't actually exercise.
5. **Commit** with a scoped message, e.g. `quick: move New Task button to header right, accent color`.
6. **Record a durable lesson, only if there is one.** If the change revealed a gotcha or settled a decision worth keeping, add a one-line entry to that unit's `knowledge/` memory (or run `close-cycle`). If it taught nothing durable, skip this — don't manufacture memory.
7. **Report:** what changed, how you verified it, and whether anything was recorded.

## Hard rules

- **The eligibility gate is the whole point.** The moment the change needs a second file it shouldn't, a new contract, a schema, an auth touch, or a design decision, STOP and escalate. Do not let a "quick" task quietly grow into an unplanned feature — that's how the fast path becomes a liability.
- **Still consult memory.** Skipping the board never means skipping recorded knowledge for the file you're editing.
- **Verify, don't vibe.** A small change is still verified by execution, not by confidence.
- **Scoped commit, no ceremony.** One focused commit; no PRD, board, or review artifacts — recreating them here defeats the purpose.

## Example

**Input:** "Move the 'New Task' button to the right of the board header and make it use the accent color."

**Behavior:** Confirms eligibility (one component, cosmetic, no contract, no parallelism). Searches `knowledge/INDEX.md`, finds `knowledge/board/board-component.md` — notes the header is a flex row and the accent token is `--accent`. Edits `board.component.html` and `board.component.scss` only. Runs `ng build` (passes) and takes a screenshot showing the button top-right in accent color. Commits `quick: move New Task button to header right, accent color`. Nothing durable learned, so memory is left untouched. Reports: "Done — moved the button, using `--accent`. `ng build` passes; screenshot attached. No new memory recorded. This stayed within one component; if you'd wanted it wired to a new action, that would've been a pipeline task instead."
