---
name: write-prd
description: Generate a Product Requirements Document (PRD) in Markdown for a task, feature, or product idea. Use this skill whenever the user asks for a PRD, product spec, requirements document, feature spec, or wants to "write up" or "document the requirements" for something they plan to build — even if they don't use the word "PRD". Also use it when the user describes a feature idea and asks to formalize, scope, or plan it before implementation.
---

# Write PRD

Generate a clear, complete Product Requirements Document in Markdown based on the user's task or feature description.

## Workflow

1. **Gather context.** Read the user's description carefully. If the request is inside an existing codebase, briefly explore the relevant parts of the repo (README, main modules, related features) so the PRD reflects the real system, not assumptions.
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
Numbered list. Each requirement must be specific and testable.
1. FR-1: The system must ...
2. FR-2: The system must ...

## 7. Non-Functional Requirements
Performance, security, accessibility, compatibility, i18n — whichever apply.

## 8. UX / Design Notes
Key flows, states (empty, loading, error), and any mockup references. Keep it brief if design is handled elsewhere.

## 9. Technical Considerations
Known constraints, affected systems/modules, dependencies, data model or API changes. Reference actual file paths and components when writing inside a repo.

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
- Prefer prose and short bullets over walls of tables. Keep the whole document readable in one sitting — typically 1–3 pages.
- Do not invent metrics, users, or constraints the user never implied; put uncertain items in Open Questions instead.
- Do not start implementing the feature. This skill produces the document only; wait for the user to review it before writing code.

## Example

**Input:** "Write a PRD for adding CSV export to our reports page."

**Output:** A file `docs/prd-reports-csv-export.md` following the template, with functional requirements like "FR-1: The reports page must display an 'Export CSV' button visible to all users with report-view permission" and non-goals like "Excel (.xlsx) export" and "scheduled/emailed exports".
