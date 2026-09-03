Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-4: Refactor

**Total waves: 4** (Wave 0 through Wave 3 + Validation Gate)

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** (one-shot, `opus`, background) -- survey architecture + dependencies that refactor may affect
- **Sn Dev** (one-shot, `opus`, background) -- survey current codebase + impact of proposed changes

**Context for agents:**
- Architecture-shaping refactors: canonical spec `.state/{TASK_ID}/spec.md` and canonical plan `.state/{TASK_ID}/plan.md`
- Mechanical maintenance refactors: raw refactor scope/goal from Human is sufficient when there is no behavior, contract, schema, or architecture change
- Current code state (read relevant files)

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- New Design

**Agents:**
- **SA** (one-shot, `fable`, background) -- create the new design/architecture for the refactored code

`fable` because the target shape of the code is the one decision this workflow is really making; everything after it is mechanical execution against that design.

**Context for agents:**
- Architecture-shaping refactors: `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`
- `wave-0-impact.md` (affected modules, constraints, risks)
- Mechanical maintenance refactors: raw scope plus current code state

**Lead action after wave:**
- Save design to `.state/{TASK_ID}/wave-1-design.md`

---

### Wave 2 -- Refactor Loop (Dev↔QA teammate pair)

Read `teammate-loops.md` before spawning. Both teammates are spawned **in the same message**.

**Agents:**
- **`dev-{TASK_ID}`** (teammate, `opus`) -- refactor code according to SA's new design, preserving behavior
- **`qa-{TASK_ID}`** (teammate, `opus`) -- run the **existing** suite as the safety net; add tests only where `.state/{TASK_ID}/plan.md` requires them

**Loop:** the pair starts from the existing green suite rather than a new red test. QA establishes the baseline (existing tests pass now) and messages `dev-{TASK_ID}` → Dev refactors → QA re-runs the suite. PASS → QA sends its QA Verify Report to the Lead. FAIL → QA messages Dev with the broken tests, max 3 rounds, then QA escalates to the Lead. A behavior change surfacing as a newly failing test is a finding, not something for Dev to work around.

**Context for agents:**
- Architecture-shaping refactors: `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`
- `wave-1-design.md` (new design/architecture)
- `wave-0-impact.md` (constraints, affected files)
- Test command + baseline suite result

**Lead action after wave:**
- The pair works through the Batch Order and Commit Plan from `plan.md` **if this workflow produced one** (otherwise the batch/commit units the Lead states in the prompt) without returning to the Lead between batches; Dev commits per the Commit Plan after QA PASS; the Convention Anchor from `research.md` **if it exists, otherwise the reference feature identified in Wave 0** is in every prompt
- Save the QA Verify Report + Dev's summary to `.state/{TASK_ID}/wave-2-implementation.md`
- Verify the suite result matches the pre-refactor baseline
- Keep both teammates alive through Wave 3 and the Validation Gate

---

### Wave 3 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **SA** (one-shot, `opus`, background) -- review that refactored code follows the new design (domains: architecture, data model, convention-structure)
- **Sn Dev** (one-shot, `opus`, background) -- review code quality + performance improvements (domains: code quality, edge cases, security, performance, convention-code)

Test coverage and the no-regression check come from the `qa-{TASK_ID}` teammate's Verify Report.

**Context for agents:**
- Architecture-shaping refactors: `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`
- `wave-1-design.md` (design to verify against)
- `wave-2-implementation.md` (what was refactored + QA Verify Report)
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
- Shorter than feature workflows because there is no BA (no new spec/AC needed)
- QA verifies existing tests still pass rather than writing new tests, so the pair loop starts green instead of red
- Focus is on maintaining behavior while improving structure
- Architecture-shaping refactors must consume canonical spec/plan. Purely mechanical maintenance may proceed from raw scope when it does not change behavior, contracts, schemas, infra flow, or policy.
