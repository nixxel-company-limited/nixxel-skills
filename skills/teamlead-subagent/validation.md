# Prompt Validation & Validation Gate

Lead reads this file at two points:
1. **Before spawning any agent** -- run Prompt Validation Checklist (Section 1)
2. **After review wave completes** -- run Validation Gate (Section 2)

---

## 1. Prompt Validation Checklist

Run this checklist before **every** `Agent` spawn. No exceptions.

```
+-- Prompt Validation ----------------------------------------+
|                                                              |
|  MUST run tool (hard gate -- never check "in head"):         |
|  [ ] 1. Repo path exists                                     |
|        ONE `ls -d {repo}` (or fold it into item 2's command) |
|        Confirm: directory exists and is accessible            |
|  [ ] 2. Files referenced exist                               |
|        ONE `ls -la p1 p2 p3 ...` listing EVERY path in the   |
|        prompt(s) of this wave; every path must appear in the |
|        output without "No such file"                         |
|                                                              |
|  Logical check:                                              |
|  [ ] 3. Context complete                                     |
|        - Wave output files from previous waves referenced?   |
|        - Canonical spec/plan included only when this workflow |
|          required TeamLead Brainstorm/Spec/Plan gates?         |
|        - Root-cause/design/test artifacts included for        |
|          bug-fix/read-only/non-gated workflows?                |
|        - Impact Report (wave-0-impact.md) included           |
|          if this is Wave 1+?                                 |
|        - Design/AC docs included if this is impl/test wave?  |
|  [ ] 4. Domain clear                                         |
|        - Agent role specified?                                |
|        - Review domains listed explicitly (for review wave)? |
|        - Mission described in 1-2 lines?                     |
|  [ ] 5. Constraints complete                                 |
|        - Monorepo rule stated (1 agent = 1 repo)?            |
|        - Working directory specified (cd {repo})?            |
|        - Project conventions referenced or inlined?          |
|  [ ] 6. Output clear                                         |
|        - Expected deliverable specified?                      |
|        - Output format described (code? report? test files?) |
|  [ ] 7. Spawn params                                         |
|        - `model` set explicitly and matches the Model Policy |
|          table in SKILL.md for this task type?               |
|        - `name` present ONLY IF the agent edits product      |
|          files, or is QA in a Dev loop?                      |
|        - `run_in_background: true` only when there is no     |
|          `name`?                                             |
|        - Teammate prompt has a Partner section (partner      |
|          name, who starts, end-of-loop condition, 3-round    |
|          cap)?                                               |
|                                                              |
|  Pass all 7 --> spawn OK                                     |
|  Fail any   --> fix prompt first, DO NOT spawn               |
+--------------------------------------------------------------+
```

### Hard Gate Rules (Items 1-2)

**Why items 1-2 must use tools:** The root cause of prompt errors in v1 was Lead assuming paths exist without checking. A single wrong path causes the spawned agent to waste its entire context window searching or hallucinating.

**How to check:**

```
Items 1+2 -- one Bash command per wave:
  Run: ls -la {repo_path} {path1} {path2} ... {pathN}
       (every file path that appears in ANY prompt of this wave, plus the repo dir)
  PASS: every path is listed; no "No such file or directory" line
  FAIL: any path errors --> fix or remove it from the prompt, re-run, then spawn
```

**One command, not one call per file.** `ls` reports each missing path on its own line, so a single invocation gives exactly the same per-file guarantee as N separate checks -- and it keeps the file *contents* out of the Lead's context, which a `Read` would pull in. Every extra tool call re-sends the Lead's whole context, so N individual checks cost N turns for no additional safety. Use `Read` only when the Lead actually needs the content (for example to curate the section an agent should receive).

If the paths are in different repos, one `ls` per repo is fine. Paths that an agent is expected to *create* (new test files) are not checked here -- state them in the prompt as "create".

### Logical Check Rules (Items 3-7)

