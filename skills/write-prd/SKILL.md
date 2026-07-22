---
name: write-prd
description: Generate a Product Requirements Document (PRD) in Markdown for a task, feature, or product idea. Use this skill whenever the user asks for a PRD, product spec, requirements document, feature spec, or wants to "write up" or "document the requirements" for something they plan to build — even if they don't use the word "PRD". Also use it when the user describes a feature idea and asks to formalize, scope, or plan it before implementation.
---

# Write PRD

Generate a clear, complete Product Requirements Document in Markdown based on the user's task or feature description.

## Workflow

0. **Right-size first.** If the request is genuinely trivial — a cosmetic tweak, a copy change, a tiny localized bugfix, moving/restyling one element, with no new module/API/schema and no real design decision — a PRD is overkill. Say so and point the user at the `quick-task` fast path instead of producing ceremony. Only continue here when the work justifies a spec.
1. **Gather context.** Read the user's description carefully. If the request is inside an existing codebase, briefly explore the relevant parts of the repo (README, main modules, related features) so the PRD reflects the real system, not assumptions. **Consult existing memory before deriving anything:** if a `knowledge/` module memory (from `close-cycle`) or a `research/` cache exists, read the entries for the modules this feature will touch, and let their recorded decisions, constraints, and gotchas shape the PRD instead of re-deciding them.
2. **Ask before assuming — but only when it matters.** If critical information is missing (target users, success criteria, hard constraints), ask up to 3 concise clarifying questions in one message. If the gaps are minor, proceed and record your assumptions in the "Open Questions / Assumptions" section instead of blocking.
3. **Write the PRD** using the template below.
4. **Save the file** as `prd-<kebab-case-feature-name>.md`. Default location: a `docs/` or `tasks/` directory if one exists in the repo; otherwise the repo root. Tell the user where it was saved.

## PRD Template

ALWAYS use this structure. Omit a section only if it is genuinely not applicable, and say so briefly rather than leaving it silently absent.

```markdown
# PRD: [Feature Name]

## 1. Overview
One or two paragraphs: what this is, the problem it solves, and why now.

## 2. Goals
- Bullet list of concrete objectives.

## 3. Non-Goals
- What is explicitly out of scope for this iteration. Be specific — this prevents scope creep.

## 4. Target Users & Use Cases
Who this is for and the main scenarios in which they'll use it.

## 5. User Stories
- As a [user type], I want [action] so that [benefit].

## 6. Functional Requirements
Numbered list. Each requirement is specific and testable, and — **proportional to its complexity** — states the concrete inputs, the expected output/behavior, and the known edge cases, so it can later become an almost decision-free task. A trivial requirement stays one line; a complex one spells out its boundaries.
1. FR-1: The system must ... — Inputs: ...; Output/behavior: ...; Edge cases: (empty / invalid / at the limit) → ...
2. FR-2: The system must ...

## 7. Non-Functional Requirements
Performance, security, accessibility, compatibility, i18n — whichever apply.

## 8. UX / Design Notes
Key flows and every state each surface can be in — **empty, loading, error, boundary** (zero items, huge input, special characters) — with the expected behavior for each, plus any mockup references. These states are what later become task edge cases, so name them here rather than leaving them implicit. Keep it brief if design is handled elsewhere.

## 9. Technical Considerations
An inventory concrete enough to seed detailed, low-guesswork tasks:
- **Affected surface:** the real files / modules / components / routes this will create or modify — actual paths when inside a repo.
- **Contracts:** data-model changes (fields, types, nullability), API request/response shapes, and external-service contracts, pinned exactly.
- **Constraints, dependencies & gotchas:** ordering requirements, migrations, feature flags, and known footguns — cite the relevant `knowledge/` module memory or `research/` chunk when one exists rather than restating it from scratch.

## 10. Success Metrics
How we'll know it worked. Prefer measurable metrics (adoption %, latency target, error-rate reduction).

## 11. Milestones / Rollout
Phases or rough sequencing if relevant (MVP → v1 → follow-ups). Include feature-flag or migration notes when applicable.

## 12. Open Questions / Assumptions
Anything unresolved, plus assumptions made while writing this PRD.
```

## Writing guidelines

- Write for a mixed audience: a junior developer should be able to implement from it, and a stakeholder should be able to review it. Avoid unnecessary jargon.
- Requirements are testable statements ("must", "must not"), not vague aspirations ("should be fast" → "p95 response time under 300 ms").
- **Detail is proportional to complexity.** A one-line change gets a one-line requirement; a subsystem gets pinned contracts, explicit states, and edge cases. The goal is that the downstream `prd-to-kanban` step can write almost decision-free tasks from this PRD — so be *precise* (real paths, exact contracts, named edge cases) even while staying *concise*. Don't pad trivia to look thorough, and don't under-spec complexity.
- Prefer prose and short bullets over walls of tables. Keep the whole document readable in one sitting — typically 1–3 pages; precision comes from specificity, not length.
- Do not invent metrics, users, or constraints the user never implied; put uncertain items in Open Questions instead.
- Do not start implementing the feature. This skill produces the document only; wait for the user to review it before writing code.

## Example

**Input:** "Write a PRD for adding CSV export to our reports page."

**Output:** A file `docs/prd-reports-csv-export.md` following the template, with functional requirements like "FR-1: The reports page must display an 'Export CSV' button visible to all users with report-view permission" and non-goals like "Excel (.xlsx) export" and "scheduled/emailed exports".
