---
name: prd-to-kanban
description: Break a PRD (Product Requirements Document) down into small, self-contained tasks organized as a file-based kanban board, designed so multiple AI agents can execute tasks in parallel. Each task is its own Markdown file. Use this skill whenever the user asks to turn a PRD, spec, or requirements document into tasks, tickets, a backlog, a kanban board, or a parallelizable work breakdown — even if they don't say "kanban". Also use it right after generating a PRD when the user asks "now split this into tasks" or "plan the implementation".
---

# PRD to Kanban (parallel-ready tasks)

Transform a PRD into a set of small, self-contained task files organized in a kanban-style directory structure. The board lives entirely in the repo as plain Markdown (git-friendly, reviewable in PRs), and tasks are written so that **multiple AI agents can pick up different tasks and work on them simultaneously** — in separate sessions, subagents, or git worktrees — without stepping on each other.

## The parallelism contract

Every task must satisfy these rules. They are the core of this skill:

1. **Self-contained.** An agent must be able to complete the task by reading *only* that one task file plus the repo. Never write "see task 003 for details" or assume knowledge from the PRD — copy the relevant context into the task.
2. **Exclusive file ownership.** Each task declares the files it will create or modify in a `files` field. **No two tasks in the same wave may touch the same file.** This is what makes parallel execution merge-conflict-free. If two pieces of work need the same file, either merge them into one task or put them in different waves.
3. **Explicit interfaces.** When task B builds on task A's output, the *interface* (function signature, API route, type, CLI contract, table schema) is decided now and written into **both** task files. Agents implement against the contract, not against each other's unpredictable output.
4. **Waves, not chains.** Group tasks into numbered waves: all tasks in wave 1 can run at the same time with zero dependencies; wave 2 tasks depend only on wave 1; and so on. Maximize the width of each wave and minimize the number of waves.
5. **Independently verifiable.** Each task's acceptance criteria must be checkable without any other in-flight task being finished (its dependencies from earlier waves are assumed done).

## Workflow

1. **Locate the PRD.** If the user points to a file, read it. Otherwise look for `prd-*.md` in `docs/`, `tasks/`, or the repo root. If several exist, ask which one. If none exists, suggest running the `write-prd` skill first.
2. **Extract the work.** Go through the PRD section by section — especially Functional Requirements, Non-Functional Requirements, UX Notes, Technical Considerations, and Milestones. Every requirement must map to at least one task; no requirement may be silently dropped.
3. **Split into small tasks.** Each task should take one agent roughly ≤2 hours of focused work. Prefer more, smaller, independent tasks over fewer large ones — small tasks are what create parallelism.
4. **Design the interfaces.** Before writing any task file, sketch the shared contracts (types, function signatures, routes, schemas) and the file-ownership map. Adjust task boundaries until no wave has overlapping files.
5. **Assign waves** based on the dependency graph.
6. **Create the board structure** and write one file per task using the template.
7. **Write `BOARD.md`** with the wave-based execution plan, and report a summary to the user: number of tasks, number of waves, maximum parallelism per wave, and any ambiguous requirements.

Do not start implementing any task. This skill produces the board only.

## Board structure

ALWAYS use this layout, inside the same directory where the PRD lives (e.g. `docs/`), in a folder named after the PRD:

```
kanban-<feature-name>/
├── BOARD.md
├── backlog/
│   ├── 101-csv-serializer.md        # wave 1 → IDs 1xx
│   ├── 102-permission-check.md
│   ├── 201-export-button-ui.md      # wave 2 → IDs 2xx
│   └── ...
├── in-progress/
└── done/
```

- All tasks start in `backlog/`. Moving a task between columns = moving the file between folders (`git mv`), which keeps history clean.
- File names: `WNN-kebab-case-title.md`, where the first digit is the **wave** and the last two digits number the task within the wave. The ID encodes the parallel schedule at a glance and never changes when files move between columns.

## Task file template

ALWAYS use this exact structure for each task:

