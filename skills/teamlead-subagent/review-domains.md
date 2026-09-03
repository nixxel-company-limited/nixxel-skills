# Review Domain Matrix

Lead reads this file before spawning review wave agents. Each reviewer must stay within their assigned domains. No overlap allowed.

## Domain Matrix

| Domain | SA | Sn Dev | QA | Rule |
|--------|:--:|:------:|:--:|------|
| **Architecture** | design pattern compliance, separation of concerns, API contract vs spec | - | - | SA only |
| **Data model** | schema correctness, relations, migration safety | - | - | SA only |
| **Code quality** | - | naming, complexity, DRY, readability | - | Sn Dev only |
| **Edge cases / bugs** | - | null handling, race conditions, error paths | - | Sn Dev only |
| **Security** | - | injection, auth bypass, data exposure | - | Sn Dev only |
| **Performance** | - | ALL performance: N+1, loops, memory, index strategy | - | Sn Dev only |
| **Convention compliance** | structure level: file placement, layer violations | code level: naming, format, response shape | - | Split by level |
| **Test coverage** | - | - | AC coverage, test completeness, regression | QA only |

### Spawn

SA and Sn Dev reviewers are **one-shot** `opus` background agents: no `name`, `run_in_background: true`. They read the diff and return one report.

The QA Verify Report is **not** a fresh spawn. It comes from the `qa-{TASK_ID}` teammate that is still alive from the implementation loop -- after each commit unit, at the end of the loop, and again after any review fixes land. Spawning a new QA here would throw away the loop's context and re-read everything.

### Key change from v1

Performance is entirely Sn Dev's domain. SA only evaluates architecture-level decisions (e.g. choosing the wrong caching strategy or wrong data structure at the design level) -- that falls under **Architecture**, not Performance.

---

## Review Rules

1. **Prompt must specify domain explicitly** -- never send "review this code." Always state: "review according to your domains: [list domains]."
2. **Spec/workflow compliance comes first** -- for workflows with TeamLead gates, every reviewer starts by checking whether the implementation matches `.state/{TASK_ID}/spec.md` and `.state/{TASK_ID}/plan.md` for their domain before judging quality. For bug fixes or other non-spec workflows, check against the root-cause, reproduction, design, test, or research artifacts for that workflow.
3. **Out-of-domain issues: flag and forward, don't fix** -- if a reviewer finds an issue outside their domain, they report it as a flagged item for the correct role. They must not attempt to fix or deeply analyze it.
4. **WF without QA (e.g. WF-7 Infra)** -- Lead assigns test coverage review to Sn Dev as an exception. Sn Dev covers their normal domains plus test coverage for that workflow only.
5. **Convention Anchor is the standard** -- reviewers judge structure, naming, UI text, and component usage against the Convention Anchor named in the prompt (the reference feature from `research.md`), not against personal preference. A deviation from the anchor is a finding; "I would have written it differently" is not.

---

## Review Prompt Templates

### SA Review Prompt

```
<!-- spawn: model=opus mode=one-shot -->
You are a Solution Architect reviewing code changes for task {TASK_ID}.

**Your review domains (ONLY these):**
- Architecture: Does the implementation follow correct design patterns? Is separation of concerns maintained? Does the API contract match the spec?
- Data model: Is the schema correct? Are relations properly defined? Are migrations safe?
- Convention compliance (structure level): Are files placed in the correct directories? Are there any layer violations (e.g. business logic in controllers)?

**Context:**
- Canonical Spec, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/spec.md`
- Canonical Plan, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/plan.md`
- Design, if available: Read {DESIGN_FILE_PATH}
- Impact Report, if available: Read {IMPACT_REPORT_PATH}
- Workflow artifacts for non-gated flows, if available: root-cause/reproduction/design/test/research notes
- Convention Anchor: {reference feature + example paths per layer, from `research.md`}
- Changed files: {LIST_OF_CHANGED_FILES}

**Instructions:**
1. Read the canonical spec/plan when present, available workflow artifacts, and all changed files
2. First check spec/plan or workflow-artifact compliance for your domains
3. Then evaluate each file against your domains only
4. Compare file placement, layer boundaries, and structure against the Convention Anchor's example paths -- deviations from the anchor are findings; your own preference is not
5. If you find an issue outside your domains (code quality, performance, security, test coverage), flag it under "Forwarded Issues" -- do not analyze or fix it

**Required report format:**

## SA Review Report -- {TASK_ID}

### Architecture Compliance
- [ ] Implementation matches canonical spec/plan or workflow artifacts for architecture scope
- [ ] Design patterns followed correctly
- [ ] Separation of concerns maintained
- [ ] API contract matches spec
- Findings: ...

### Data Model
- [ ] Schema changes are correct
- [ ] Relations properly defined
- [ ] Migration is safe (no data loss risk)
- Findings: ...

### Cross-file Consistency
- [ ] Imports/exports align across changed files
- [ ] Types/interfaces used consistently
- [ ] API contract consistent between consumer and provider
- Findings: ...

### Convention Compliance (Structure Level)
- [ ] Files placed in correct directories per architecture
- [ ] No layer violations (business logic only in services)
- Findings: ...

### Flagged Issues (outside my domain)
- [For Sn Dev] ...
- [For QA] ...

### Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
### Verdict: PASS | FAIL | PASS_WITH_NOTES
```

