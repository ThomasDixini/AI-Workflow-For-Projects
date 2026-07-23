---
name: implement-task
description: Execute tasks from a file-based kanban board (created by the prd-to-kanban skill), using one or more agents in parallel to implement them. Use this skill whenever the user asks to implement, work on, resolve, execute, or "run" tasks from the board or backlog — e.g. "implement wave 1", "work on task 201", "resolve all the tasks", "start the kanban" — even if they don't name the board explicitly. Also use it when the user asks to continue implementation of a previously started board.
---

# Implement Task

Execute tasks from a kanban board produced by the `prd-to-kanban` skill. Tasks within a wave are designed to be independent (no shared files, pinned interfaces), so run them **in parallel with subagents whenever subagents are available**; otherwise run them sequentially yourself, one at a time.

## The build rule (read this first)

**The project is built exactly once per board, at the very end — never per task, never per wave, never per agent.**

Builds, typechecks, and full test suites are the most expensive thing in this pipeline and the parallelism makes it worse: N agents running `build` in the same working tree fight over the same `dist/` · `.next/` · `.angular/cache` · `node_modules/.cache`, so N concurrent builds are not just N× slower — they're flaky. And a per-task build proves almost nothing anyway: the task's own wave-mates aren't finished, so the tree it compiles isn't the tree that ships.

So execution-based verification is **deferred and batched**: agents verify statically, park what needs running as `pending: build-gate`, and one build gate at the end (step 8) settles all of it at once.

## Workflow

1. **Locate the board.** Find the `kanban-*/` directory (usually under `docs/` or `tasks/`). If several boards exist, ask which one. Read `BOARD.md` for the execution plan, shared interfaces, and file-ownership map.
2. **Select tasks.** 
   - If the user named specific tasks or a wave, do those.
   - Otherwise, take the lowest incomplete wave: all tasks in `backlog/` belonging to it.
   - Skip tasks with `agent_ready: false`; report them to the user with their open questions instead of guessing.
   - Never start a wave while the previous wave has tasks outside `done/`.
3. **Start tasks.** For each selected task: `git mv` its file from `backlog/` to `in-progress/` and set `status: in-progress` in its frontmatter. Commit this board update before implementation starts so the board reflects reality.
4. **Execute.**
   - **Load module memory first.** For each task, for the files in its `files` field, check `knowledge/INDEX.md` (the `close-cycle` memory) and load any matching entry. `prd-to-kanban` should already have folded the relevant knowledge into the task file, but pass the memory to the agent too as a belt-and-suspenders — it must respect recorded decisions, gotchas, and contracts for the units it touches.
   - **With subagents:** spawn one subagent per task in the same turn, using the prompt template below. Each subagent receives its self-contained task file plus any matching module memory.
   - **Without subagents:** implement the tasks yourself, strictly one at a time, following each task file exactly.
5. **Verify each task statically — do not build.** Check each item in the task's `## Acceptance criteria` by inspection: the file/export exists, the signature matches the pinned contract, the logic covers every row of the edge-case table, the wiring points at the right symbol. Any criterion that genuinely requires execution (build, typecheck, full suite, e2e, visual check) is **not** run here — mark it `- [ ] … *(pending: build-gate)*` in the task file. A task may close when every criterion is either statically checked off or explicitly parked as `pending: build-gate`.
6. **Close each task.** Check off its statically-verified boxes, set `status: done`, `git mv` it to `done/`, update the tables in `BOARD.md`, and commit with message `task(<id>): <title>`.
7. **Report the wave.** Per task: done/failed/blocked, what was built, what's parked for the gate, any deviations. If this was **not** the final wave, tell the user the next wave is unlocked, offer to continue, and state plainly that nothing has been compiled yet.
8. **Run the build gate — once, only after the final wave closes.** See the section below. This is the only place the project gets built.

## Subagent prompt template

Give each subagent exactly this, filling in the blanks:

```
You are implementing one task from a larger feature. Work ONLY within the
files listed in the task's `files` field. If you find you must modify any
other file, STOP and report back instead of editing it — another agent may
own that file.

Assume all tasks from earlier waves are complete: the interfaces listed in
"Interfaces you must conform to" exist and work as specified.

Respect the MODULE MEMORY below (if any): its recorded decisions, gotchas,
patterns, and contracts about the units you're touching are known truths —
follow them, don't relitigate them.

DO NOT BUILD. Do not run the build, the typechecker, the full test suite,
the linter, or an e2e run — not even "just to check". Other agents are
working in this same tree right now and the project is built once, later,
by the orchestrator. Running it here is wasted time and corrupts the shared
build cache.

Implement the task below, then verify each item in "Acceptance criteria"
STATICALLY: read the code you wrote and confirm the export/signature/route
exists as specified, the pinned contracts are matched exactly, and every
edge case listed is actually handled. For any criterion that cannot be
settled without executing something, say so explicitly and label it
"pending: build-gate" instead of running it.

Report: files changed, how each criterion was verified (or why it's parked
for the build gate), and any problems.

--- MODULE MEMORY (from knowledge/, or "none") ---
<contents of the knowledge/ files matching this task's `files`, or "none">
--- END MODULE MEMORY ---

--- TASK FILE ---
<full contents of the task .md file>
--- END TASK FILE ---
```

