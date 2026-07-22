---
name: close-cycle
description: Close out a finished work cycle by distilling what was learned into a durable, per-module knowledge base — a file-based memory indexed by source file — so future PRDs, tasks, and agents reuse recorded decisions, gotchas, patterns, and contracts instead of re-deriving them. Use this skill at the END of a cycle, after implement-task and/or review, when the user says "close the cycle", "capture what we learned", "update the knowledge base", "record the learnings", "remember this for next time", or "post-mortem this feature". This is different from for-research (upfront, per-big-task, disposable): close-cycle writes permanent memory keyed to code units and grows across cycles.
---

# Close Cycle (durable module memory)

At the end of a cycle, do the one thing the pipeline otherwise throws away: **persist what was learned**. Distill the durable, non-obvious knowledge produced while building — the decisions and their *why*, the gotchas that bit, the local patterns, the interface contracts — into small Markdown files organized by module and unit (Controller, service, component). Later, when any agent is about to touch a file, it looks up what is already known about that unit and reuses it instead of rediscovering it.

Think of it as `for-research` inverted in time: research caches context *before* a big task and can be deleted afterward; close-cycle records lessons *after* the work and keeps them permanently. The two are complementary — close-cycle can **promote** a durable conclusion out of a `research/` chunk or a `REVIEW.md` finding into module memory.

## The memory store

ALWAYS use this layout. Default location: `knowledge/` at the repo root; use `docs/knowledge/` if the repo centralizes docs under `docs/`.

```
knowledge/
├── INDEX.md                    # the retriever: source path → memory file
├── <module>/
│   ├── _module.md              # module-level: architecture, cross-cutting decisions, deps
│   └── <unit>.md               # one file per Controller / service / component
```