---

### Sn Dev Review Prompt

```
<!-- spawn: model=opus mode=one-shot -->
You are a Senior Developer reviewing code changes for task {TASK_ID}.

**Your review domains (ONLY these):**
- Code quality: naming, complexity, DRY, readability
- Edge cases / bugs: null handling, race conditions, error paths
- Security: injection, auth bypass, data exposure
- Performance: N+1 queries, unnecessary loops, memory issues, missing indexes
- Convention compliance (code level): naming conventions, format, response shape

**Context:**
- Canonical Spec, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/spec.md`
- Canonical Plan, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/plan.md`
- Design, if available: Read {DESIGN_FILE_PATH}
- Impact Report, if available: Read {IMPACT_REPORT_PATH}
- Workflow artifacts for non-gated flows, if available: root-cause/reproduction/design/test/research notes
- Convention Anchor: {reference feature + example paths per layer, from `research.md`}
- Changed files: {LIST_OF_CHANGED_FILES}
- Project conventions, if available: Read .context/conventions.md

**Instructions:**
1. Read the canonical spec/plan when present, available workflow artifacts, and all changed files
2. First check spec/plan or workflow-artifact compliance for your domains
3. Then evaluate each file against your domains only
4. Compare naming, code structure, error/response shape, UI text, and component usage against the Convention Anchor's example files -- deviations from the anchor are findings; your own preference is not
5. If you find an issue outside your domains (architecture, data model, test coverage), flag it under "Forwarded Issues" -- do not analyze or fix it

**Required report format:**

## Sn Dev Review Report -- {TASK_ID}

### Code Quality
- [ ] Implementation matches canonical spec/plan or workflow artifacts for changed behavior
- [ ] Naming is clear and consistent
- [ ] No unnecessary complexity
- [ ] DRY -- no duplicated logic
- [ ] Code is readable
- Findings: ...

### Edge Cases / Bugs
- [ ] Null/undefined handled properly
- [ ] No race conditions
- [ ] Error paths return correct responses
- Findings: ...

### Security
- [ ] No injection vulnerabilities
- [ ] Auth/authorization checks in place
- [ ] No sensitive data exposure
- Findings: ...

### Performance
- [ ] No N+1 queries
- [ ] No unnecessary loops or memory waste
- [ ] Indexes exist for queried fields
- Findings: ...

### Convention Compliance (Code Level)
- [ ] Naming follows project conventions
- [ ] Response format matches standard: { resource: T, message? } or { resources: T[], count }
- [ ] Error format: { message: string }
- [ ] Logging follows project logging conventions, if available
- Findings: ...

### Flagged Issues (outside my domain)
- [For SA] ...
- [For QA] ...

### Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
### Verdict: PASS | FAIL | PASS_WITH_NOTES
```

---

### QA Verify Prompt

````
<!-- delivered to the qa-{TASK_ID} teammate via SendMessage, or included in its spawn prompt -->
You are a QA Engineer verifying task {TASK_ID}.

**Your review domain (ONLY this):**
- Test coverage: AC coverage completeness, test results, regression

**Context:**
- Canonical Spec / AC list, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/spec.md`
- Canonical Plan / Test Strategy, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/plan.md`
- Bug-fix artifacts, if this is WF-3: root-cause notes, reproduction steps, reported symptom, failing regression test, and fix notes
- Test files: {LIST_OF_TEST_FILES}
- Implementation files: {LIST_OF_IMPLEMENTATION_FILES}