These are checked by Lead reviewing the prompt text. No tool required, but Lead must verify consciously -- not skip.

**Item 3 -- Context complete:**
- Work required TeamLead Brainstorm/Spec/Plan gates? Prompt must reference `.state/{taskId}/spec.md`
- Implementation/test/review wave for gated work? Prompt must reference `.state/{taskId}/plan.md`
- Bug fix, read-only, or other workflow without TeamLead Spec/Plan gates? Prompt must reference the relevant root-cause, reproduction, design, test, or research artifacts instead of forcing canonical spec/plan paths
- Feature/behavior work that required TeamLead gates? Spec must show written spec approval before spawning implementation agents
- Wave 0 completed? Prompt must reference `.state/{taskId}/wave-0-impact.md`
- Wave 1 completed? Prompt must reference the design/AC output file
- Wave N depends on Wave N-1? Previous wave output must be in the prompt
- Exception: Wave 0 itself has no prior context requirement

**Item 4 -- Domain clear:**
- For implementation agents: role + mission is enough
- For review agents: role + explicit domain list is mandatory (see review-domains.md)
- Never send a bare "review this code" prompt

**Item 5 -- Constraints complete:**
- Every prompt must contain the monorepo rule: work only in `{repo}`, do not modify files outside
- Every prompt must specify `cd {repo}` as the working directory
- Convention reference: either inline key conventions or point to `.context/conventions.md`

**Item 6 -- Output clear:**
- Dev agents: "send back: list of files changed + brief summary"
- QA agents: "send back: test files created, test run results"
- QA teammate in a Dev loop: the report goes to the Lead with `SendMessage` to `main`, not as a task result -- a teammate does not return a task result, so a prompt that only says "return a report" leaves the Lead waiting
- Review agents: "send back: review report in the format from review-domains.md"
- Research agents: "send back: findings + recommendation"
- Every agent must end with one status: `DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, or `BLOCKED`

**Item 7 -- Spawn params:**
- `model` is set explicitly on every `Agent` call and matches the Model Policy table in `SKILL.md` for this task type. Never omit it -- an omitted `model` inherits the Lead's Fable and wastes tokens on execution work
- `name` is present **if and only if** the agent edits product files (Dev, always) or is the QA working in a red/green loop with a Dev. Reviewers, researchers, impact surveys, BA/SA design, and spec reviewers get no `name`
- `run_in_background: true` is set only when there is no `name`. Teammates do not support it
- Teammate names follow the convention: `dev-{TASK_ID}`, `qa-{TASK_ID}`; cross-repo `dev-api`, `qa-api`, `dev-web`, `qa-web`
- Every teammate prompt contains a `## Partner` section naming the partner, who starts, the end-of-loop condition, and the 3-round cap before escalating to the Lead (see `teammate-loops.md`)

### TeamLead Gate Checks Before Spawn

For feature, behavior change, UX/API/data contract change, architecture-shaping refactor, or infra flow design:

1. Confirm `teamlead-brainstorm.md` was followed and the proposed design was approved.
2. Confirm `.state/{TASK_ID}/spec.md` exists and passed the Spec Quality Gate in `teamlead-spec.md`.
3. Confirm the written spec review approval is recorded.
4. Confirm `.state/{TASK_ID}/plan.md` exists and passed the Plan Quality Gate in `teamlead-plan.md`.
5. Confirm implementation prompts include the canonical spec path, canonical plan path, assigned task slice, repo boundary, expected output, and verification expectation.

Fail any item -> do not spawn. Return to the relevant TeamLead local protocol.

---

## 2. Validation Gate (Before Reporting to Human)

After the review wave completes, Lead runs the Validation Gate before delivering results to Human. This ensures all output is consistent and complete.

### Responsibility Matrix

