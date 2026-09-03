# Workflows (WF-1 to WF-7)

Read this file once, then read only `wf-N.md` for the selected workflow. Never load all seven.

> **Before reading a `wf-N.md` file, Lead must have read:**
> - `state-management.md` -- state file schema, resume flow, output file conventions
> - `validation.md` -- Prompt Validation Checklist (run before every spawn) + Validation Gate (run after final review wave)
> - `review-domains.md` -- Review Domain Matrix + review prompt templates (required for every review wave)
> - `teammate-loops.md` -- Dev↔QA teammate pair protocol, prompt templates, escalation (required before any implementation wave)
>
> Every `Agent` call in this file carries an explicit spawn spec: **one-shot** (no `name`, `run_in_background: true`) or **teammate** (has `name`, no `run_in_background`), plus an explicit `model`. See "Model Policy" and "Spawn Mode" in `SKILL.md`.

This directory holds the wave-by-wave definition of all 7 workflows, one per `wf-N.md`. Lead selects a workflow from the Decision Table in SKILL.md, reads this file for the shared rules, then follows only the matching `wf-N.md` step-by-step.

---

## Wave 0 -- Impact + Feasibility Review

### Rules

**Mandatory Wave 0:** All workflows must start with Wave 0, EXCEPT the cases below.

**Source of truth:** For workflows that pass through Brainstorm/Spec/Plan gates, Wave 0 is an impact + feasibility review of the canonical artifacts:

- `.state/{TASK_ID}/spec.md` -- canonical requirements, behavior, acceptance criteria, contracts
- `.state/{TASK_ID}/plan.md` -- canonical implementation sequence, test strategy, repo/task scope

`.state/{TASK_ID}/wave-0-impact.md` is supporting context created from the review findings. It is never the source of truth for requirements, scope, design intent, or the implementation plan.

If `.state/{TASK_ID}/research.md` exists (from the Research Gate, see `teamlead-research.md`), pass it to Wave 0 agents as supporting context so they do not re-discover what research already mapped. Like `wave-0-impact.md`, it is supporting context only — never canonical.

**Wave 0 skip is evaluated only after required gates complete.** A skip decision must never bypass Brainstorm/Spec/Plan gates. Brainstorm-required work (WF-1, WF-2, WF-5) may not skip canonical spec/plan feasibility review. The skip criteria below apply only to non-brainstorm workflows and cases that do not require canonical spec/plan review.

**Skip Wave 0 when ALL 4 criteria are met:**
1. Change affects a single file only
2. No schema change
3. No new dependency added
4. No API contract change

All 4 must be true to skip. If even one is false, Wave 0 is mandatory.

**Always skip Wave 0:** WF-6 (Research/POC) -- the workflow itself is research, Wave 0 would be redundant.

### Who to spawn in Wave 0

Refer to the Decision Table in SKILL.md:

| Workflow | SA in Decision Table? | Wave 0 agents | Spawn |
|----------|:---------------------:|---------------|-------|
| WF-1 Feature (spec) | Yes | SA + Sn Dev (parallel) | one-shot, `opus`, background |
| WF-2 Feature (no spec) | Yes | SA + Sn Dev (parallel) | one-shot, `opus`, background |
| WF-3 Bug Fix | No | Sn Dev only | one-shot, `opus`, background |
| WF-4 Refactor | Yes | SA + Sn Dev (parallel) | one-shot, `opus`, background |
| WF-5 Cross-Repo | Yes | SA + Sn Dev (parallel) | one-shot, `opus`, background |
| WF-6 Research | -- | No Wave 0 | -- |
| WF-7 Infra | No | Sn Dev only | one-shot, `opus`, background |

Wave 0 is always one-shot `opus`: every agent reads a lot of code and returns one report, and nobody needs to talk to it afterwards.

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
3. If Wave 0 finds requirement/design blockers, revise `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md`, then get Human approval again before implementation waves
4. Pass `wave-0-impact.md` as context to all agents in the next wave

### Test-first Scope Rule

Backend/API/service changes require QA to write the test before Dev implementation starts. How the test is proven depends on the plan's Test Classification:

