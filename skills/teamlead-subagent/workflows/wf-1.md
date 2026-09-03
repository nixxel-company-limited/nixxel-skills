Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-1: Feature (canonical TeamLead spec exists)

**Total waves: 4** (Wave 0 through Wave 3 + Validation Gate)

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** (one-shot, `opus`, background) -- survey schema, modules, dependencies that may be affected
- **Sn Dev** (one-shot, `opus`, background) -- survey codebase, environment, technical feasibility

**Context for agents:**
- Canonical spec: `.state/{TASK_ID}/spec.md`
- Canonical plan: `.state/{TASK_ID}/plan.md`
- `.state/{TASK_ID}/research.md`, if it exists, as supporting context
- Original Human PRD/spec, if any, as background only

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Analysis + Design (parallel)

**Agents:**
- **BA** (one-shot, `fable`, background) -- analyze spec, produce AC (Acceptance Criteria) list
- **SA** (one-shot, `fable`, background) -- design architecture: components, API contracts, data model changes

Both run on `fable`. This is the thinking wave: a wrong AC or a wrong contract here is more expensive than every wave that follows it.

**Context for agents:**
- `wave-0-impact.md` (from Wave 0)
- `.state/{TASK_ID}/spec.md` (canonical requirements)
- `.state/{TASK_ID}/plan.md` (canonical implementation plan)
- Original Human PRD/spec, if any, as background only

**Lead action after wave:**
- Merge BA's AC list + SA's design into `.state/{TASK_ID}/wave-1-design.md`

---

### Wave 2 -- Implementation Loop (Dev↔QA teammate pair)

Read `teammate-loops.md` before spawning. Both teammates are spawned **in the same message** and then loop on their own; the Lead stays out of the inner loop.

**Agents:**
- **`qa-{TASK_ID}`** (teammate, `opus`) -- write tests from the AC list according to `.state/{TASK_ID}/plan.md`; backend/API/service tests MUST FAIL first, then message `dev-{TASK_ID}` — fast tests seen RED; heavy tests may record `RED assumed: {why}` per the plan's Test Classification
- **`dev-{TASK_ID}`** (teammate, `opus`) -- implement the planned change so QA's tests pass; stay inside the assigned task slice

**Loop:** QA RED → Dev implements → QA verifies. PASS → QA writes the QA Verify Report and sends it to the Lead. FAIL → QA messages Dev directly with the failing tests, max 3 rounds, then QA escalates to the Lead with what it tried and the suspected cause.

**Context for agents:**
- `.state/{TASK_ID}/spec.md` (canonical AC and behavior)
- `.state/{TASK_ID}/plan.md` (task slice + test strategy)
- `wave-1-design.md` (AC list + architecture design)
- `wave-0-impact.md` (affected files, constraints, risks)

**Lead action after wave:**
- The pair works through the plan's Batch Order without returning to the Lead between batches; Dev commits per the Commit Plan after QA PASS; the Convention Anchor from `research.md` is in every prompt
- Save the QA Verify Report + Dev's summary to `.state/{TASK_ID}/wave-2-implementation.md`
- Run `git diff --stat`, then read the files the plan said would change -- Dev drifting outside the planned files is the most common problem to catch here
- Verify required backend/API/service tests existed and were red before implementation; for frontend-only/non-backend scope, verify the plan's risk-based test strategy was followed — fast tests seen RED; heavy tests may record `RED assumed: {why}` per the plan's Test Classification
- Keep both teammates alive through Wave 3 and the Validation Gate

While the pair runs, the Lead prepares the review-wave prompts and updates state. Do not poll the pair.

---

### Wave 3 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **SA** (one-shot, `opus`, background) -- review architecture compliance + cross-file consistency (domains: architecture, data model, convention-structure)
- **Sn Dev** (one-shot, `opus`, background) -- review code quality + performance + convention compliance (domains: code quality, edge cases, security, performance, convention-code)

Test coverage and AC coverage come from the `qa-{TASK_ID}` teammate's Verify Report. Do **not** spawn a fresh QA for the review wave.

**Context for agents:**
- `.state/{TASK_ID}/spec.md` (canonical spec to review against first)
- `.state/{TASK_ID}/plan.md` (canonical plan/task scope)
- `wave-1-design.md` (architecture design to review after spec compliance)
- `wave-2-implementation.md` (what was implemented + QA Verify Report)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect both review reports and save them to `.state/{TASK_ID}/wave-3-review.md`
- Route findings per the "Review findings → back to the pair" table in `teammate-loops.md` -- code changes go to `dev-{TASK_ID}`, test gaps to `qa-{TASK_ID}`
- Proceed to Validation Gate

---

### Validation Gate

Preceded by the Final Verification wave (see `common.md`) -- full unit suite, heavy suite, E2E, run once.

Lead performs checks 4-6 (Monorepo + State Sync + Commit Plan), takes check 1 from the `qa-{TASK_ID}` report (re-verified after any fixes) and checks 2-3 from the SA/Sn Dev reports. All checks PASS → `shutdown_request` to `qa-{TASK_ID}` and `dev-{TASK_ID}` → summarize for Human.

**Notes:**
- This is the standard feature workflow after TeamLead Brainstorm/Spec/Plan gates have produced canonical artifacts
- Wave 1 parallelizes BA + SA because they work independently (BA reads spec, SA reads spec + impact)
- Wave 2 merges the old "write tests" and "implementation" waves into one pair loop -- the red/green round trips no longer pass through the Lead
- Wave 3 parallelizes both reviewers since their domains do not overlap
