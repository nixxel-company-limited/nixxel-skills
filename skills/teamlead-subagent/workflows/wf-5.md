Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-5: Cross-Repo Feature (Monorepo)

**Total waves: 5** (Wave 0 through Wave 4 + Validation Gate)

### Wave 0 -- Impact Check (parallel, ALL repos)

**Agents:**
- **SA** (one-shot, `opus`, background) -- survey architecture across ALL affected repos; identify cross-repo contracts
- **Sn Dev** (one-shot, `opus`, background) -- survey codebase in ALL affected repos; check feasibility per repo

**Context for agents:**
- Canonical spec: `.state/{TASK_ID}/spec.md`
- Canonical plan: `.state/{TASK_ID}/plan.md`
- Repo list and raw Human request as background only

**Cross-repo exception:** Wave 0 permits read-only cross-repo impact/contract review by SA and Sn Dev because the work is analysis only. These agents must not edit code, format files, run write-producing commands, or make repo-local implementation changes. Implementation remains one agent per repo.

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`
- Include per-repo breakdown: which files in which repo

---

### Wave 1 -- Analysis + Design (parallel)

**Agents:**
- **BA** (one-shot, `fable`, background) -- analyze AC, explicitly tag each AC with its target repo (e.g. "AC-1 [prathan-api]", "AC-2 [prathan-customer]")
- **SA** (one-shot, `fable`, background) -- design cross-repo architecture; define API contracts between repos (what API provides, what frontend consumes)

Both run on `fable`. The cross-repo contract defined here is what two independent pairs will build against in Waves 2 and 3, and getting it wrong means redoing both.

**Context for agents:**
- `wave-0-impact.md` (per-repo breakdown)
- `.state/{TASK_ID}/spec.md` (canonical requirements)
- `.state/{TASK_ID}/plan.md` (canonical implementation plan)
- Raw Human request as background only

**Lead action after wave:**
- Merge into `.state/{TASK_ID}/wave-1-design.md`
- Ensure AC-to-repo mapping is clear
- Ensure API contract between repos is explicitly defined

---

### Wave 2 -- Dependency Repo (e.g. API) -- Dev↔QA pair, API repo only

Read `teammate-loops.md` before spawning. Both teammates are spawned **in the same message**.

**Agents:**
- **`qa-api`** (teammate, `opus`) -- write tests for API-side AC; tests MUST FAIL first, then message `dev-api`
- **`dev-api`** (teammate, `opus`) -- implement API code so those tests pass

**Loop:** QA RED → Dev implements → QA verifies. PASS → `qa-api` sends its QA Verify Report to the Lead. FAIL → `qa-api` messages `dev-api` directly, max 3 rounds, then escalate to the Lead.

**Context for agents:**
- `wave-1-design.md` (API-side AC + architecture + API contract)
- `wave-0-impact.md` (API-side affected files)
- `.state/{TASK_ID}/spec.md` + `.state/{TASK_ID}/plan.md` (canonical, API-side scope)

**Important:** both teammates work ONLY in the API repo (`cd {api repo}`), and they message **only each other and the Lead** -- never the frontend pair.

**Lead action after wave:**
- The pair works through the plan's Batch Order without returning to the Lead between batches; `dev-api` commits per the Commit Plan after `qa-api` PASS; the Convention Anchor from `research.md` is in every prompt
- Save the QA Verify Report + Dev summary to `.state/{TASK_ID}/wave-2-api.md`
- Verify the API is functional and its tests pass before spawning Wave 3
- Keep `dev-api` and `qa-api` alive -- Wave 4 findings and any contract change from the frontend pair go back to them

---

### Wave 3 -- Consumer Repo (e.g. Frontend) -- Dev↔QA pair, WAITS for Wave 2

**Agents:**
- **`qa-web`** (teammate, `opus`) -- write tests for frontend-side AC according to `.state/{TASK_ID}/plan.md`; failing tests first are required only when the plan/risk profile calls for them
- **`dev-web`** (teammate, `opus`) -- implement frontend code against the now-implemented API contract

**Loop:** same protocol and 3-round cap as Wave 2.

**Context for agents:**
- `wave-1-design.md` (frontend-side AC + architecture)
- `wave-2-api.md` (API contract that is now implemented -- frontend can consume it)
- `wave-0-impact.md` (frontend-side affected files)

**Important:** Wave 3 MUST NOT be spawned until the Wave 2 QA report comes back `DONE` (frontend depends on API). Both teammates work ONLY in the frontend repo. If the frontend pair needs an API change, `qa-web` tells the **Lead**; the Lead decides whether to route it to `dev-api` (still alive) or revise the contract in the plan. Pairs never message across repos.

**Lead action after wave:**
- The pair works through the plan's Batch Order without returning to the Lead between batches; `dev-web` commits per the Commit Plan after `qa-web` PASS; the Convention Anchor from `research.md` is in every prompt
- Save the QA Verify Report + Dev summary to `.state/{TASK_ID}/wave-3-frontend.md`
- Keep both pairs alive through Wave 4 and the Validation Gate

---

### Wave 4 -- Review (parallel, per Review Domain, SEPARATE agents per repo)

**Agents:**
- **SA** (one-shot, `opus`, background) -- review architecture + cross-repo API contract consistency (domains: architecture, data model, convention-structure, cross-repo consistency)
- **Sn Dev-API** (one-shot, `opus`, background) -- review code quality in API repo (domains: code quality, edge cases, security, performance, convention-code; API repo only)
- **Sn Dev-Frontend** (one-shot, `opus`, background) -- review code quality in frontend repo (same domains; frontend repo only)

AC and test coverage for each repo come from the `qa-api` and `qa-web` Verify Reports. Do **not** spawn fresh QA agents for the review wave.

**Context for agents:**
- `wave-1-design.md` (design + AC + API contract)
- `wave-2-api.md` + `wave-3-frontend.md` (implementation summaries + both QA Verify Reports)
- Changed file list from `git status` (filtered per repo for each agent)
- Review prompt templates from `review-domains.md`

**Important:** SA is the only agent that reviews ACROSS repos (checking contract consistency). All other agents are scoped to a single repo.

**Lead action after wave:**
- Collect all 3 review reports and save them to `.state/{TASK_ID}/wave-4-review.md`
- Route each finding to the pair that owns the repo it lives in, per the "Review findings → back to the pair" table in `teammate-loops.md`
- Proceed to Validation Gate

---

### Validation Gate

Preceded by the Final Verification wave (see `common.md`) -- full unit suite, heavy suite, E2E, run once.

Lead performs checks 4-6. Check 4 (Monorepo) is especially critical here -- verify no agent touched files outside their assigned repo.

Check 1 combines the `qa-api` and `qa-web` Verify Reports (both re-verified after any fixes); checks 2-3 come from the reviewer reports. All checks PASS → `shutdown_request` to all four teammates → summarize for Human.

**Notes:**
- Wave 2 and Wave 3 are sequential (API first, then frontend) because frontend depends on API
- Within each wave the pair owns its own red/green loop -- the Lead is not in the round trips
- Wave 4 parallelizes all reviewers since they have non-overlapping domains and repos
- Each repo gets its own Dev↔QA pair and its own Sn Dev reviewer -- enforce Monorepo Rule (1 agent = 1 repo)
- Four teammates stay alive at once from Wave 3 onward. That is the cost of cross-repo work; shut them all down at the gate
- SA is the read-only exception for cross-repo consistency/contract review and does NOT modify code. Wave 0 Sn Dev may also perform read-only cross-repo feasibility review; implementation remains one agent per repo.
- If more than 2 repos are involved, add more waves (Wave 2 for dependency repo, Wave 3 for next dependency, Wave 4 for leaf consumer, etc.) -- always dependency-first order
