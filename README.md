# AI Workflow For Projects

This repo exists to save my [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills and document how I use AI in my software development workflow. It stores the skills and techniques I actually use day to day — from turning an idea into a PRD, all the way to implementing it with parallel AI agents and validating the result.

Feel free to look around if you want to contribute or use these skills.

## The workflow

The skills are designed to chain together as a pipeline:

```
for-research → write-prd → prd-to-kanban → implement-task → review → qa → close-cycle
 (big tasks)                                  (per wave)      (AI)   (human)  (learnings)

quick-task ─── fast path: small, obvious changes skip the whole pipeline
```

1. **Research first** (optional, for big tasks) — explore the codebase/docs once and cache the findings as a markdown knowledge base.
2. **Write a PRD** — turn the idea into a clear requirements document.
3. **Break it into a kanban board** — small, self-contained task files, organized in waves so multiple AI agents can work **in parallel** without conflicts. Each task is spec'd in as much detail as its complexity warrants, so the implementing agent decides as little as possible.
4. **Implement** — one agent per task, wave by wave. Agents verify their work **statically** and never build; once the last wave closes, the project is built, tested and linted **exactly once** and the results are recorded in `BUILD.md`.
5. **Review** — the AI audits everything against the PRD and acceptance criteria, reusing the build gate's results instead of re-running them.
6. **QA** — a plan for a *human* to validate the work and check best practices.
7. **Close the cycle** — distill what was learned into a durable, per-module knowledge base.

Three things cut across the pipeline:

- **One build per cycle.** Building is the most expensive step and N agents building concurrently in one working tree fight over the same caches (`dist/`, `.next/`, `.angular/cache`) — slow *and* flaky. So execution is deferred and batched: task agents verify by inspection, park anything needing execution as `(pending: build-gate)`, and a single gate at the end of the last wave settles all of it, attributing any failure back to its owning task via the board's file-ownership map. `review` reads that record rather than re-running it.
- **`quick-task`** is a fast path: a trivial change (move a button, tweak a color, fix some copy) skips the PRD/board/review ceremony entirely — consult memory, change, verify, commit.
- **`close-cycle` builds a `knowledge/` memory** — durable per-module notes (decisions, gotchas, patterns, contracts) keyed by source file. `write-prd`, `prd-to-kanban`, `implement-task`, and `quick-task` all **read** it before they act, so recorded knowledge is reused instead of re-derived; `close-cycle` **writes** it when the cycle ends. The learnings loop back to the front of the pipeline.

## Skills

| Skill | What it does |
|-------|--------------|
| [`for-research`](skills/for-research/SKILL.md) | Researches a big task upfront and caches the context as a file-based RAG (markdown chunks + index), so later prompts load only what they need. |
| [`write-prd`](skills/write-prd/SKILL.md) | Generates a PRD in markdown for a task or feature — precise enough (real paths, pinned contracts, named edge cases) to seed near-decision-free tasks. |
| [`prd-to-kanban`](skills/prd-to-kanban/SKILL.md) | Transforms the PRD into a file-based kanban board of small tasks, designed for parallel execution by AI agents (waves, exclusive file ownership, pinned interfaces). Spec's each task in detail proportional to its complexity so the executor barely has to think. |
| [`implement-task`](skills/implement-task/SKILL.md) | Executes the board's tasks using one or more agents in parallel, respecting the board's rules and the `knowledge/` module memory. Owns the single build gate: one build/test/lint run after the final wave, recorded in `BUILD.md`. |
| [`review`](skills/review/SKILL.md) | AI review of everything created: requirements traceability, task verification, code quality, and respect for recorded module decisions. Reuses the build gate's output instead of re-running it. |
| [`qa`](skills/qa/SKILL.md) | Generates a QA + code-review plan for a human to validate the feature and check development best practices. |
| [`close-cycle`](skills/close-cycle/SKILL.md) | At the end of a cycle, distills the learnings into a durable, per-module knowledge base (keyed by source file) that the other skills read before they act. |
| [`quick-task`](skills/quick-task/SKILL.md) | Fast path for small, obvious changes — skips the PRD/board/review pipeline: consult memory, change, verify, commit. |
| [`grilling`](skills/grilling/SKILL.md) / [`grill-me`](skills/grill-me/SKILL.md) | A relentless one-question-at-a-time interview to stress-test a plan or design before building it. |

## How to use

Copy the skills into your project (or globally):

```bash
# per project
cp -r skills/* your-project/.claude/skills/

# or globally, for all projects
cp -r skills/* ~/.claude/skills/
```

Claude Code picks them up automatically — just describe what you want ("write a PRD for X", "turn this PRD into tasks", "implement wave 1") and the right skill triggers. You can also invoke them directly, e.g. `/grill-me`.

## Contributing

Found a better technique, or improved one of these skills? PRs and issues are welcome.