| # | Check | Who | When | How |
|---|-------|-----|------|-----|
| 1 | **AC Coverage** | QA | During review wave | QA reports AC coverage checklist as part of their verify report |
| 2 | **Cross-file Consistency** | SA | During review wave | SA checks API contracts, imports, types are consistent across changed files |
| 3 | **Convention Check** | Sn Dev | During review wave | Sn Dev checks naming, format, response shape against project conventions |
| 4 | **Monorepo Check** | Lead | After review wave | Lead runs `git status` in each affected repo to verify modified files are in the correct repo |
| 5 | **State Sync** | Lead | After review wave | Lead runs `git status`, branch/worktree checks, and `git log` when commits are part of the requested delivery mode, and confirms every teammate has been shut down |

**Key design:** Items 1-3 are done by review wave agents as part of their normal review output. Lead only needs to collect and verify those reports. Items 4-5 are Lead's own checks using git commands.

### Validation Gate Flow

```
Review wave completes (all agents return reports)
  |
  v
Lead collects reports:
  - QA Verify Report (contains AC coverage)
  - SA Review Report (contains cross-file consistency)
  - Sn Dev Review Report (contains convention check)
  |
  v
Lead runs own checks:
  - Item 4: git status in each affected repo
    PASS: all modified files are within expected repo boundaries
    FAIL: files modified outside expected repos
  - Item 5: git status + branch/worktree check + git log --oneline -5 when commit/PR delivery was requested
    PASS: branch/worktree are correct, changed files are understood, dirty files match expected delivery mode, commit state matches the Human-requested delivery mode, and every teammate is accounted for (shut down as the last step below)
    FAIL: wrong branch/worktree, unexpected dirty files, unexplained changed files, missing commit for requested commit/PR flow, or unexpected commit when the Human did not request one
  |
  v
Combine all verdicts:
  - All applicable items PASS and N/A items documented --> verdict: PASS
      |
      v
    Send `shutdown_request` to EVERY teammate (dev-{id}, qa-{id}, and any
    cross-repo pairs), then deliver the Human Summary. Never shut a teammate
    down before the gate passes -- fixes must reach the same Dev.
  - Any FAIL --> identify failing item(s)
      |
      v
    Route the failure to the owner (Dev/QA teammates are still alive at this
    point -- do NOT re-spawn a fresh agent):
      - Lead may fix orchestration, state, or prompt issues directly
      - Product/source/test/config/migration fixes go back to the responsible
        teammate with `SendMessage dev-{id}` / `SendMessage qa-{id}`, per the
        routing table in `teammate-loops.md`
      - Re-run only the reviewer domain that failed (a fresh one-shot `opus`
        reviewer with the same prompt plus the previous findings) -- not the
        whole review wave
      - Ownership unclear or repeated failure --> escalate to Human
      |
      v
    Responsible owner fixes and re-reports
      |
      v
    Still FAIL after fix? --> Escalate to Human:
      "Validation item {N} ({description}) failed after retry.
       Agent: {role}. Issue: {detail}. Need Human decision."
```

### Human Summary Format

When Validation Gate passes (or on escalation), deliver this summary:

```markdown
## Summary -- {TASK_ID}

**Workflow**: {WF-N} ({workflow description})
**Validation**: {PASS | FAIL -- escalated} ({X}/{Y} applicable items passed; {N} N/A)

### What was done
- Wave 0: {Impact summary -- 1 line}
- Wave 1: {Design/Analysis summary -- 1 line}
- Wave 2: {Implementation loop (qa+dev pair) summary -- 1 line}
- Wave 3: {Review summary -- 1 line}

### Files changed
- {repo-name}: {list of changed files}
- {repo-name}: {list of changed files}

### Test results
- {X} passed / {Y} failed / {Z} skipped

### Validation detail
| # | Check | Verdict | Checked by | Detail |
|---|-------|---------|------------|--------|
| 1 | AC Coverage | PASS/FAIL/N/A | QA | {X}/{Y} ACs covered, regression coverage, or N/A reason |
| 2 | Cross-file Consistency | PASS/FAIL/N/A | SA | {brief finding or N/A reason} |
| 3 | Convention Check | PASS/FAIL/N/A | Sn Dev | {brief finding or N/A reason} |
| 4 | Monorepo Check | PASS/FAIL/N/A | Lead | {brief finding or N/A reason} |
| 5 | State Sync | PASS/FAIL/N/A | Lead | {brief finding or N/A reason} |

### Caveats (if any)
- {risks, manual steps required, known limitations}
```