- **The tree mirrors the codebase's own module boundaries** (e.g. `features/labels`, `features/board`, `core/`) — do not invent a taxonomy the code doesn't have.
- **The indexing key is the source file path.** Every memory file lists the code files it describes in `source_paths`, and every file appears in `INDEX.md`. This is what makes retrieval mechanical: a task already declares `files:`, so an agent about to touch `F` just searches the index for `F`.
- A memory file may cover a *unit* that spans a few files (an Angular component's `.ts`/`.html`/`.scss`) — list them all in `source_paths`.

## What to record (and what not to)

Record only what is **durable and not obvious from reading the code**:

- **Key decisions — with the why.** "Signals over BehaviorSubject, to match the app-wide convention" — the reasoning a future agent would otherwise relitigate.
- **Gotchas / footguns.** Ordering requirements, quirks, non-obvious coupling, the things that already cost someone an hour.
- **Patterns / conventions to follow.** The local idiom: "all HTTP via `firstValueFrom`", "new endpoints validate with X first".
- **Dependencies & contracts.** What this unit depends on, what depends on it, and which interfaces must stay stable (breaking them breaks consumers).
- **Change log.** Dated, one line per cycle: which PRD/task touched this unit and what changed — the traceability trail.

Do **not** record: a mirror of the code itself (the repo already has it), transient task status, or anything you can't source. If it's not something a future agent would be glad to have been told before editing, leave it out.

## Workflow

### Writing memory (at cycle close)

1. **Gather what happened.** Read the cycle's PRD, the board's `done/` tasks (their `files`, Interfaces, and any `## Blockers`/deviations), the `task(...)` commits and their diffs, and `REVIEW.md` if `review` ran. This is your raw material.
2. **Map touched units.** From the `files` fields and the diff, list every module/unit the cycle changed or created.
3. **Distill, per unit.** For each, extract the *durable* learnings using the categories above. Ask: "what did we figure out here that isn't visible in the final code, and that the next agent to touch this file should know?"
4. **Write or update the memory file.** Create `knowledge/<module>/<unit>.md` if new, or update the existing one: revise decisions/gotchas/patterns/contracts, and **append** a dated change-log entry. Bump `verified`.
5. **Reconcile, don't just append.** If this cycle reversed an earlier decision or retired a gotcha, *edit* it and note the reversal in the change log — memory that only grows becomes wrong. Keep each file tight.
6. **Update `INDEX.md`.** Every memory file must appear, with its `source_paths` and a one-line "knows about..." hook.
7. **Report** to the user: which units gained/updated memory, and the top 2–3 lessons worth their attention.

### Reading memory (retrieval protocol — used by the other skills)

Any skill about to touch code follows this, and `INDEX.md` repeats it verbatim:

1. For each file `F` you're about to create or change, search `INDEX.md` for `F`.
2. If a memory file is listed, load ONLY that file (and anything in its `related`).
3. Apply what it says — respect the decisions, avoid the gotchas, follow the patterns, honor the contracts.
4. Trust it only within its `verified` date: if `source_paths` changed since (`git log --since` on them), re-verify before relying on it.

The skills that MUST consult memory before acting: **`prd-to-kanban`** (fold findings into the task files it writes), **`implement-task`** (inject matching memory into each agent's prompt), **`write-prd`** (respect recorded decisions/constraints while scoping), **`quick-task`** (a one-line change can still trip a known footgun), and **`review`** (verify recorded contracts/decisions were respected).

## INDEX.md template

```markdown
# Module memory — [repo / product name]

Durable, per-unit knowledge captured at the end of each cycle by `close-cycle`.
Keyed by source file, so any skill about to touch a file can find what's known.

## How to use this memory (retrieval protocol)
Before creating or changing a file F:
1. Search the table below for F.
2. If a memory file is listed, load ONLY that file (and its `related`).
3. Respect its decisions, avoid its gotchas, follow its patterns, honor its contracts.
4. Trust it within its `verified` date; if `source_paths` changed since
   (`git log --since`), re-verify first.
Skills that must do this: prd-to-kanban, implement-task, write-prd, quick-task, review.

## Index
| Module | Unit | Memory file | Source paths | Knows about... |
|--------|------|-------------|--------------|----------------|
| labels | LabelService | labels/label-service.md | client/.../labels/label.service.ts | signals convention, slug ids, immutable updates |
| labels | (module) | labels/_module.md | client/.../features/labels/** | label-catalog architecture, backend contract |
```

## Memory file template

```markdown
---
module: labels
unit: LabelService          # Controller / service / component / …; omit for _module.md
source_paths:
  - client/src/app/features/labels/label.service.ts
verified: 2026-07-21
related: [knowledge/labels/_module.md]
---

# LabelService — module memory

## Role
1–2 lines: what this unit is responsible for.

## Key decisions (why, not just what)
- Signals-based state (`_labels` writable, `labels` readonly) — chosen over an
  RxJS `BehaviorSubject` to match the app-wide signal convention. Don't
  reintroduce subjects here.

## Gotchas / footguns
- `id` is a server-generated slug of `name`; never construct it client-side.
- Mutations MUST update `_labels` immutably, or the board's computed signals
  won't refresh.

## Patterns / conventions to follow
- All HTTP via `firstValueFrom`; base URL from `environment.apiBaseUrl`.
- create/update return the affected entity; delete returns void.

## Dependencies & contracts
- Depends on: backend `/api/labels` CRUD (see knowledge/backend/labels-api.md).
- Depended on by: Settings Labels UI, task-card label resolution.
- STABLE contract: public surface `load / create / update / delete` — changing
  it breaks consumers.

## Change log
- 2026-07-21 (cycle: board-customization, task 202): added create/update/delete.
- 2026-06-30 (cycle: sprout, task 303): initial signals-based read service.
```

## Hard rules

- **Durable and non-obvious only.** If a fact is visible by reading the code, it doesn't belong here. Memory stores *why*, *watch-out*, and *contract* — not *what*.
- **Keyed by source path, always indexed.** Every memory file lists `source_paths` and appears in `INDEX.md`. Memory an agent can't find by file path is dead weight.
- **One unit per file; module-wide facts in `_module.md`.** Keep each file about one Controller/service/component so it pastes cleanly into a prompt.
- **Reconcile, don't accumulate.** Update reversed decisions and retire stale gotchas; note the change in the log. Append-only memory rots into misinformation.
- **Staleness is managed, not ignored.** `verified` dates + `source_paths` make cheap re-verification possible; bump dates only on files you actually touched.
- **Don't duplicate `research/`.** A `research/` cache is task-scoped and disposable; promote only its *durable* conclusions into `knowledge/`, and link rather than copy the rest.

## Example

**Input:** "We finished the board-customization cycle — close it out and capture what we learned."

**Behavior:** Reads `prd-board-customization.md`, the board's `done/` tasks and their commits, and `REVIEW.md`. Identifies the units touched (LabelService, the columns API, the Settings panel). Writes `knowledge/labels/label-service.md` (decision: signals over subjects; gotcha: slug ids are server-owned; stable contract: the CRUD surface), updates `knowledge/backend/columns-api.md`, and appends dated change-log lines to each. Updates `knowledge/INDEX.md` with three rows. Reports: "Recorded memory for 3 units. Biggest lesson for next time: label `id`s are server-generated slugs — the earlier attempt to build them client-side caused the 202 rework. `prd-to-kanban` will now fold that into any future task touching `label.service.ts`."
