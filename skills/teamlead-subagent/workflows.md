# Workflows (WF-1 to WF-7)

> **Before reading this file:** Lead must read these sub-files first:
> - `state-management.md` -- state file schema, resume flow, output file conventions
> - `validation.md` -- Prompt Validation Checklist (run before every spawn) + Validation Gate (run after final review wave)
> - `review-domains.md` -- Review Domain Matrix + review prompt templates (required for every review wave)

This file contains the complete wave-by-wave definition of all 7 workflows. Lead selects a workflow from the Decision Table in SKILL.md, then follows the matching WF here step-by-step.

---

## Wave 0 -- Research/Impact Check

### Rules

**Mandatory Wave 0:** All workflows must start with Wave 0, EXCEPT the cases below.

**Skip Wave 0 when ALL 4 criteria are met:**
1. Change affects a single file only
2. No schema change
3. No new dependency added
4. No API contract change

All 4 must be true to skip. If even one is false, Wave 0 is mandatory.

**Always skip Wave 0:** WF-6 (Research/POC) -- the workflow itself is research, Wave 0 would be redundant.

### Who to spawn in Wave 0

Refer to the Decision Table in SKILL.md:

| Workflow | SA in Decision Table? | Wave 0 agents |
|----------|:---------------------:|---------------|
| WF-1 Feature (spec) | Yes | SA + Sn Dev (parallel) |
| WF-2 Feature (no spec) | Yes | SA + Sn Dev (parallel) |
| WF-3 Bug Fix | No | Sn Dev only |
| WF-4 Refactor | Yes | SA + Sn Dev (parallel) |
| WF-5 Cross-Repo | Yes | SA + Sn Dev (parallel) |
| WF-6 Research | -- | No Wave 0 |
| WF-7 Infra | No | Sn Dev only |

### Wave 0 Output

Each agent produces an Impact Report using this template:

```markdown
## Impact Report -- {Role}

### Files/Modules Affected
- path/to/file.ts -- reason it is affected
- ...

### Risks / Blockers
- [BLOCKER] ... (must resolve before starting)
- [RISK] ... (proceed with caution)

### Constraints
- ...

### Recommendations
- ...
```

**Lead action after Wave 0:**
1. Collect Impact Reports from all Wave 0 agents
2. Merge into a single file: `.state/{TASK_ID}/wave-0-impact.md`
3. Pass `wave-0-impact.md` as context to all agents in the next wave

---

## Validation Gate

Every workflow (except WF-6) ends with a Validation Gate after the final review wave.

### Who checks what

| Check | Who | Method |
|-------|-----|--------|
| 1. AC Coverage | **QA** (in review wave) | QA reports AC coverage as part of their verify report |
| 2. Cross-file Consistency | **SA** (in review wave) | SA checks API contract, imports, types match across files/repos |
| 3. Convention Check | **Sn Dev** (in review wave) | Sn Dev checks naming, format, response shape |
| 4. Monorepo Check | **Lead** | Run `git status` -- verify modified files are in the correct repo(s) |
| 5. State Sync | **Lead** | Run `git status` + `git log` -- verify all changes are committed |

Checks 1-3 are performed by agents during the review wave (they include these in their review reports). Lead only performs checks 4-5 after collecting all review reports.

### Gate Flow

```
Review wave completes
  |
Lead collects QA + SA + Sn Dev review reports
  |
Lead runs git status/log for checks 4-5
  |
All 5 checks PASS --> summarize for Human with verdict: PASS
Any check FAIL --> SendMessage back to the failing agent to fix
  |
Agent fails again on re-check --> escalate to Human
```

### Summary format for Human

```
## Summary -- {TASK_ID}

**Workflow**: {WF-X} ({description})
**Validation**: PASS (5/5) | FAIL (X/5)

### What was done
- Wave 0: {impact summary}
- Wave 1: {wave 1 summary}
- Wave 2: {wave 2 summary}
- ...

### Files changed
- {repo}: {file list}

### Test Results
- X passed / Y failed

### Validation Detail
- AC Coverage: PASS/FAIL (QA) -- details
- Cross-file Consistency: PASS/FAIL (SA) -- details
- Convention: PASS/FAIL (Sn Dev) -- details
- Monorepo: PASS/FAIL (Lead) -- details
- State Sync: PASS/FAIL (Lead) -- details

### Things to know (if any)
- {risks, caveats, manual steps}
```

---

## WF-1: Feature (has spec/PRD)

**Total waves: 5** (Wave 0 through Wave 4 + Validation Gate)

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** -- survey schema, modules, dependencies that may be affected
- **Sn Dev** -- survey codebase, environment, technical feasibility

