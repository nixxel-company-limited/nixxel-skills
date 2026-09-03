# TeamLead Plan Protocol

This file defines the local implementation planning protocol for TeamLead. It is the source of truth for turning an approved TeamLead spec into agent-ready work.

Use it after the Human has approved `.state/{TASK_ID}/spec.md`.

---

## Purpose

The plan turns an approved spec into agent-ready work. It must give Dev, QA, SA, and reviewers enough context to work independently while preserving TeamLead control over scope, sequencing, and validation.

The plan is an orchestration artifact. Lead may write it. Lead must not edit product source code, tests, migrations, or product config.

---

## Location

Default path:

```text
.state/{TASK_ID}/plan.md
```

Keep the spec and plan in the same `.state/{TASK_ID}/` directory so every wave can reference both.

---

## Required Template

```markdown
# Plan -- {TASK_ID}: {Title}

## Source Spec
- Spec: `.state/{TASK_ID}/spec.md`
- Approval: {approval reference}

## Goal
{One sentence describing what this implementation delivers.}

## Architecture Summary
{2-5 bullets describing the chosen approach and boundaries.}

## Files and Ownership
| Area | Files / Paths | Owner | Notes |
|------|---------------|-------|-------|
| ... | ... | `dev-{TASK_ID}` / `qa-{TASK_ID}` / SA / Sn Dev | ... |

## Agent Waves
### Wave 0 -- Impact
- Agents: {role} (mode, model, name) -- e.g. `SA (one-shot, opus, -)`, `Sn Dev (one-shot, opus, -)`
- Inputs: spec + plan
- Outputs: impact report / blockers / recommendations
- Note: Wave 0 output is supporting context for feasibility, risk, and sequencing. It does not supersede the canonical approved spec or this plan; requirement changes found in Wave 0 must go back through spec revision and Human approval.

### Wave 1 -- Design
- Agents: {role} (mode, model, name) -- e.g. `BA (one-shot, fable, -)`, `SA (one-shot, fable, -)`
- Inputs: ...
- Outputs: ...

### Wave 2 -- Implementation Loop
- Agents: {role} (mode, model, name) -- e.g. `QA (teammate, opus, qa-{TASK_ID})`, `Dev (teammate, opus, dev-{TASK_ID})`
- Loop: QA and Dev run the red/green loop themselves -- see `teammate-loops.md` for the message protocol, the 3-round cap, and the escalation path back to Lead
- Inputs: ...
- Outputs: ...

### Wave 3 -- Review
- Agents: {role} (mode, model, name) -- e.g. `SA (one-shot, opus, -)`, `Sn Dev (one-shot, opus, -)`
- Inputs: ...
- Outputs: ...

## Task Breakdown
### Task 1 -- {Name}
**Owner:** {Role}
**Files:** ...
**Depends on:** ...
**Instructions:**
- ...
**Expected output:**
- ...
**Verification:**
- Command: `...`
- Expected result: ...

## Convention Anchor
- Reference feature: {name} -- {why it is the standard for this area}
- Examples per layer: route `{path}`, schema `{path}`, service `{path}`, repository `{path}`, test `{path}`, component/text `{path}` (from research.md)

## Batch Order
| Batch | Tasks | Test type (fast / heavy) | Parallel-safe? | Depends on |
|-------|-------|--------------------------|:--------------:|------------|
| 1 | Task 1, Task 2 | fast | no | -- |
| 2 | Task 3 | heavy | no | Batch 1 |
(Batches with heavy tests are always sequential. Mark parallel-safe only when every batch in the group has fast tests only and touches disjoint modules.)

## Test Classification
| Test group | Type | RED before implement? | Run scope during the loop | Command |
|------------|:----:|:---------------------:|---------------------------|---------|
| {new unit tests} | fast | yes -- new file only | batch tests + touched module | `{cmd}` |
| {API/integration tests hitting DB} | heavy | no (RED assumed: endpoint does not exist yet) | once after implement, batch scope only | `{cmd}` |
| {bug-fix regression test} | fast/heavy | yes -- always, proves reproduction | that test only | `{cmd}` |
| E2E | heavy | no | Final Verification only | `{cmd}` |

## Test Strategy
- Unit:
- Integration:
- E2E: run once in Final Verification, never inside the loop
- Manual / exploratory: {cases auto tests cannot cover -- these are what the Human will test at the end}

## TDD Requirements
- Backend/API/service changes: QA writes tests before Dev implementation; fast tests must be seen RED, heavy tests may skip the pre-run when failure is certain
- Frontend-only changes: tests required when behavior is non-trivial or regression risk is meaningful

## Commit Plan
| Commit unit | Covers | Message |
|-------------|--------|---------|
| 1 | Batch 1 (Task 1-2) | `feat(refunds): add export query schema + admin route` |
| 2 | Batch 2 (Task 3) | `feat(refunds): build xlsx export service` |
(A commit unit is a readable step of the feature -- it may span several tasks and loop rounds. Do not commit per task or per round. The sequence should read as a story a reviewer can follow commit by commit.)

## Risks / Sequencing Notes
- ...

## Plan Quality Gate
- Status: Pending | Passed
```

