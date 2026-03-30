# Prompt Validation & Validation Gate

Lead reads this file at two points:
1. **Before spawning any agent** -- run Prompt Validation Checklist (Section 1)
2. **After review wave completes** -- run Validation Gate (Section 2)

---

## 1. Prompt Validation Checklist

Run this checklist before **every** `SubAgentTool` spawn. No exceptions.

```
+-- Prompt Validation ----------------------------------------+
|                                                              |
|  MUST run tool (hard gate -- never check "in head"):         |
|  [ ] 1. Repo path exists                                     |
|        Run Glob or ls on the repo directory                  |
|        Confirm: directory exists and is accessible            |
|  [ ] 2. Files referenced exist                               |
|        Run Glob/Read on EVERY file path in the prompt        |
|        Confirm: all paths resolve to real files               |
|                                                              |
|  Logical check:                                              |
|  [ ] 3. Context complete                                     |
|        - Wave output files from previous waves referenced?   |
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
|                                                              |
|  Pass all 6 --> spawn OK                                     |
|  Fail any   --> fix prompt first, DO NOT spawn               |
+--------------------------------------------------------------+
```

### Hard Gate Rules (Items 1-2)

**Why items 1-2 must use tools:** The root cause of prompt errors in v1 was Lead assuming paths exist without checking. A single wrong path causes the spawned agent to waste its entire context window searching or hallucinating.

**How to check:**

```
Item 1 -- Repo path:
  Run: Glob pattern="{repo_path}/*" or Bash: ls {repo_path}
  PASS: directory listing returns files
  FAIL: error or empty result --> wrong path, fix before spawn

Item 2 -- Referenced files:
  For EACH file path in the prompt:
    Run: Read file_path={path} (limit=1 is enough to confirm existence)
    PASS: file contents returned
    FAIL: error --> file doesn't exist, fix or remove from prompt
```

**Do not batch check.** Check each file individually. If a prompt references 5 files, run 5 Read calls.

### Logical Check Rules (Items 3-6)

These are checked by Lead reviewing the prompt text. No tool required, but Lead must verify consciously -- not skip.

**Item 3 -- Context complete:**
- Wave 0 completed? Prompt must reference `.claude/state/{taskId}/wave-0-impact.md`
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
- Convention reference: either inline key conventions or point to `.claude/context/conventions.md`

**Item 6 -- Output clear:**
- Dev agents: "send back: list of files changed + brief summary"
- QA agents: "send back: test files created, test run results"
- Review agents: "send back: review report in the format from review-domains.md"
- Research agents: "send back: findings + recommendation"

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
| 5 | **State Sync** | Lead | After review wave | Lead runs `git status` + `git log` to verify all changes are committed and branches are correct |

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
  - Item 5: git status + git log --oneline -5
    PASS: changes committed, branch matches task branch
    FAIL: uncommitted changes or wrong branch
  |
  v
Combine all verdicts:
  - All 5 PASS --> verdict: PASS --> report to Human
  - Any FAIL --> identify failing item(s)
      |
      v
    SendMessage back to the failing agent:
      "Your review found issue X. Fix it and report back."
      (or for Lead's own items: fix directly)
      |
      v
    Agent fixes and re-reports
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
**Validation**: {PASS | FAIL -- escalated} ({X}/{Y} items passed)

### What was done
- Wave 0: {Impact summary -- 1 line}
- Wave 1: {Design/Analysis summary -- 1 line}
- Wave 2: {Test summary -- 1 line}
- Wave 3: {Implementation summary -- 1 line}
- Wave 4: {Review summary -- 1 line}

### Files changed
- {repo-name}: {list of changed files}
- {repo-name}: {list of changed files}

### Test results
- {X} passed / {Y} failed / {Z} skipped

### Validation detail
| # | Check | Verdict | Checked by | Detail |
|---|-------|---------|------------|--------|
| 1 | AC Coverage | PASS/FAIL | QA | {X}/{Y} ACs covered |
| 2 | Cross-file Consistency | PASS/FAIL | SA | {brief finding} |
| 3 | Convention Check | PASS/FAIL | Sn Dev | {brief finding} |
| 4 | Monorepo Check | PASS/FAIL | Lead | {brief finding} |
| 5 | State Sync | PASS/FAIL | Lead | {brief finding} |

### Caveats (if any)
- {risks, manual steps required, known limitations}
```

---

## 3. Per-Workflow Validation Handling

Not all workflows use every validation item. This table specifies which items apply per workflow.

| Validation Item | WF-1 Feature (spec) | WF-2 Feature (no spec) | WF-3 Bug Fix | WF-4 Refactor | WF-5 Cross-Repo | WF-6 Research | WF-7 Infra |
|-----------------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Prompt Validation (Section 1)** | All 6 items | All 6 items | All 6 items | All 6 items | All 6 items | Items 1-2, 4, 6 | All 6 items |
| **1. AC Coverage** | QA | QA | QA | QA (existing tests) | QA per repo | N/A | N/A |
| **2. Cross-file Consistency** | SA | SA | N/A | SA | SA (critical -- cross-repo contracts) | N/A | N/A |
| **3. Convention Check** | Sn Dev | Sn Dev | Sn Dev | Sn Dev | Sn Dev per repo | N/A | Sn Dev |
| **4. Monorepo Check** | Lead | Lead | Lead | Lead | Lead (critical -- multi-repo) | N/A | Lead |
| **5. State Sync** | Lead | Lead | Lead | Lead | Lead | N/A | Lead |
| **Validation Gate runs?** | YES | YES | YES | YES | YES | NO | YES |

### Per-Workflow Notes

**WF-1 / WF-2 (Feature):** Full validation. All 5 items checked. SA present in review wave handles cross-file consistency.

**WF-3 (Bug Fix):** No SA in review wave, so cross-file consistency (item 2) is N/A. QA covers AC coverage (regression test + fix verification). Sn Dev covers convention + code quality.

**WF-4 (Refactor):** QA checks AC coverage against existing tests (no new ACs created -- verify nothing broke). SA checks that new architecture matches design. Full validation otherwise.

**WF-5 (Cross-Repo):** All items apply with heightened scrutiny. Items 2 and 4 are critical because changes span multiple repos. SA must verify API contracts match between provider (API) and consumer (frontend). Lead must verify each repo's git status independently.

**WF-6 (Research/POC):** No Validation Gate. Output is a research report, not code. Prompt Validation still applies (items 1-2 for repo paths, 4 for domain, 6 for expected output) but items 3 and 5 are not applicable since there are no prior waves or constraints to enforce.

**WF-7 (Infra/Docker/CI):** No QA agent, so AC Coverage (item 1) is N/A. Sn Dev handles test coverage as an exception (see review-domains.md WF-7 section). Convention check and monorepo check still apply. Cross-file consistency (item 2) is N/A since no SA in review wave.
