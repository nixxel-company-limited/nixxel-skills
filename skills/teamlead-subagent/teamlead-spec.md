# TeamLead Spec Protocol

This file defines the spec artifact created after TeamLead Brainstorm Protocol produces an approved design.

The spec is the shared source of truth for SA, BA, Dev, QA, review waves, and final validation. It should be detailed enough for agents to work independently without re-asking the same requirement questions.

External PRDs, Notion pages, issue descriptions, and design docs can inform the spec, but they are not canonical for execution until their relevant requirements are converted into `.state/{TASK_ID}/spec.md` and approved by the Human.

---

## Ownership

Lead may write orchestration artifacts, including specs, wave summaries, agent prompts, and review summaries.

Lead must not edit product source code, tests, migrations, or product config. If a team wants a stricter model where Lead writes no files at all, spawn BA to write the spec artifact, then Lead reviews it against the quality gate in this file.

---

## Location

Default path:

```text
.state/{TASK_ID}/spec.md
```

If there is no task ID:

- Use a short slug based on the work, such as `checkout-empty-state`
- Ask the Human only when external tracking or branch naming depends on the exact ID

Create the `.state/{TASK_ID}/` directory only when writing the spec or wave artifacts.

---

## Required Template

Use this template unless the repo has a stronger local spec format.

```markdown
# Spec -- {TASK_ID}: {Title}

## Goal
{What we are building or changing, and why it matters.}

## Scope
### In Scope
- ...

### Out of Scope
- ...

## User / System Behavior
- ...

## Acceptance Criteria
- [ ] ...
- [ ] ...

## Technical Notes
### Affected Areas
- ...

### API / Data / Contract Changes
- ...

### Dependencies / Migration
- ...

## Test Scenarios
- ...

## Risks and Open Questions
- ...

## Human Approval
- Status: Pending | Approved
- Proposed design approved by: {Human / date or conversation reference}
- Written spec reviewed by: Pending | {Human / date or conversation reference}
```

---

## Section Guidance

Goal:

- State the user or system outcome, not just the implementation task
- Include the reason the change matters when known

Scope:

- In Scope describes what agents are allowed to implement
- Out of Scope prevents accidental expansion
- Include non-goals that came up during brainstorming

User / System Behavior:

- Describe observable behavior
- Include empty, loading, error, permission, and edge behavior when relevant
- For backend work, describe request/response behavior and side effects

Acceptance Criteria:

- Make each item testable
- Prefer observable statements over implementation wishes
- Include regression expectations when changing existing behavior

Good:

- [ ] When an unauthenticated user opens `/billing`, they are redirected to login and returned to `/billing` after sign-in.

Weak:

- [ ] Billing auth works.

Technical Notes:

- List likely affected modules from the context scan
- Describe isolation boundaries and interfaces between units
- State API/data/contract impact explicitly, even when the answer is "none"
- Note dependencies, migrations, env vars, feature flags, or rollout concerns

Test Scenarios:

- Cover happy path
- Cover important edge cases
- Cover permission/error behavior
- Note whether QA should write tests before Dev for backend/API/service work

Risks and Open Questions:

- Open questions must be resolved before implementation, or explicitly marked accepted by the Human
- Risks should tell agents what to watch for, not just list vague concerns

Human Approval:

- Proposed design must be `Approved` before writing the spec artifact
- Written spec must be reviewed and approved before writing implementation plans or spawning Dev
- Include a short reference like "Approved in chat after proposed design revision 2"

---

## Spec Self-Review

After writing the spec and before asking the Human to review it, Lead must do a quick self-review and fix issues inline:

1. Placeholder scan: no `TBD`, `TODO`, ellipses that stand in for missing content, or vague "handle edge cases" wording.
2. Internal consistency: goals, scope, behavior, ACs, and technical notes do not contradict each other.
3. Scope fit: the spec is small enough for one implementation plan; if it spans independent subsystems, split it.
4. Ambiguity check: requirements cannot reasonably be interpreted two different ways.
5. Interface check: units, APIs, data contracts, and ownership boundaries are explicit enough for agents.

Do not move to the quality gate until the self-review is clean.

---

## Spec Quality Gate

Before starting TeamLead Plan Protocol or any implementation workflow, Lead must verify:

- Goal is clear
- In scope and out of scope are explicit
- Acceptance Criteria are testable
- Test Scenarios cover happy path plus important edge cases
- API/data/contract impact is stated, even if "none"
- Dependencies/migration concerns are stated, even if "none"
- Risks/open questions are resolved or accepted
- Spec Self-Review passed
- Proposed design approval is recorded
- Written spec review approval is recorded

If any item fails, revise the spec before proceeding.

---

## Handoff Rules

After the spec passes the quality gate:

1. Use the spec as context for TeamLead Plan Protocol
2. Pass the spec path to Wave 0 agents
3. Require SA/Sn Dev to review the spec for feasibility and gaps during Wave 0 when the workflow calls for them
4. Update the spec only for requirement/design changes, not for implementation trivia
5. If Wave 0 finds a requirement blocker, revise the spec and get Human approval again before implementation planning

Do not let agents replace the spec with their own interpretation. Agent reports can refine the spec, but Lead owns the canonical artifact.