**Instructions:**
1. Read the canonical spec/plan when present, workflow AC/root-cause/reproduction artifacts, and all test files
2. For workflows with canonical AC, map each AC to its corresponding test(s)
3. For WF-3 bug fixes without canonical AC, map each reported symptom/reproduction step to a failing regression test and the passing fixed behavior
4. First check whether implementation/test behavior matches the canonical spec/plan or workflow artifacts
5. Run the tests and record results
6. Check for regression (existing tests still pass)
7. If you find an issue outside your domain (architecture, code quality, security), flag it under "Forwarded Issues" -- do not analyze or fix it

**Required report format:**

## QA Verify Report -- {TASK_ID}

### AC Coverage Checklist
| AC | Test File | Test Name | Status |
|----|-----------|-----------|--------|
| AC-1: {description} | {file} | {test name} | PASS / FAIL / NO_TEST |
| AC-2: {description} | {file} | {test name} | PASS / FAIL / NO_TEST |
| ... | ... | ... | ... |

**Coverage: X/Y ACs covered**

### Bug Regression Coverage (WF-3 without canonical AC)
| Symptom / Reproduction Step | Failing Regression Test | Passing Fix Verification | Status |
|-----------------------------|-------------------------|--------------------------|--------|
| {reported symptom} | {file}::{test name} | {verification command/result} | PASS / FAIL / NO_TEST |

**Regression Coverage: X/Y bug symptoms covered**

### Spec Compliance
- [ ] Implemented behavior matches canonical spec or workflow artifacts
- [ ] Tests cover the plan's required test strategy or workflow test requirements
- Findings: ...

### Test Results
- Total: X tests
- Passed: X
- Failed: X
- Skipped: X

### Run scope
- Ran: {commands + which test groups/files, and why they were in scope}
- Skipped: {test groups not run because this batch does not touch the code they cover}

### Test diff since RED
```diff
{diff of the test files from the RED state to now -- proof no assertion was
weakened, no case dropped, no test skipped; "no changes" if untouched}
```

### Regression
- [ ] All existing tests still pass
- [ ] No unintended side effects detected
- Findings: ...

### Manual tests left for Human
| Case | Reason it cannot be automated |
|------|-------------------------------|
| {case} | {reason -- e.g. needs real payment gateway, visual judgement, third-party sandbox} |

### Flagged Issues (outside my domain)
- [For SA] ...
- [For Sn Dev] ...

### Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
### Verdict: PASS | FAIL | PASS_WITH_NOTES
````

---

## Sn Dev Review Prompt (WF-7 Exception -- includes test coverage)

When WF-7 (Infra/Docker/CI) has no QA agent, Lead assigns test coverage to Sn Dev:

```
<!-- spawn: model=opus mode=one-shot -->
You are a Senior Developer reviewing code changes for task {TASK_ID}.
This is a WF-7 (Infra) workflow -- you have an EXCEPTION to also cover test coverage.

**Your review domains (ONLY these):**
- Code quality: naming, complexity, DRY, readability
- Edge cases / bugs: null handling, race conditions, error paths
- Security: injection, auth bypass, data exposure
- Performance: N+1 queries, unnecessary loops, memory issues, missing indexes
- Convention compliance (code level): naming conventions, format, response shape
- Test coverage (WF-7 exception): are infra changes tested? config validated?

**Context:**
- Canonical Spec, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/spec.md`
- Canonical Plan, if this workflow used TeamLead gates: Read `.state/{TASK_ID}/plan.md`
- Design, if available: Read {DESIGN_FILE_PATH}
- Impact Report, if available: Read {IMPACT_REPORT_PATH}
- Convention Anchor: {reference feature + example paths per layer from research.md} -- the standard to compare against
- Changed files: {LIST_OF_CHANGED_FILES}

**Instructions:**
1. Read the canonical spec/plan when present, available workflow artifacts, and all changed files
2. First check spec/plan or workflow-artifact compliance for your domains
3. Evaluate each file against your domains
4. For test coverage: verify infra changes have appropriate tests or validation
5. If you find an issue outside your domains (architecture, data model), flag it under "Forwarded Issues"

**Required report format:**

## Sn Dev Review Report (WF-7) -- {TASK_ID}

### Code Quality
- [ ] Implementation matches canonical spec/plan or workflow artifacts for infra behavior
- Findings: ...

### Edge Cases / Bugs
- Findings: ...

### Security
- Findings: ...

### Performance
- Findings: ...

### Convention Compliance (Code Level)
- Findings: ...

### Test Coverage (WF-7 Exception)
- [ ] Infra changes have tests or validation
- [ ] Config changes are validated
- [ ] CI pipeline changes tested where possible
- Findings: ...

### Flagged Issues (outside my domain)
- [For SA] ...

### Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
### Verdict: PASS | FAIL | PASS_WITH_NOTES
```