**Context for agents:** Task description + spec/PRD from Human

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Analysis + Design (parallel)

**Agents:**
- **BA** -- analyze spec, produce AC (Acceptance Criteria) list
- **SA** -- design architecture: components, API contracts, data model changes

**Context for agents:**
- `wave-0-impact.md` (from Wave 0)
- Original spec/PRD from Human

**Lead action after wave:**
- Merge BA's AC list + SA's design into `.state/{TASK_ID}/wave-1-design.md`

---

### Wave 2 -- Write Tests

**Agents:**
- **QA** -- write tests from AC list; tests MUST FAIL at this point (no implementation yet)

**Context for agents:**
- `wave-1-design.md` (AC list + architecture design)
- `wave-0-impact.md` (affected files for test targeting)

**Lead action after wave:**
- Save QA's test plan/file list to `.state/{TASK_ID}/wave-2-tests.md`
- Verify tests exist and fail as expected

---

### Wave 3 -- Implementation

**Agents:**
- **Dev** -- write code to make QA's tests pass

**Context for agents:**
- `wave-1-design.md` (architecture design + AC list)
- `wave-2-tests.md` (test file locations + what they test)
- `wave-0-impact.md` (constraints, risks)

**Lead action after wave:**
- Save implementation summary to `.state/{TASK_ID}/wave-3-implementation.md`
- Verify Dev stayed within the designed architecture

---

### Wave 4 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **QA** -- run tests, verify AC coverage, check regression (domain: test coverage)
- **SA** -- review architecture compliance + cross-file consistency (domains: architecture, data model, convention-structure)
- **Sn Dev** -- review code quality + performance + convention compliance (domains: code quality, edge cases, security, performance, convention-code)

**Context for agents:**
- `wave-1-design.md` (spec/design to review against)
- `wave-3-implementation.md` (what was implemented)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect all 3 review reports
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5 (Monorepo + State Sync), combines with agent reports for checks 1-3, then summarizes for Human.

**Notes:**
- This is the standard feature workflow when a spec/PRD already exists
- Wave 1 parallelizes BA + SA because they work independently (BA reads spec, SA reads spec + impact)
- Wave 4 parallelizes all 3 reviewers since their domains do not overlap

---

## WF-2: Feature (no spec)

**Total waves: 6** (Wave 0 through Wave 5 + Validation Gate)

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** -- survey architecture that may be affected
- **Sn Dev** -- survey codebase + technical feasibility

**Context for agents:** Task description + raw requirements from Human

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Create Spec

**Agents:**
- **BA** -- write spec + AC from the requirements Human provided

**Context for agents:**
- `wave-0-impact.md` (from Wave 0)
- Raw requirements from Human

**Lead action after wave:**
- Save BA's spec + AC list to `.state/{TASK_ID}/wave-1-spec.md`

---

### Wave 2 -- Design

**Agents:**
- **SA** -- design architecture based on BA's spec

