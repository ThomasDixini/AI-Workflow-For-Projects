# AI Workflow For Projects

This repo exists to save my [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills and document how I use AI in my software development workflow. It stores the skills and techniques I actually use day to day — from turning an idea into a PRD, all the way to implementing it with parallel AI agents and validating the result.

Feel free to look around if you want to contribute or use these skills.

## The workflow

The skills are designed to chain together as a pipeline:

```
for-research → write-prd → prd-to-kanban → implement-task → review → qa
 (big tasks)                                  (per wave)      (AI)   (human)
```

1. **Research first** (optional, for big tasks) — explore the codebase/docs once and cache the findings as a markdown knowledge base.
2. **Write a PRD** — turn the idea into a clear requirements document.
3. **Break it into a kanban board** — small, self-contained task files, organized in waves so multiple AI agents can work **in parallel** without conflicts.
4. **Implement** — one agent per task, wave by wave.
5. **Review** — the AI audits everything against the PRD and acceptance criteria.
6. **QA** — a plan for a *human* to validate the work and check best practices.

## Skills

| Skill | What it does |
|-------|--------------|
| [`for-research`](skills/for-research/SKILL.md) | Researches a big task upfront and caches the context as a file-based RAG (markdown chunks + index), so later prompts load only what they need. |
| [`write-prd`](skills/write-prd/SKILL.md) | Generates a PRD in markdown for a task or feature. |
| [`prd-to-kanban`](skills/prd-to-kanban/SKILL.md) | Transforms the PRD into a file-based kanban board of small tasks, designed for parallel execution by AI agents (waves, exclusive file ownership, pinned interfaces). |
| [`implement-task`](skills/implement-task/SKILL.md) | Executes the board's tasks using one or more agents in parallel, respecting the board's rules. |
| [`review`](skills/review/SKILL.md) | AI review of everything created: requirements traceability, task verification, code quality. |
| [`qa`](skills/qa/SKILL.md) | Generates a QA + code-review plan for a human to validate the feature and check development best practices. |
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