```markdown
---
id: 201
title: Add "Export CSV" button to reports page
status: backlog
wave: 2
depends_on: [101, 102]   # ids from earlier waves only, or []
priority: high            # high | medium | low
estimate: S               # S (≤1h) | M (≤2h)
files:                    # files this task OWNS (creates or modifies)
  - src/pages/Reports.tsx
  - src/pages/Reports.test.tsx
prd_refs: [FR-1, FR-2]
agent_ready: true         # false if the task still has open questions
---

# 201 – Add "Export CSV" button to reports page

## Context (self-contained)
Everything an agent needs to know, restated here even if it repeats the PRD:
what the feature is, why this task exists, and how it fits the whole. Assume
the agent has NOT read the PRD or any other task.

## Interfaces you must conform to
Contracts shared with other tasks, stated exactly:
- Call `serializeReportToCsv(report: Report): string` from
  `src/lib/csv.ts` (implemented in task 101 — assume it exists and works).
- Gate the button behind `canExportReports(user)` from
  `src/lib/permissions.ts` (task 102).

## What to do
- Concrete, ordered steps. Reference real file paths. Stay strictly within
  the files listed in `files` — if you find you need to edit another file,
  STOP and report back instead of editing it.

## Acceptance criteria
- [ ] Objectively verifiable outcomes, checkable with only earlier-wave
      tasks completed. Each item must be testable (run a command, click a
      thing, assert an output).

## Out of scope
Anything an implementer might be tempted to do here but must not — especially
work owned by a parallel task.
```

## BOARD.md template

```markdown
# Kanban – [Feature Name]

Source PRD: [../prd-feature-name.md](../prd-feature-name.md)

## How to run this board with AI agents
- Tasks in the same wave are safe to run **in parallel** (one agent, session,
  or git worktree per task): they share no files.
- Start a wave only when every task in the previous wave is in `done/`.
- To work a task: move its file to `in-progress/`, set `status`, implement,
  verify acceptance criteria, move to `done/`.
- Each task is self-contained: give an agent just the task file.

## Execution plan
| Wave | Tasks | Max parallel agents |
|------|-------|---------------------|
| 1 | 101, 102, 103 | 3 |
| 2 | 201, 202 | 2 |
| 3 | 301 | 1 |

## Shared interfaces
The contracts tasks are built against (signatures, routes, schemas), listed
once here for human review.

## File ownership map
| File | Owned by task |
|------|---------------|
| src/lib/csv.ts | 101 |
| src/pages/Reports.tsx | 201 |

## Backlog
| ID | Wave | Task | Priority | Estimate | Depends on |
|----|------|------|----------|----------|------------|
| 101 | 1 | ... | high | S | – |

## In progress
_(empty)_

## Done
_(empty)_
```

## Task-splitting guidelines

- **Split along file boundaries, then along features.** The file-ownership rule usually dictates the cut: serializer module, UI component, API route, migration, and e2e test naturally become separate tasks.
- Shared plumbing that everything depends on (types, config, scaffolding, migrations) goes in wave 1 as one or a few small tasks, so later waves fan out wide.
- Integration and end-to-end tests that span many files belong in the **final wave**, alone, after parallel work has merged.
- Acceptance criteria come from the PRD's requirements, restated as checkable outcomes — not vague ("works well") but concrete ("clicking Export downloads a `.csv` with one row per report entry").
- Include non-code tasks when the PRD implies them (docs, feature-flag cleanup), but do not invent work the PRD never asked for.
- If a PRD item is too vague to become a runnable task, still create the task, set `agent_ready: false`, list the open questions inside it, and flag it in your final summary — agents should skip non-ready tasks.

## Example

**Input:** "Turn docs/prd-reports-csv-export.md into a kanban board."

**Output:** `docs/kanban-reports-csv-export/` containing `BOARD.md` and:
- Wave 1 (parallel): `backlog/101-csv-serializer.md`, `backlog/102-permission-helper.md`
- Wave 2 (parallel): `backlog/201-export-button-ui.md`, `backlog/202-export-api-route.md`
- Wave 3: `backlog/301-e2e-export-test.md`

Final summary in chat: "Created 5 tasks in 3 waves; up to 2 agents can work simultaneously in waves 1–2. All interfaces are pinned in BOARD.md. Task 202 has one open question about auth, marked `agent_ready: false`."
