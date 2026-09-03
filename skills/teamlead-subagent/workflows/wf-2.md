Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-2: Feature (started with no spec)

**Total waves: 4** (Wave 0 through Wave 3 + Validation Gate)

WF-2 starts only after TeamLead has already completed:

1. `teamlead-brainstorm.md`
2. `teamlead-spec.md`
3. `teamlead-plan.md`

Therefore `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md` must exist before WF-2 begins. BA may refine AC during Wave 1, but BA must not be the first creator of the canonical spec after agents are spawned.

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** (one-shot, `opus`, background) -- survey architecture that may be affected
- **Sn Dev** (one-shot, `opus`, background) -- survey codebase + technical feasibility

**Context for agents:**
- Canonical spec: `.state/{TASK_ID}/spec.md`
- Canonical plan: `.state/{TASK_ID}/plan.md`
- `.state/{TASK_ID}/research.md`, if it exists, as supporting context
- Raw Human request as background only

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`
- If blockers require requirement changes, revise spec/plan and get Human approval again

---

### Wave 1 -- Analysis + Design (parallel)

**Agents:**
- **BA** (one-shot, `fable`, background) -- review canonical spec, refine AC/test scenarios, identify gaps
- **SA** (one-shot, `fable`, background) -- design architecture: components, API contracts, data model changes

Both run on `fable` for the same reason as WF-1: this is the wave whose output every later wave is built on.

**Context for agents:**
- `wave-0-impact.md` (from Wave 0)
- `.state/{TASK_ID}/spec.md` (canonical requirements)
- `.state/{TASK_ID}/plan.md` (canonical implementation plan)

**Lead action after wave:**
- Merge BA's AC refinement + SA's design into `.state/{TASK_ID}/wave-1-design.md`
- If BA/SA finds spec gaps, update `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`, then get Human approval again

---

### Wave 2 -- Implementation Loop (Dev↔QA teammate pair)

Read `teammate-loops.md` before spawning. Both teammates are spawned **in the same message** and loop on their own.

**Agents:**
- **`qa-{TASK_ID}`** (teammate, `opus`) -- write tests from AC according to `.state/{TASK_ID}/plan.md`; backend/API/service tests MUST FAIL first, then message `dev-{TASK_ID}` — fast tests seen RED; heavy tests may record `RED assumed: {why}` per the plan's Test Classification
- **`dev-{TASK_ID}`** (teammate, `opus`) -- implement the planned change so QA's tests pass; stay inside the assigned task slice

**Loop:** QA RED → Dev implements → QA verifies. PASS → QA sends its QA Verify Report to the Lead. FAIL → QA messages Dev directly, max 3 rounds, then QA escalates to the Lead.

**Context for agents:**
- `.state/{TASK_ID}/spec.md` (canonical AC and behavior)
- `.state/{TASK_ID}/plan.md` (task slice + test strategy)
- `wave-1-design.md` (architecture design + refined AC)
- `wave-0-impact.md` (affected files, constraints)

**Lead action after wave:**
- The pair works through the plan's Batch Order without returning to the Lead between batches; Dev commits per the Commit Plan after QA PASS; the Convention Anchor from `research.md` is in every prompt
- Save the QA Verify Report + Dev's summary to `.state/{TASK_ID}/wave-2-implementation.md`
- Run `git diff --stat` and check the changed files against the plan's expected file list
- Verify required backend/API/service tests were red before implementation; for frontend-only/non-backend scope, verify the plan's risk-based test strategy was followed — fast tests seen RED; heavy tests may record `RED assumed: {why}` per the plan's Test Classification
- Keep both teammates alive through Wave 3 and the Validation Gate

---

### Wave 3 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **SA** (one-shot, `opus`, background) -- review architecture compliance + cross-file consistency (domains: architecture, data model, convention-structure)
- **Sn Dev** (one-shot, `opus`, background) -- review code quality + performance + convention compliance (domains: code quality, edge cases, security, performance, convention-code)

AC and test coverage come from the `qa-{TASK_ID}` teammate's Verify Report, not from a fresh QA spawn.

**Context for agents:**
- `.state/{TASK_ID}/spec.md` (canonical spec to review against first)
- `.state/{TASK_ID}/plan.md` (canonical plan/task scope)
- `wave-1-design.md` (design to review after spec compliance)
- `wave-2-implementation.md` (what was implemented + QA Verify Report)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect both review reports and save them to `.state/{TASK_ID}/wave-3-review.md`
- Route findings per the "Review findings → back to the pair" table in `teammate-loops.md`
- Proceed to Validation Gate

---

### Validation Gate

Preceded by the Final Verification wave (see `common.md`) -- full unit suite, heavy suite, E2E, run once.

Lead performs checks 4-6, takes check 1 from the `qa-{TASK_ID}` report and checks 2-3 from the reviewer reports. All checks PASS → `shutdown_request` to both teammates → summarize for Human.

**Notes:**
- WF-2 no longer creates a spec inside the agent workflow. TeamLead local gates create the canonical spec before workflow selection.
- WF-2 exists to preserve routing for tasks that started without a spec, but by the time agents spawn it consumes `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`.
- Wave structure is identical to WF-1 once the canonical artifacts exist; only the Wave 1 BA mission differs (refine rather than author the AC list).