---

## Planning Rules

Map files before tasks:

- Identify likely files or directories before assigning work
- Keep monorepo boundaries explicit
- One agent owns one repo
- One Dev agent gets only 1-2 focused tasks at a time

Make task steps agent-ready:

- Include exact paths when known
- Include prior wave artifacts each task depends on
- Include expected output format
- Include verification command and expected result
- Mark dependencies so only independent tasks run in parallel

Avoid:

- "TBD", "TODO", "handle edge cases", "write tests" without details
- Assigning one agent broad ownership over unrelated areas
- Asking Dev to discover requirements already decided in the spec
- Spawning implementation before tests for backend/API/service changes

---

## Plan Quality Gate

Before choosing workflow and spawning agents, Lead must verify:

- Source spec path is included
- Goal matches the approved spec
- Affected areas and file ownership are mapped
- Agent waves are listed with inputs and outputs
- Every agent has a spawn mode and a model per the Model Policy in `SKILL.md`; the Dev/QA teammate names are listed
- The Dev/QA loop escalation path (3 rounds -> Lead) is stated
- Every Acceptance Criteria item maps to at least one task or test scenario
- Task dependencies are explicit
- Parallelizable tasks are identified
- Backend/API/service TDD requirements are explicit when applicable
- Convention Anchor is named with example paths per layer (from research.md); no Dev/QA prompt will have to guess the repo standard
- Every test group is classified fast or heavy, with RED expectation and loop run scope
- Batch Order is listed; batches with heavy tests are sequential; parallel-safe groups are justified
- Commit Plan groups tasks into readable commit units with intended messages (no per-task, per-round commits)
- Manual/exploratory cases that auto tests cannot cover are listed for the Human summary
- Verification commands are listed where known
- No placeholders remain
- Monorepo constraints are represented
- Branch/worktree isolation and baseline test expectations are stated

If any item fails, revise the plan before spawning agents.

---

## Plan Self-Review

Before marking the plan gate as passed, Lead must review the plan against the spec:

1. Requirement coverage: every goal, scope item, behavior, and AC has a corresponding task, test, or explicit non-implementation note.
2. Placeholder scan: no `TBD`, `TODO`, "similar to above", or vague instructions that force Dev to invent requirements.
3. Type and contract consistency: names, paths, API shapes, and data fields stay consistent across tasks.
4. Sequencing: dependencies are explicit and independent work is marked parallelizable.
5. Verification: each implementation task has a concrete verification path or explains why verification is manual.

Fix issues inline before spawning agents.

---

## Execution Mode

After the plan passes the quality gate, Lead chooses the execution mode:

- **Wave/Subagent execution:** default for implementation work; spawn agents according to `workflows/common.md` + the selected `workflows/wf-N.md`. Implementation runs as a `qa-{TASK_ID}` + `dev-{TASK_ID}` teammate pair; impact, design, and review agents are one-shot.
- **Inline execution:** only for tiny, low-risk orchestration-only work where no product files are edited by Lead.
- **Human handoff:** when the user wants the plan/spec but not execution.

For Wave/Subagent execution, use two-stage review:

1. Spec compliance review: did the implementation build exactly what the approved spec and plan require?
2. Quality review: architecture, correctness, security, performance, conventions, and tests.

---

## Agent Statuses

Require agents to end reports with one status:

- `DONE`: completed as requested, no known concerns
- `DONE_WITH_CONCERNS`: completed, but risks or follow-up items remain
- `NEEDS_CONTEXT`: cannot continue safely without specific missing context
- `BLOCKED`: cannot proceed because of a blocker that Lead/Human must resolve

Lead action:

- `DONE`: review output and continue
- `DONE_WITH_CONCERNS`: decide whether concerns require another wave or Human approval
- `NEEDS_CONTEXT`: provide targeted context and resume the agent
- `BLOCKED`: escalate, revise spec/plan, or ask Human

---

## Execution Preflight

Before spawning Dev:

- Verify branch/worktree isolation is appropriate for the repo
- Run or identify baseline tests when practical
- Confirm `git status` so unrelated dirty files are not mixed into the task
- Confirm spec and plan paths exist and are referenced in prompts
- Confirm each prompt includes only the assigned task slice, relevant context, constraints, and expected output

Do not make subagents rediscover the whole spec or plan. Curate the exact section they need.

---

## Handoff

After the plan passes the quality gate:

1. Select the workflow from the Decision Table, then read `workflows/common.md` and only the matching `workflows/wf-N.md` -- together with `validation.md`, `review-domains.md`, and `teammate-loops.md` in the same turn, since all of them will be needed before the first spawn
2. Run the Prompt Validation Checklist (`validation.md`) before every spawn
3. Follow `teammate-loops.md` when spawning the Dev/QA pair
4. Pass both spec and plan paths to each agent
5. Require agents to report changed files, verification commands, and blockers
6. Keep `.state/{TASK_ID}/` updated after each wave

The plan can evolve after Wave 0 if reviewers find blockers, but Wave 0 remains supporting context. It does not replace the canonical approved spec or plan, and requirement changes must go back through the spec and Human approval.
