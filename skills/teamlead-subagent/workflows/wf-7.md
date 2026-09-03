Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-7: Infra / Docker / CI

**Total waves: 3** (Wave 0 through Wave 2 + Validation Gate)

### Wave 0 -- Impact Check

**Agents:**
- **Sn Dev** (one-shot, `opus`, background) -- design approach + impact check for infrastructure changes
- No SA -- infra workflows follow the Decision Table (no SA for infra)

`opus`, not `fable`: infra design here has a narrow scope and follows the repo's existing CI/Docker/config conventions rather than choosing a new architecture.

**Context for agents:**
- New infra flow/policy changes: canonical spec `.state/{TASK_ID}/spec.md` and canonical plan `.state/{TASK_ID}/plan.md`
- Mechanical maintenance (for example version bumps, config cleanup, or non-policy CI edits): raw infra task description from Human is sufficient when there is no new flow, policy, contract, or architecture decision
- Current infra files and existing CI/Docker/config conventions

**Lead action after wave:**
- Save Impact Report to `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Implementation (solo Dev teammate)

**Agents:**
- **`dev-{TASK_ID}`** (teammate, `opus`) -- implement the infrastructure changes

No QA partner in this workflow, so there is no pair loop: Dev implements, verifies with the commands named in the plan, and sends its summary to the Lead. It is still a **teammate** and not a one-shot, because the Wave 2 Sn Dev review findings have to reach the same Dev without re-reading the whole infra setup. Use the "Solo Dev" template in `teammate-loops.md`.

**Context for agents:**
- New infra flow/policy changes: `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`
- `wave-0-impact.md` (design + constraints from Sn Dev's impact check)
- Mechanical maintenance: raw Human task description is allowed when Wave 0 classified it as non-policy/non-flow maintenance

**Lead action after wave:**
- Dev works through the Batch Order and Commit Plan from `plan.md` **if this workflow produced one** (otherwise the batch/commit units the Lead states in the prompt) without returning to the Lead between batches; with no QA partner the Lead runs each batch's verification command itself, and Dev commits per the Commit Plan once that verification passes; the Convention Anchor from `research.md` **if it exists, otherwise the reference feature identified in Wave 0** is in the prompt
- Save implementation summary to `.state/{TASK_ID}/wave-1-implementation.md`
- Keep `dev-{TASK_ID}` alive through Wave 2 and the Validation Gate

---

### Wave 2 -- Review (per Review Domain, Sn Dev gets test coverage exception)

**Agents:**
- **Sn Dev** (one-shot, `opus`, background) -- review code quality + performance + convention compliance + **test coverage (exception)**

**Context for agents:**
- New infra flow/policy changes: `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`
- `wave-0-impact.md` (original design)
- `wave-1-implementation.md` (what was implemented)
- Changed file list from `git status`
- Use the **WF-7 Exception** review prompt from `review-domains.md`

**Important:** No QA agent in this workflow. Sn Dev receives an exception to also cover the test coverage domain. Use the dedicated WF-7 Sn Dev review prompt template from `review-domains.md`.

**Lead action after wave:**
- Collect the Sn Dev review report and save it to `.state/{TASK_ID}/wave-2-review.md`
- Findings that need a code change go back to `dev-{TASK_ID}` by `SendMessage`; there is no QA to re-verify, so the Lead re-runs the named verification command itself
- Proceed to Validation Gate

---

### Validation Gate

Preceded by the Final Verification wave (see `common.md`) -- full unit suite, heavy suite, E2E, run once. With no QA teammate, the Lead runs it and writes `final-verification.md` itself.

Lead performs checks 4-6. For checks 1-3:
- Check 1 (AC Coverage): **N/A** — no QA teammate; Sn Dev's test-coverage exception is reported under check 3
- Check 2 (Cross-file Consistency): **Skipped** -- no SA in this workflow. Lead does a basic check
- Check 3 (Convention): Sn Dev report

All checks PASS → `shutdown_request` to `dev-{TASK_ID}` → summarize for Human.

**Notes:**
- No QA in any wave -- infra changes are reviewed by Sn Dev with test coverage exception
- No SA in any wave -- infra follows Decision Table
- Sn Dev appears in both Wave 0 (impact/design) and Wave 2 (review) -- two different one-shot agents with different missions, both `opus`
- The single Dev is the only teammate in this workflow
- Shortest workflow with Validation Gate (3 waves)
- New infra flows and policy changes must consume canonical spec/plan. Mechanical maintenance may use raw scope only when it does not introduce new flow, policy, contract, or architecture decisions.
