---
name: for-research
description: Research a BIG task upfront and cache the findings as a structured Markdown knowledge base (a file-based RAG) so that any later prompt, agent, or workflow step can retrieve exactly the context it needs in one shot — without re-exploring the codebase or overflowing the context window. Use this skill whenever a task is too big to hold in one prompt, whenever the user says "research this first", "cache the context", "build a knowledge base", "prepare context for the agents", or before starting a large feature, migration, or refactor. Also use it when repeated conversations keep re-discovering the same codebase facts.
---

# For Research (context cache / markdown RAG)

For a big task, do the expensive exploration **once** — reading the codebase, docs, and external sources — and persist what was learned as small, focused Markdown files with an index. Afterwards, any agent or prompt (a `write-prd` run, an `implement-task` subagent, a fresh chat session) loads the index plus only the 1–3 relevant chunk files, getting full context in a single prompt instead of re-researching.

Think of it as RAG where the filesystem is the vector store and `INDEX.md` is the retriever.

## When this pays off

- The task spans many files/systems and no single context window fits it all.
- Multiple agents (or multiple sessions over days) will need the same background.
- You keep re-answering "how does X work in this repo?" in every conversation.

## Workflow

### Building the cache

1. **Define the research questions.** From the user's big-task description, list what a future implementer must know (architecture, data flow, conventions, external APIs, constraints, prior art). Confirm the list with the user if the task is fuzzy.
2. **Research.** Explore the codebase (entry points, key modules, tests, configs), read relevant docs, and — only if needed and available — search the web for external library/API facts. Take notes as you go.
3. **Chunk the findings.** Split knowledge into files of ONE topic each, sized to be pasted whole into a prompt (target 100–300 lines per chunk). A chunk answers one research question completely and independently.
4. **Write the cache** using the structure below, then verify it: pick 2–3 realistic future prompts and check that INDEX.md alone tells you exactly which chunks to load.
5. **Report**: where the cache lives, what it covers, what it deliberately does not cover, and how to use it (the retrieval protocol below).

### Using the cache (retrieval protocol)

Any prompt/agent that needs context follows this, and the cache's own INDEX.md repeats it:

1. Read `research/<topic>/INDEX.md` (small, always fits).
2. From its table, pick ONLY the chunks relevant to the current question.
3. Load those chunk files; do not load the whole directory.
4. Trust chunks only within their `verified` date — if the cited source files have changed since (`git log --since`), re-verify before relying on them.

When other skills in this repo run (`write-prd`, `prd-to-kanban`, `implement-task`), they should check for an existing `research/` cache first and cite chunk files in their outputs (e.g. a task file's Context section can say "background: research/payments/03-webhook-flow.md").

## Cache structure

ALWAYS use this layout:

```
research/<topic-kebab-case>/
├── INDEX.md
├── 01-architecture-overview.md
├── 02-data-model.md
├── 03-webhook-flow.md
├── 04-external-api-stripe.md
└── 99-open-questions.md
```

### INDEX.md template

```markdown
# Research cache: [Big Task Name]

Purpose: one paragraph — the big task this cache was built for.
Built: YYYY-MM-DD · Sources: repo @ <commit-sha>, docs, [urls]

## How to use this cache
Read this index, load ONLY the chunks matching your question (usually 1–3),
check each chunk's `verified` date against `git log` for its source files.

## Chunks
| File | Answers the question... | Load when you are... |
|------|-------------------------|----------------------|
| 01-architecture-overview.md | How is the system structured end to end? | new to the codebase / writing a PRD |
| 03-webhook-flow.md | How do incoming webhooks get processed? | touching webhook handling |

## Not covered (on purpose)
Topics deliberately out of scope, so readers don't assume absence = ignorance.
```

### Chunk file template

```markdown
---
topic: webhook processing flow
verified: 2026-07-18
sources:
  - src/webhooks/handler.ts
  - src/queue/consumer.ts
  - https://docs.example.com/webhooks
related_chunks: [02-data-model.md]
---

# Webhook processing flow

## Summary (3–5 lines)
The answer in miniature — enough for a reader deciding whether to read on.

## Details
The full findings: how it works, in prose + minimal code excerpts. Always
cite concrete evidence: file paths with line references, exact function
names, config keys. Facts, not impressions.

## Gotchas & constraints
Non-obvious behavior, ordering requirements, rate limits, legacy quirks —
the things that bite implementers.

## Verified how
How each key claim was confirmed (read code at path:line, ran command X,
official doc URL) — so future readers can re-verify cheaply.
```

## Hard rules

- **One topic per chunk, chunks stand alone.** A chunk pasted into a prompt by itself must make sense; cross-references go in `related_chunks`, not mid-sentence assumptions.
- **Every claim is sourced.** File path (with line numbers where useful), command output, or URL. An unsourced "fact" is a future bug: mark genuine uncertainty explicitly and park it in `99-open-questions.md`.
- **Distill, don't dump.** The cache stores conclusions and maps, not wholesale copies of files (the repo already stores those) and not long verbatim excerpts of external docs — summarize in your own words and link.
- **Keep the index honest.** Every chunk appears in INDEX.md's table; the "Load when you are..." column is what makes one-prompt retrieval work, so write it thinking of the future prompt that will need it.
- **Staleness is managed, not ignored.** `verified` dates + source lists make cheap re-verification possible; when updating a cache, re-verify and bump dates on touched chunks only.

## Example

**Input:** "We're going to migrate billing to Stripe — research it first and cache the context."

**Output:** `research/stripe-billing-migration/` with `INDEX.md` and chunks `01-current-billing-architecture.md`, `02-data-model.md` (tables + who writes them, with file:line sources), `03-invoice-lifecycle.md`, `04-stripe-api-essentials.md` (from official docs, summarized + linked), `05-repo-conventions.md`, `99-open-questions.md` ("Is proration required? PRD must decide"). Chat summary: "Cache built from repo @ a1b2c3d, 6 chunks. A later `write-prd` or any implement agent needs only INDEX.md + 1–2 chunks (~200 lines) instead of re-reading ~40 files."