**Context for agents:**
- `wave-1-spec.md` (BA's spec + AC)
- `wave-0-impact.md` (affected modules, constraints)

**Lead action after wave:**
- Save SA's design to `.state/{TASK_ID}/wave-2-design.md`

---

### Wave 3 -- Write Tests

**Agents:**
- **QA** -- write tests from AC; tests MUST FAIL

**Context for agents:**
- `wave-1-spec.md` (AC list)
- `wave-2-design.md` (architecture design)
- `wave-0-impact.md` (affected files)

**Lead action after wave:**
- Save test plan to `.state/{TASK_ID}/wave-3-tests.md`
- Verify tests fail as expected

---

### Wave 4 -- Implementation

**Agents:**
- **Dev** -- write code to make QA's tests pass

**Context for agents:**
- `wave-2-design.md` (architecture design)
- `wave-3-tests.md` (test files + what they test)
- `wave-1-spec.md` (AC list for reference)
- `wave-0-impact.md` (constraints, risks)

**Lead action after wave:**
- Save implementation summary to `.state/{TASK_ID}/wave-4-implementation.md`

---

### Wave 5 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **QA** -- run tests, verify AC coverage, check regression (domain: test coverage)
- **SA** -- review architecture compliance + cross-file consistency (domains: architecture, data model, convention-structure)
- **Sn Dev** -- review code quality + performance + convention compliance (domains: code quality, edge cases, security, performance, convention-code)

**Context for agents:**
- `wave-2-design.md` (design to review against)
- `wave-1-spec.md` (AC list for coverage check)
- `wave-4-implementation.md` (what was implemented)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect all 3 review reports
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5, combines with agent reports for checks 1-3, summarizes for Human.

**Notes:**
- Extra wave compared to WF-1 because BA must create spec first (Wave 1), then SA designs from it (Wave 2) -- these are sequential, not parallel
- Wave 0 still runs parallel (SA + Sn Dev impact) because impact check does not need the spec

---

## WF-3: Bug Fix

**Total waves: 5** (Wave 0 through Wave 4 + Validation Gate)

### Wave 0 -- Impact Check

**Agents:**
- **Sn Dev** only -- survey code related to the bug + impact check
- No SA -- bug fixes do not require architecture review

**Context for agents:** Bug description + reproduction steps from Human

**Lead action after wave:**
- Save Impact Report to `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Root Cause Analysis

**Agents:**
- **Sn Dev** -- analyze root cause of the bug

**Context for agents:**
- `wave-0-impact.md` (affected files, related modules)
- Bug description from Human

**Lead action after wave:**
- Save root cause analysis to `.state/{TASK_ID}/wave-1-root-cause.md`

---

### Wave 2 -- Write Regression Test

**Agents:**
- **QA** -- write test that reproduces the bug; test MUST FAIL (bug not fixed yet)

**Context for agents:**
- `wave-1-root-cause.md` (what causes the bug, where it occurs)
- `wave-0-impact.md` (affected files)

**Lead action after wave:**
- Save test info to `.state/{TASK_ID}/wave-2-tests.md`
- Verify test reproduces the bug (fails)

---

### Wave 3 -- Fix

**Agents:**
- **Dev** -- fix the bug so QA's test passes

**Context for agents:**
- `wave-1-root-cause.md` (root cause analysis)
- `wave-2-tests.md` (test file locations + what they test)
- `wave-0-impact.md` (constraints, other affected areas)

**Lead action after wave:**
- Save fix summary to `.state/{TASK_ID}/wave-3-fix.md`

---

### Wave 4 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **QA** -- verify fix works + no regression + report AC coverage (domain: test coverage)
- **Sn Dev** -- review code quality + performance (domains: code quality, edge cases, security, performance, convention-code)
- No SA -- bug fix does not require architecture review

**Context for agents:**
- `wave-1-root-cause.md` (root cause for context)
- `wave-3-fix.md` (what was fixed)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect QA + Sn Dev review reports
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5 (Monorepo + State Sync). For checks 1-3:
- Check 1 (AC Coverage): QA report
- Check 2 (Cross-file Consistency): **Skipped** -- no SA in this workflow. Lead does a basic check instead (verify imports/types are consistent in changed files)
- Check 3 (Convention): Sn Dev report

Summarize for Human.

**Notes:**
- No SA in any wave -- bug fixes are scoped to existing architecture
- Sn Dev appears in both Wave 0 (impact) and Wave 1 (root cause) -- these are sequential, not the same agent session

---

## WF-4: Refactor

**Total waves: 4** (Wave 0 through Wave 3 + Validation Gate)

### Wave 0 -- Impact Check (parallel)

**Agents:**
- **SA** -- survey architecture + dependencies that refactor may affect
- **Sn Dev** -- survey current codebase + impact of proposed changes

**Context for agents:** Refactor scope/goal from Human

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- New Design

**Agents:**
- **SA** -- create the new design/architecture for the refactored code

**Context for agents:**
- `wave-0-impact.md` (affected modules, constraints, risks)
- Current code state (SA reads relevant files)

**Lead action after wave:**
- Save design to `.state/{TASK_ID}/wave-1-design.md`

---

### Wave 2 -- Implementation

**Agents:**
- **Dev** -- refactor code according to SA's new design

**Context for agents:**
- `wave-1-design.md` (new design/architecture)
- `wave-0-impact.md` (constraints, affected files)

**Lead action after wave:**
- Save implementation summary to `.state/{TASK_ID}/wave-2-implementation.md`

---

### Wave 3 -- Verify + Review (parallel, per Review Domain)

**Agents:**
- **QA** -- run existing tests, verify nothing is broken (domain: test coverage)
- **SA** -- review that refactored code follows the new design (domains: architecture, data model, convention-structure)
- **Sn Dev** -- review code quality + performance improvements (domains: code quality, edge cases, security, performance, convention-code)

**Context for agents:**
- `wave-1-design.md` (design to verify against)
- `wave-2-implementation.md` (what was refactored)
- Changed file list from `git status`
- Review prompt templates from `review-domains.md`

**Lead action after wave:**
- Collect all 3 review reports
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5, combines with agent reports for checks 1-3, summarizes for Human.

**Notes:**
- Shorter than feature workflows because there is no BA (no new spec/AC needed)
- QA verifies existing tests still pass rather than writing new tests
- Focus is on maintaining behavior while improving structure

---

## WF-5: Cross-Repo Feature (Monorepo)

**Total waves: 5** (Wave 0 through Wave 4 + Validation Gate)

### Wave 0 -- Impact Check (parallel, ALL repos)

**Agents:**
- **SA** -- survey architecture across ALL affected repos; identify cross-repo contracts
- **Sn Dev** -- survey codebase in ALL affected repos; check feasibility per repo

**Context for agents:** Task description + which repos are involved

**Lead action after wave:**
- Merge Impact Reports into `.state/{TASK_ID}/wave-0-impact.md`
- Include per-repo breakdown: which files in which repo

---

### Wave 1 -- Analysis + Design (parallel)

**Agents:**
- **BA** -- analyze AC, explicitly tag each AC with its target repo (e.g. "AC-1 [prathan-api]", "AC-2 [prathan-customer]")
- **SA** -- design cross-repo architecture; define API contracts between repos (what API provides, what frontend consumes)

**Context for agents:**
- `wave-0-impact.md` (per-repo breakdown)
- Task description / spec from Human

**Lead action after wave:**
- Merge into `.state/{TASK_ID}/wave-1-design.md`
- Ensure AC-to-repo mapping is clear
- Ensure API contract between repos is explicitly defined

---

### Wave 2 -- Dependency Repo (e.g. API) -- SEQUENTIAL within wave

**Agents (sequential, not parallel):**
1. **QA-API** -- write tests for API-side AC; tests MUST FAIL
2. **Dev-API** -- implement API code to make tests pass (waits for QA-API to finish)

**Context for agents:**
- `wave-1-design.md` (API-side AC + architecture + API contract)
- `wave-0-impact.md` (API-side affected files)

**Important:** QA-API must complete before Dev-API starts. Both agents work ONLY in the API repo.

**Lead action after wave:**
- Save to `.state/{TASK_ID}/wave-2-api.md`
- Verify API is functional and tests pass before proceeding to Wave 3

---

### Wave 3 -- Consumer Repo (e.g. Frontend) -- SEQUENTIAL within wave, WAITS for Wave 2

**Agents (sequential, not parallel):**
1. **QA-Frontend** -- write tests for frontend-side AC; tests MUST FAIL
2. **Dev-Frontend** -- implement frontend code to make tests pass (waits for QA-Frontend to finish)

**Context for agents:**
- `wave-1-design.md` (frontend-side AC + architecture)
- `wave-2-api.md` (API contract that is now implemented -- frontend can consume it)
- `wave-0-impact.md` (frontend-side affected files)

**Important:** Wave 3 MUST NOT start until Wave 2 is fully complete (frontend depends on API). Both agents work ONLY in the frontend repo.

**Lead action after wave:**
- Save to `.state/{TASK_ID}/wave-3-frontend.md`

---

### Wave 4 -- Verify + Review (parallel, per Review Domain, SEPARATE agents per repo)

**Agents:**
- **QA-API** -- verify API tests pass + AC coverage (domain: test coverage, API repo only)
- **QA-Frontend** -- verify frontend tests pass + AC coverage (domain: test coverage, frontend repo only)
- **SA** -- review architecture + cross-repo API contract consistency (domains: architecture, data model, convention-structure, cross-repo consistency)
- **Sn Dev-API** -- review code quality in API repo (domains: code quality, edge cases, security, performance, convention-code; API repo only)
- **Sn Dev-Frontend** -- review code quality in frontend repo (domains: code quality, edge cases, security, performance, convention-code; frontend repo only)

**Context for agents:**
- `wave-1-design.md` (design + AC + API contract)
- `wave-2-api.md` + `wave-3-frontend.md` (implementation summaries)
- Changed file list from `git status` (filtered per repo for each agent)
- Review prompt templates from `review-domains.md`

**Important:** SA is the only agent that reviews ACROSS repos (checking contract consistency). All other agents are scoped to a single repo.

**Lead action after wave:**
- Collect all 5 review reports
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5. Check 4 (Monorepo) is especially critical here -- verify no agent touched files outside their assigned repo.

Combine with agent reports for checks 1-3, summarize for Human.

**Notes:**
- Wave 2 and Wave 3 are sequential (API first, then frontend) because frontend depends on API
- Within Wave 2 and Wave 3, QA writes tests first, then Dev implements -- also sequential
- Wave 4 parallelizes all reviewers since they have non-overlapping domains and repos
- Each repo gets its own QA, Dev, and Sn Dev agent -- enforce Monorepo Rule (1 agent = 1 repo)
- SA is the exception: SA reviews cross-repo consistency but does NOT modify code
- If more than 2 repos are involved, add more waves (Wave 2 for dependency repo, Wave 3 for next dependency, Wave 4 for leaf consumer, etc.) -- always dependency-first order

---

## WF-6: Research / POC

**Total waves: 1** (Wave 1 only, NO Wave 0, NO Validation Gate)

### NO Wave 0

This workflow skips Wave 0 entirely. The workflow itself IS research -- a separate impact check would be redundant.

---

### Wave 1 -- Research

**Agents:**
- **Sn Dev** -- research the topic, analyze options, produce recommendation

**Context for agents:** Research question / POC scope from Human

**Lead action after wave:**
- Return Sn Dev's recommendation directly to Human for decision
- No further waves -- Human decides next steps

---

### NO Validation Gate

No code changes are produced, so Validation Gate does not apply.

**Notes:**
- Simplest workflow -- single wave, single agent
- Output goes directly to Human for decision-making
- If Human decides to proceed with implementation after research, Lead starts a new workflow (WF-1, WF-2, etc.) using the research output as input context

---

## WF-7: Infra / Docker / CI

**Total waves: 3** (Wave 0 through Wave 2 + Validation Gate)

### Wave 0 -- Impact Check

**Agents:**
- **Sn Dev** only -- design approach + impact check for infrastructure changes
- No SA -- infra workflows follow the Decision Table (no SA for infra)

**Context for agents:** Infra task description from Human

**Lead action after wave:**
- Save Impact Report to `.state/{TASK_ID}/wave-0-impact.md`

---

### Wave 1 -- Implementation

**Agents:**
- **Dev** -- implement the infrastructure changes

**Context for agents:**
- `wave-0-impact.md` (design + constraints from Sn Dev's impact check)
- Task description from Human

**Lead action after wave:**
- Save implementation summary to `.state/{TASK_ID}/wave-1-implementation.md`

---

### Wave 2 -- Review (per Review Domain, Sn Dev gets test coverage exception)

**Agents:**
- **Sn Dev** -- review code quality + performance + convention compliance + **test coverage (exception)**

**Context for agents:**
- `wave-0-impact.md` (original design)
- `wave-1-implementation.md` (what was implemented)
- Changed file list from `git status`
- Use the **WF-7 Exception** review prompt from `review-domains.md`

**Important:** No QA agent in this workflow. Sn Dev receives an exception to also cover the test coverage domain. Use the dedicated WF-7 Sn Dev review prompt template from `review-domains.md`.

**Lead action after wave:**
- Collect Sn Dev review report
- Proceed to Validation Gate

---

### Validation Gate

Lead performs checks 4-5. For checks 1-3:
- Check 1 (AC Coverage): Covered by Sn Dev (exception)
- Check 2 (Cross-file Consistency): **Skipped** -- no SA in this workflow. Lead does a basic check
- Check 3 (Convention): Sn Dev report

Summarize for Human.

**Notes:**
- No QA in any wave -- infra changes are reviewed by Sn Dev with test coverage exception
- No SA in any wave -- infra follows Decision Table
- Sn Dev appears in both Wave 0 (impact/design) and Wave 2 (review) -- these are different sessions with different missions
- Shortest workflow with Validation Gate (3 waves)

---

## Quick Reference -- Workflow Wave Counts

| WF | Name | Waves | Wave 0 | Validation Gate | Agents involved |
|----|------|:-----:|:------:|:---------------:|-----------------|
| WF-1 | Feature (spec) | 5 | SA + Sn Dev | Yes | SA, BA, QA, Dev, Sn Dev |
| WF-2 | Feature (no spec) | 6 | SA + Sn Dev | Yes | SA, BA, QA, Dev, Sn Dev |
| WF-3 | Bug Fix | 5 | Sn Dev only | Yes | QA, Dev, Sn Dev |
| WF-4 | Refactor | 4 | SA + Sn Dev | Yes | SA, QA, Dev, Sn Dev |
| WF-5 | Cross-Repo | 5 | SA + Sn Dev | Yes | SA, BA, QA-per-repo, Dev-per-repo, Sn Dev-per-repo |
| WF-6 | Research/POC | 1 | None | No | Sn Dev |
| WF-7 | Infra/Docker/CI | 3 | Sn Dev only | Yes | Dev, Sn Dev (with test exception) |
