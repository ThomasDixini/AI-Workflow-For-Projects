---
name: implement-task
description: Execute tasks from a file-based kanban board (created by the prd-to-kanban skill), using one or more agents in parallel to implement them. Use this skill whenever the user asks to implement, work on, resolve, execute, or "run" tasks from the board or backlog — e.g. "implement wave 1", "work on task 201", "resolve all the tasks", "start the kanban" — even if they don't name the board explicitly. Also use it when the user asks to continue implementation of a previously started board.
---

# Implement Task

Execute tasks from a kanban board produced by the `prd-to-kanban` skill. Tasks within a wave are designed to be independent (no shared files, pinned interfaces), so run them **in parallel with subagents whenever subagents are available**; otherwise run them sequentially yourself, one at a time.

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
5. **Verify each task** against its own `## Acceptance criteria` checklist. Run the commands/tests the criteria imply. A task is done only when every box can be honestly checked.
6. **Close each task.** Check off its acceptance-criteria boxes in the file, set `status: done`, `git mv` it to `done/`, update the tables in `BOARD.md`, and commit with message `task(<id>): <title>`.
7. **Report.** Summarize per task: done/failed/blocked, what was built, test results, and any deviations. If the whole wave is done, tell the user the next wave is unlocked and offer to continue.

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

Implement the task below, then verify every item in "Acceptance criteria"
by actually running the relevant commands or tests. Report: files changed,
how each criterion was verified, and any problems.

--- MODULE MEMORY (from knowledge/, or "none") ---
<contents of the knowledge/ files matching this task's `files`, or "none">
--- END MODULE MEMORY ---

--- TASK FILE ---
<full contents of the task .md file>
--- END TASK FILE ---
```

## Hard rules

- **File ownership is law.** An agent (or you) never edits a file owned by another task in the same wave. Violations are how parallel work corrupts itself. If a task turns out to need a file it doesn't own, stop that task, mark it blocked, and tell the user — the board's ownership map needs fixing, which is a `prd-to-kanban` concern, not an improvisation.
- **The interface contract is fixed.** Implement exactly the signatures/routes/schemas in "Interfaces you must conform to". If a contract seems wrong, flag it; don't unilaterally change it, because a parallel task is building against it.
- **No scope creep.** Do what the task says, honor its "Out of scope" section, and resist drive-by refactors — they create conflicts with parallel work.
- **Failures are reported, not hidden.** If acceptance criteria can't be met, leave the task in `in-progress/`, add a `## Blockers` section to the task file describing exactly what failed, and surface it in the final report.
- **Commit per task**, not one mega-commit, so each task's changes are reviewable in isolation.

## Handling partial failure in a wave

If some tasks in a wave succeed and others fail: close the successful ones normally, leave failed ones in `in-progress/` with their `## Blockers` documented, and do NOT start the next wave. Present the user the failure summary and options (retry, re-scope the task, or fix manually).

## Example

**Input:** "Implement wave 1 of the CSV export board."

**Behavior:** Reads `docs/kanban-reports-csv-export/BOARD.md`, moves `101-csv-serializer.md` and `102-permission-helper.md` to `in-progress/`, spawns two subagents in parallel (one per task file), verifies both acceptance checklists (running their unit tests), moves both to `done/`, updates `BOARD.md`, commits `task(101): …` and `task(102): …`, and reports: "Wave 1 complete (2/2). Wave 2 (tasks 201, 202) is unlocked — want me to continue?"