---

## 3. Per-Workflow Validation Handling

Not all workflows use every validation item. This table specifies which items apply per workflow.

| Validation Item | WF-1 Feature (spec) | WF-2 Feature (no spec) | WF-3 Bug Fix | WF-4 Refactor | WF-5 Cross-Repo | WF-6 Research | WF-7 Infra |
|-----------------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Prompt Validation (Section 1)** | All 7 items | All 7 items | All 7 items | All 7 items | All 7 items | Items 1-2, 4, 6, 7 | All 7 items |
| **1. AC Coverage** | QA | QA | QA | QA (existing tests) | QA per repo | N/A | N/A |
| **2. Cross-file Consistency** | SA | SA | N/A | SA | SA (critical -- cross-repo contracts) | N/A | N/A |
| **3. Convention Check** | Sn Dev | Sn Dev | Sn Dev | Sn Dev | Sn Dev per repo | N/A | Sn Dev |
| **4. Monorepo Check** | Lead | Lead | Lead | Lead | Lead (critical -- multi-repo) | N/A | Lead |
| **5. State Sync** | Lead | Lead | Lead | Lead | Lead | N/A | Lead |
| **Validation Gate runs?** | YES | YES | YES | YES | YES | NO | YES |

### Per-Workflow Notes

**WF-1 / WF-2 (Feature):** Full validation. All 5 items checked. SA present in review wave handles cross-file consistency.

**WF-3 (Bug Fix):** No SA in review wave, so cross-file consistency (item 2) is N/A. QA covers AC coverage (regression test + fix verification). Sn Dev covers convention + code quality.

For WF-3, "AC Coverage" means bug validation coverage: QA maps reported symptom -> reproduction/failing regression test -> passing fixed behavior. If no canonical AC list exists, report regression coverage as `X/Y bug symptoms covered`, and mark traditional AC coverage N/A rather than inventing acceptance criteria.

**WF-4 (Refactor):** QA checks AC coverage against existing tests (no new ACs created -- verify nothing broke). SA checks that new architecture matches design. Full validation otherwise.

**WF-5 (Cross-Repo):** All items apply with heightened scrutiny. Items 2 and 4 are critical because changes span multiple repos. SA must verify API contracts match between provider (API) and consumer (frontend). Lead must verify each repo's git status independently.

**WF-6 (Research/POC):** No Validation Gate. Output is a research report, not code. Prompt Validation still applies (items 1-2 for repo paths, 4 for domain, 6 for expected output, 7 for spawn params -- researchers are one-shot `opus` background agents with no `name`) but items 3 and 5 are not applicable since there are no prior waves or constraints to enforce.

**WF-7 (Infra/Docker/CI):** No QA agent, so AC Coverage (item 1) is N/A. Sn Dev handles test coverage as an exception (see review-domains.md WF-7 section). Convention check and monorepo check still apply. Cross-file consistency (item 2) is N/A since no SA in review wave.

**N/A Denominators:** Validation summaries must count only applicable items in the denominator. Do not report `5/5` for workflows with N/A items. Example: WF-3 normally reports `4/4 applicable items passed; 1 N/A` because Cross-file Consistency is N/A.

**Validation Failure Ownership:** Lead may directly fix only orchestration, state-management, prompt wording, routing, and validation bookkeeping issues. Any failure requiring product/source code, tests, config, migrations, data model, or business behavior changes must be sent back to the responsible implementation/review agent, or escalated to Human if ownership is unclear.