- **fast tests** (no DB, network, container, or real infra): must be **seen RED** -- QA runs only the new test file(s), which costs a second and proves the test actually catches the behavior.
- **heavy tests** (hit DB/real infra, E2E, anything needing a container): the pre-run may be skipped when failure is certain (the endpoint does not exist yet, the bug still reproduces). Write the test, note "RED assumed: {why}", and run it once after the implementation is complete.
- **bug-fix regression test**: always run once before the fix, fast or heavy -- the red run *is* the proof of reproduction. For a heavy one, run that single test file only.
- **Heavy tests never run concurrently.** Two agents hitting the same DB/containers corrupt each other's results, so batches with heavy tests are sequential (see `teammate-loops.md`).

Frontend-only and other non-backend changes follow the risk-based test strategy in `.state/{TASK_ID}/plan.md`; QA may still write tests first, but a red test is mandatory only when the plan or risk profile calls for it.

---

## Final Verification wave

Runs **once**, after the LAST batch/commit unit of the workflow and before the Validation Gate. No E2E run and no full-suite run happens inside the implementation loops -- the pair only runs the batch's tests plus the touched module's existing tests. Final Verification is where the whole system is checked one time.

The `qa-{TASK_ID}` teammate (already alive from the implementation loop) runs, **sequentially, one time each**:

1. the full unit suite
2. the heavy / integration suite
3. E2E

Failures go back to `dev-{TASK_ID}` by `SendMessage` with the exact failing tests. After the fix, **only the failing tests are re-run** -- never the whole sequence again.

**Output:** `.state/{TASK_ID}/final-verification.md` -- what was run (commands + scope), the results of each of the three runs, and what could not be automated (cases the Human must test manually, with the reason each one cannot be automated).

WF-6 has no Final Verification (no code is produced). WF-7 has no QA teammate: the Lead runs the named verification commands itself and writes the same file.

---

## Validation Gate

Every workflow (except WF-6) ends with a Validation Gate, run after the Final Verification wave completes.

### Who checks what

| Check | Who | Method |
|-------|-----|--------|
| 1. AC Coverage | **`qa-{TASK_ID}` teammate** | The QA Verify Report the pair produced, re-verified by the same teammate after any review fixes. Never spawn a fresh QA for this |
| 2. Cross-file Consistency | **SA** (review wave, one-shot `opus`) | SA checks API contract, imports, types match across files/repos |
| 3. Convention Check | **Sn Dev** (review wave, one-shot `opus`) | Sn Dev checks naming, format, response shape |
| 4. Monorepo Check | **Lead** | Run `git status` -- verify modified files are in the correct repo(s) |
| 5. State Sync | **Lead** | Run `git status` + `git log` -- verify all changes are committed |
| 6. Commit Plan followed | **Lead** | Run `git log --oneline` on the task branch -- commits match the plan's Commit Plan units, no per-round micro-commits, no `.state/` files committed, nothing pushed unless the Human requested it |

Check 1 comes from the QA teammate, checks 2-3 from the one-shot reviewers. Lead only performs checks 4-6 after collecting all reports.

### Gate Flow

```
Review wave completes (Dev/QA teammates are still alive)
  |
Final Verification wave completes (final-verification.md written)
  |
Lead collects the QA Verify Report + SA/Sn Dev review reports
  |
Lead runs git status/log for checks 4-6
  |
Any check FAIL --> route the finding per the table below --> re-verify --> re-check
  |
All applicable checks PASS --> shutdown_request to every teammate --> summarize for Human with verdict: PASS
```

### Routing a failed check

Follow the "Review findings → back to the pair" table in `teammate-loops.md`:

| Failed check | Route |
|--------------|-------|
| 1. AC Coverage | `SendMessage qa-{TASK_ID}` to close the gap; `SendMessage dev-{TASK_ID}` only if the new test then fails |
| 2. Cross-file Consistency, 3. Convention (code change needed) | `SendMessage dev-{TASK_ID}` with the exact finding + `file:line`, then `SendMessage qa-{TASK_ID}` to re-verify once Dev reports back |
| 2-3 (architecture / spec deviation) | Lead decides: Dev slip → route to Dev; spec/plan gap → revise the artifact, Human approval if requirements change |
| 4. Monorepo, 5. State Sync | Lead's own checks -- Lead fixes the commit/repo scope or routes the stray file to the owning Dev teammate |
| 6. Commit Plan followed | Lead's own check -- a missing or mis-scoped commit goes back to `dev-{TASK_ID}` with the intended unit and message; Lead never rewrites product commits itself |

