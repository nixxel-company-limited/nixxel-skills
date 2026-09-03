Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-3: Bug Fix

**Total waves: 4** (Wave 0 through Wave 3 + Validation Gate)

### Wave 0 -- Impact Check

**Agents:**
- **Sn Dev** (one-shot, `opus`, background) -- survey code related to the bug + impact check
- No SA -- bug fixes do not require architecture review

**Context for agents:** Bug description + reproduction steps from Human

**Lead action after wave:**
- Save Impact Report to `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Root Cause Analysis

**Agents:**
- **Sn Dev** (one-shot, `fable`, background) -- analyze root cause of the bug

This is a **different agent from Wave 0**, and the only one in this workflow on `fable`. Wave 0 is reading: survey the code, list what the bug touches. Wave 1 is thinking: reason from symptom to cause through code that did not announce where it was wrong. Getting the root cause wrong sends the Dev to fix the wrong line, so this step gets the stronger model.

**Context for agents:**
- `wave-0-impact.md` (affected files, related modules)
- Bug description + reproduction steps from Human

**Lead action after wave:**
- Save root cause analysis to `.state/{TASK_ID}/wave-1-root-cause.md`

---

### Wave 2 -- Fix Loop (Dev↔QA teammate pair)

Read `teammate-loops.md` before spawning. Both teammates are spawned **in the same message**.

**Agents:**
- **`qa-{TASK_ID}`** (teammate, `opus`) -- write the regression test that reproduces the bug; it MUST FAIL first, then message `dev-{TASK_ID}`
- **`dev-{TASK_ID}`** (teammate, `opus`) -- fix the bug so the regression test passes, without changing the test

**Loop:** QA RED (bug reproduced) → Dev fixes → QA verifies the reproduction is gone and nothing regressed. PASS → QA sends its QA Verify Report to the Lead. FAIL → QA messages Dev directly, max 3 rounds, then QA escalates to the Lead.

**Context for agents:**
- `wave-1-root-cause.md` (what causes the bug, where it occurs)
- `wave-0-impact.md` (affected files, constraints, other affected areas)
- Bug description + reproduction steps from Human

**Lead action after wave:**
- The pair works through the Batch Order and Commit Plan from `plan.md` **if this workflow produced one** (otherwise the batch/commit units the Lead states in the prompt) without returning to the Lead between batches; Dev commits per the Commit Plan after QA PASS; the Convention Anchor from `research.md` **if it exists, otherwise the reference feature identified in Wave 0** is in every prompt
- The regression test is always run RED first, even if it is a heavy test -- that red run is the proof the bug reproduces (run that single test file only)
- Save the QA Verify Report + Dev's fix summary to `.state/{TASK_ID}/wave-2-fix.md`
- Verify the regression test was red before the fix and green after
- Run `git diff --stat` -- a bug fix that touches far more than the root cause named is a signal to check with Dev
- Keep both teammates alive through Wave 3 and the Validation Gate

---

### Wave 3 -- Verify + Review (per Review Domain)

**Agents:**
- **Sn Dev** (one-shot, `opus`, background) -- review code quality + performance (domains: code quality, edge cases, security, performance, convention-code)
- No SA -- bug fix does not require architecture review

Regression coverage and the reproduction check come from the `qa-{TASK_ID}` teammate's Verify Report.

**Context for agents:**
- `wave-1-root-cause.md` (root cause for context)
- `wave-2-fix.md` (what was fixed + QA Verify Report)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect the Sn Dev review report and save it to `.state/{TASK_ID}/wave-3-review.md`
- Route findings per the "Review findings → back to the pair" table in `teammate-loops.md`
- Proceed to Validation Gate

---

### Validation Gate

Preceded by the Final Verification wave (see `common.md`) -- full unit suite, heavy suite, E2E, run once.

Lead performs checks 4-6 (Monorepo + State Sync + Commit Plan). For checks 1-3:
- Check 1 (Bug Reproduction/Regression Coverage): `qa-{TASK_ID}` Verify Report, re-verified after any fixes
- Check 2 (Cross-file Consistency): **Skipped** -- no SA in this workflow. Lead does a basic check instead (verify imports/types are consistent in changed files)
- Check 3 (Convention): Sn Dev report

All checks PASS → `shutdown_request` to both teammates → summarize for Human.

**Notes:**
- No SA in any wave -- bug fixes are scoped to existing architecture
- Sn Dev appears in both Wave 0 (impact, `opus`) and Wave 1 (root cause, `fable`) -- different agents, different missions, different models