## The build gate (once per board)

Trigger: **every task on the board is in `done/`** — the final wave just closed. Not before. If the user only asked for a mid-board wave, skip the gate and say so in the report; the code is uncompiled until the board finishes.

1. **Detect the real commands** from the project (`package.json` scripts, `Makefile`, `pyproject.toml`, CI config) — don't assume `npm run build`.
2. **Run them once, in this order, stopping at the first failure:** typecheck/build → test suite → linter. Stopping early is deliberate: a type error makes the test output noise.
3. **Resolve the parked criteria.** Walk every `pending: build-gate` item across `done/` task files. Green gate ⇒ check them off. Red gate ⇒ leave them open.
4. **Attribute failures to tasks** using `BOARD.md`'s file-ownership map: map each error's file back to its owning task. That's the payoff of deferring — the map tells you who broke it without a per-task build.
5. **Write `kanban-<feature>/BUILD.md`** (template below) and commit it as `build: gate results`.
6. **On failure:** `git mv` each implicated task back to `in-progress/`, add a `## Blockers` section quoting the actual error, and report. Fix them, then re-run the gate — a re-run is a *repair* cycle, not routine.

```markdown
# Build gate – [Feature Name]

Date: YYYY-MM-DD · Commit: <sha> · Board: all N tasks in done/

| Step | Command | Result | Time |
|------|---------|--------|------|
| Typecheck/build | `npm run build` | ✅ pass | 48s |
| Tests | `npm test` | ❌ 2 failed | 31s |
| Lint | `npm run lint` | not reached | – |

## Failures → owning task
| Error (file:line) | Owning task | Status |
|-------------------|-------------|--------|
| src/lib/csv.ts:44 – TS2345 | 101 | reopened |

## Criteria resolved
Which `pending: build-gate` items this run settled, per task.

## Raw output
Relevant excerpts (trimmed, not the whole log).
```

## Hard rules

- **One build per board.** No subagent builds. No per-task build. No per-wave build. If you catch yourself running a build outside step 8, stop — that is the exact bottleneck this skill is designed to avoid.
- **Static verification is real verification.** "Don't build" is not "don't check". Reading the code against the pinned contract and the edge-case table catches most task-local defects; the gate exists for the rest. Never check off a criterion you neither inspected nor parked.
- **File ownership is law.** An agent (or you) never edits a file owned by another task in the same wave. Violations are how parallel work corrupts itself. If a task turns out to need a file it doesn't own, stop that task, mark it blocked, and tell the user — the board's ownership map needs fixing, which is a `prd-to-kanban` concern, not an improvisation.
- **The interface contract is fixed.** Implement exactly the signatures/routes/schemas in "Interfaces you must conform to". If a contract seems wrong, flag it; don't unilaterally change it, because a parallel task is building against it.
- **No scope creep.** Do what the task says, honor its "Out of scope" section, and resist drive-by refactors — they create conflicts with parallel work.
- **Failures are reported, not hidden.** If acceptance criteria can't be met, leave the task in `in-progress/`, add a `## Blockers` section to the task file describing exactly what failed, and surface it in the final report.
- **Commit per task**, not one mega-commit, so each task's changes are reviewable in isolation.

## Handling partial failure in a wave

If some tasks in a wave succeed and others fail: close the successful ones normally, leave failed ones in `in-progress/` with their `## Blockers` documented, and do NOT start the next wave — and therefore do not run the build gate, which requires a fully closed board. Present the user the failure summary and options (retry, re-scope the task, or fix manually).

## Example

**Input:** "Implement wave 1 of the CSV export board."

**Behavior:** Reads `docs/kanban-reports-csv-export/BOARD.md`, moves `101-csv-serializer.md` and `102-permission-helper.md` to `in-progress/`, spawns two subagents in parallel (one per task file, both told not to build). Each agent verifies its criteria statically — `serializeReportToCsv` is exported with the pinned signature and handles all four edge-case rows — and parks "unit tests pass" as `pending: build-gate`. Moves both to `done/`, updates `BOARD.md`, commits `task(101): …` and `task(102): …`. No build ran. Reports: "Wave 1 complete (2/2), verified statically; 3 criteria parked for the build gate. Wave 2 (201, 202) is unlocked — nothing is compiled until the board closes. Continue?"

**Later — "implement wave 3":** wave 3's only task closes, so the board is fully in `done/`. The gate fires once: `npm run build` (pass, 48s) → `npm test` (2 failures in `csv.test.ts`) → lint not reached. The ownership map traces both failures to task 101, which goes back to `in-progress/` with the real error text in `## Blockers`. Writes `BUILD.md`, commits `build: gate results`, and reports: "Board complete, gate red. 2 failures, both owned by task 101 — reopened with the errors. Everything else stays green."