After fixes, re-run **only** the reviewer whose domain failed (a fresh one-shot `opus` reviewer with the same prompt plus the previous findings). Do not re-run the whole review wave for one finding.

**Closing the gate:** all applicable checks PASS → `SendMessage {"type": "shutdown_request"}` to every teammate spawned for this task → summarize for Human. Teammates must not be shut down before the gate passes, because review findings have to reach the same Dev.

Same domain fails twice on re-check → escalate to Human.

### Summary format for Human

```
## Summary -- {TASK_ID}

**Workflow**: {WF-X} ({description})
**Validation**: {PASS | FAIL} ({X}/{Y} applicable items passed; {N} N/A)

### What was done
- Wave 0: {impact summary}
- Wave 1: {wave 1 summary}
- Wave 2: {wave 2 summary}
- ...

### Files changed
- {repo}: {file list}

### Commits
- {hash} {message}   <- commit unit 1
- {hash} {message}   <- commit unit 2

### Test Results
- X passed / Y failed
- Final Verification: full unit suite / heavy suite / E2E -- results (see `.state/{TASK_ID}/final-verification.md`)

### Validation Detail
- AC Coverage: PASS/FAIL (QA) -- details
- Cross-file Consistency: PASS/FAIL (SA) -- details
- Convention: PASS/FAIL (Sn Dev) -- details
- Monorepo: PASS/FAIL (Lead) -- details
- State Sync: PASS/FAIL (Lead) -- details
- Commit Plan followed: PASS/FAIL (Lead) -- details

### Manual tests left for Human
- {case} -- {why it could not be automated}

### Things to know (if any)
- {risks, caveats}
```

---

## Quick Reference -- Workflow Wave Counts

Every workflow with a Validation Gate runs the Final Verification wave first.

| WF | Name | Waves | Wave 0 | Validation Gate | Spawn mode + model |
|----|------|:-----:|:------:|:---------------:|--------------------|
| WF-1 | Feature (spec) | 4 | SA + Sn Dev (one-shot `opus`) | Yes | Wave 0 SA+SnDev(opus), Wave 1 SA(fable)+BA(fable), Wave 2 qa+dev pair(opus, teammate), Wave 3 reviewers(opus) |
| WF-2 | Feature (no spec) | 4 | SA + Sn Dev (one-shot `opus`) | Yes | Wave 0 SA+SnDev(opus), Wave 1 SA(fable)+BA(fable), Wave 2 qa+dev pair(opus, teammate), Wave 3 reviewers(opus) |
| WF-3 | Bug Fix | 4 | Sn Dev only (one-shot `opus`) | Yes | Wave 0 SnDev(opus), Wave 1 SnDev root cause(fable), Wave 2 qa+dev pair(opus, teammate), Wave 3 SnDev(opus) |
| WF-4 | Refactor | 4 | SA + Sn Dev (one-shot `opus`) | Yes | Wave 0 SA+SnDev(opus), Wave 1 SA(fable), Wave 2 dev+qa pair(opus, teammate), Wave 3 reviewers(opus) |
| WF-5 | Cross-Repo | 5 | SA + Sn Dev (one-shot `opus`) | Yes | Wave 0 SA+SnDev(opus), Wave 1 SA(fable)+BA(fable), Wave 2 dev-api+qa-api pair(opus), Wave 3 dev-web+qa-web pair(opus), Wave 4 reviewers(opus) |
| WF-6 | Research/POC | 1 | None | No | Sn Dev(opus, one-shot) -- no teammates |
| WF-7 | Infra/Docker/CI | 3 | Sn Dev only (one-shot `opus`) | Yes | Wave 0 SnDev(opus), Wave 1 solo dev teammate(opus), Wave 2 SnDev review(opus, test exception) |

`fable` appears only in the design/analysis waves. Every implementation, test, research, and review spawn is `opus`. Every agent that edits product files is a named teammate; every report-only agent is a one-shot with `run_in_background: true`.
