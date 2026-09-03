# Teammate Loops — Dev↔QA Pair Protocol

Lead reads this file before spawning any agent that edits product files (Dev) or any QA that works in a red/green loop with a Dev.

The point of a teammate pair is to take the Lead out of the inner loop. In v2 every round trip (QA writes test → Dev implements → QA verifies → fails → Dev fixes) went through the Lead: the Lead read each report, wrote the next prompt, and re-spawned a fresh agent that had to rediscover the code. That burned the Lead's context and the agents' tokens on re-reading. With teammates, Dev and QA keep their own context for the whole task, talk to each other directly by name, and the Lead only hears from them when the loop finishes or gets stuck.

---

## Mechanics (Claude Code)

The spawn rules (what makes a teammate, naming, no `run_in_background`, `model: "opus"`, reset/resume behaviour) are defined once in `SKILL.md` → "Spawn Mode" and "Model Policy"; this file does not repeat them. Two mechanics matter specifically for loops:

- Teammates talk to each other with `SendMessage` `to: "<name>"`, and reach the Lead as `"main"`. The Lead receives an idle notification each time a teammate stops — it never needs to poll.
- End a teammate with `SendMessage` `{"type": "shutdown_request"}` only after the Validation Gate passes (or the Human cancels), because review findings must go back to the same Dev.

---

## Loop Shape

```
Lead spawns qa-{id} and dev-{id} together (same message, both model=opus, both named)
  |
qa-{id}: write tests for the assigned batch (AC slice), following the Convention Anchor's test pattern
  -> fast tests (no DB/infra): run ONLY the new test file(s), confirm RED
  -> heavy tests (DB/infra/E2E): do NOT pre-run when failure is certain; write them and note "RED assumed: <why>"
     (exception: a bug-fix regression test is always run once to prove reproduction)
  -> SendMessage dev-{id}: "tests ready" + file list + which are fast/heavy + how to run each
  |
dev-{id}: implement to make those tests pass (stay inside the plan's task slice, match the Convention Anchor)
  -> run the fast tests for this batch; run the batch's heavy tests once, after the implementation is complete
  -> SendMessage qa-{id}: "implemented" + changed files + test output
  |
qa-{id}: run the batch's tests + the touched module's existing tests (not the whole repo)
  -> PASS  -> tell dev-{id} which commit unit is complete (per the plan's Commit Plan)
              -> dev-{id} commits that unit -> QA writes QA Verify Report (incl. test diff since RED + run scope)
              -> SendMessage main (Lead) with report -> next batch, or stop if none
  -> FAIL  -> SendMessage dev-{id}: exact failing tests + expected vs actual   (round 2, then 3)
  |
after 3 failed rounds:
  qa-{id} -> SendMessage main: what fails, what was tried, suspected cause (plan gap? missing context? spec ambiguity?)
  Lead decides: add context / adjust plan / ask Human — then messages dev-{id} or qa-{id} to resume
```

Frontend-only or other non-backend scope follows the plan's risk-based test strategy: QA may start with tests or with a verification checklist, but the message protocol and the 3-round cap are the same.

**Batches, not waves.** One pair works through the plan's batches in order without returning to the Lead between batches — the Lead hears from QA after each commit unit and at the end. Two pairs run in parallel only when the plan marks both batches parallel-safe (fast tests only, disjoint modules). Any batch with heavy tests runs alone, because two agents hitting the same DB/containers corrupt each other's results.

**Independence inside the loop.** QA owns the tests: it never weakens an assertion, drops a case, or skips a test because Dev asked. If Dev believes a test is wrong, Dev explains why; QA decides; if they still disagree, QA escalates to the Lead with both positions. Dev never edits test files; QA never edits implementation. The QA Verify Report includes the diff of the test files since RED so the Lead can see the tests were not softened.

**Full regression and E2E run once.** The pair never runs the whole suite or E2E per round. After the last batch, the Final Verification wave (see `workflows/common.md`) runs the full unit suite, then the heavy suite, then E2E, one time — QA reports failures back to Dev, and only the failing tests are re-run after the fix.

**Why cap at 3 rounds:** two agents can loop forever on a spec ambiguity that neither can resolve. Three rounds is enough to fix genuine implementation slips; anything that survives three rounds is almost always a requirements or plan problem, which is the Lead's job, not the pair's.

---

## What the Lead does while the pair runs

- Prepare the review-wave prompts (`review-domains.md`) with the known file list from the plan
- Update `.state/teamlead.json` with both teammate names and `status: running`
- Do **not** poll or send "are you done?" messages — idle notifications arrive on their own
- Do **not** read product files to "check progress" — that is what the QA report is for

When the QA report arrives with `DONE` or `DONE_WITH_CONCERNS`, the Lead reviews the diff (`git diff --stat`, then the files the plan said would change) before spawning reviewers. Dev drifting outside the planned files is the most common problem to catch here.

---

## Review findings → back to the pair

Review-wave reviewers (SA, Sn Dev) are one-shot `opus` agents that return reports. The Lead triages:

| Finding | Route |
|---------|-------|
| Code change needed (bug, security, convention, performance) | `SendMessage dev-{id}` with the exact finding + file:line; then `SendMessage qa-{id}` to re-verify after Dev reports back |
| Test gap | `SendMessage qa-{id}` to add the test; Dev only if it then fails |
| Architecture / spec deviation | Lead decides whether it is a Dev slip (route to Dev) or a spec/plan gap (revise artifact, Human approval if requirements change) |
| Out-of-domain flag | Forward to the owning role as above |

After fixes, re-run **only** the reviewer whose domain failed (a fresh one-shot `opus` reviewer with the same prompt plus the previous findings). Do not re-run the whole review wave for one finding.

---

## Cross-repo (WF-5)

One pair per repo, dependency repo first:

```
dev-api + qa-api    (API repo only)      -> loop -> QA report -> Lead verifies API tests pass
dev-web + qa-web    (frontend repo only) -> spawned only after the API pair reports DONE
```

Pairs never message across repos. If the frontend pair needs an API change, `qa-web` tells the Lead; the Lead decides whether to route it to `dev-api` (still alive) or revise the contract in the plan.

---

## Prompt Templates

Fill these in, run the Prompt Validation Checklist, then spawn both in the same message.

### QA teammate

```
<!-- spawn: model=opus mode=teammate name=qa-{TASK_ID} -->
คุณคือ QA — เขียน test จาก AC แล้ว verify งานของ Dev จนผ่าน

## งาน
เขียน test สำหรับ batch: {Batch N — AC list หรือ path ไป spec.md section} แล้วทำ batch ถัดไปตามลำดับใน plan
ตาม Test Classification ใน `.state/{TASK_ID}/plan.md`:
- fast (ไม่แตะ DB/infra): เขียนแล้ว run เฉพาะไฟล์ใหม่ให้เห็น RED ก่อน
- heavy (hit DB/infra/E2E): เขียนไว้ ไม่ต้อง run ก่อนถ้าแน่ใจว่าแดง ระบุเหตุผล; run หลัง Dev implement เสร็จ
- bug-fix regression test: run ให้เห็น RED เสมอ (หลักฐาน reproduce)

## Context
- Repo: {repo path} — `cd {repo}` ก่อนทำงาน
- Spec: `.state/{TASK_ID}/spec.md`   Plan: `.state/{TASK_ID}/plan.md` (Test Classification, Batch Order, Commit Plan)
- Design/Root cause: `.state/{TASK_ID}/wave-1-{design|root-cause}.md` (ถ้า workflow มี)   Impact: `.state/{TASK_ID}/wave-0-impact.md`
- Convention Anchor: {feature ต้นแบบ + path ไฟล์ test ตัวอย่าง — test ใหม่ต้องใช้ pattern เดียวกัน: setup, mocking, naming, โครงสร้าง}
- Test commands: fast `{command}` / heavy `{command}` (heavy ห้าม run ซ้อนกับใคร)

## ข้อจำกัด
- ทำงานเฉพาะใน {repo} เท่านั้น แก้ได้เฉพาะ test files
- ห้ามแก้ implementation code — ถ้าเจอบั๊ก ส่งให้ dev-{TASK_ID}
- **ห้ามผ่อน assertion / ลบ case / skip test ตามคำขอ Dev** — ถ้า Dev บอกว่า test ผิด ให้พิจารณาเอง ถ้ายังเห็นต่าง escalate Lead
- run test เท่าที่จำเป็น: เฉพาะ test ของ batch + test เดิมของ module ที่แตะ ไม่ run ทั้ง repo ไม่ run E2E (มี Final Verification แยกตอนท้าย)

## Partner
- `dev-{TASK_ID}` — คุณเริ่มก่อน: test พร้อม (RED สำหรับ fast) แล้ว SendMessage บอก dev-{TASK_ID} ว่า test อยู่ไฟล์ไหน ตัวไหน fast/heavy รันยังไง
- เมื่อ dev-{TASK_ID} แจ้งว่า implement แล้ว → รัน test ของ batch + test เดิมของ module ที่แตะ + ตรวจ AC coverage
  - ผ่าน → บอก dev-{TASK_ID} ว่า commit unit {ตาม Commit Plan} เสร็จให้ commit → เขียน QA Verify Report (format ใน review-domains.md, แนบ diff ของ test files นับจาก RED + scope ที่ run) → SendMessage ถึง Lead (main) → เริ่ม batch ถัดไปทันที ไม่ต้องรอ Lead
  - ไม่ผ่าน → SendMessage dev-{TASK_ID} ระบุ test ที่ fail, expected vs actual (ไม่ต้องผ่าน Lead)
- วนได้สูงสุด 3 รอบต่อ batch ถ้ายังไม่ผ่านให้ SendMessage ถึง Lead สรุปว่าติดอะไร ลองอะไรไปแล้ว คิดว่าสาเหตุคืออะไร แล้วหยุดรอ
- ห้าม spawn subagent ห้ามสร้าง team

## สิ่งที่ต้องส่งกลับ (ถึง Lead ต่อ commit unit และตอนจบ)
QA Verify Report: AC coverage table, test results + scope ที่ run และเหตุผล, regression ของ module ที่แตะ, test diff since RED, flagged issues, รายการ case ที่ auto test ไม่ได้และต้องให้ Human ทดสอบเอง (พร้อมเหตุผล)
จบด้วย status เดียว: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
```

### Dev teammate

```
<!-- spawn: model=opus mode=teammate name=dev-{TASK_ID} -->
คุณคือ Dev — implement ตาม plan ให้ test ของ QA ผ่าน

## งาน
{Batch N จาก plan.md — task ที่อยู่ใน batch นี้, สรุป 2-5 บรรทัด} แล้วทำ batch ถัดไปตาม Batch Order เมื่อ QA ผ่าน

## Context
- Repo: {repo path} — `cd {repo}` ก่อนทำงาน
- Spec: `.state/{TASK_ID}/spec.md`   Plan: `.state/{TASK_ID}/plan.md` (Batch Order, Commit Plan, Test Classification)
- Design/Root cause: `.state/{TASK_ID}/wave-1-{design|root-cause}.md` (ถ้า workflow มี)   Impact: `.state/{TASK_ID}/wave-0-impact.md`
- ไฟล์ที่คาดว่าต้องแก้: {list — verify แล้ว}
- Convention Anchor: {feature ต้นแบบ + path ตัวอย่างต่อชั้น (route/schema/service/repository/component/text) — โค้ดใหม่ต้องมี structure, naming, error shape, UI text pattern เหมือนนี้}
- Conventions: {inline หรือ path ไป .context/conventions.md / CLAUDE.md}

## ข้อจำกัด
- ทำงานเฉพาะใน {repo} เท่านั้น ห้ามแก้ไฟล์นอก repo นี้
- ห้ามแก้ test ของ QA เพื่อให้ผ่าน — ถ้าคิดว่า test ผิด ให้ SendMessage qa-{TASK_ID} อธิบายเหตุผล QA เป็นคนตัดสิน
- อยู่ใน scope ของ batch — ถ้าต้องแก้นอก scope ให้แจ้ง Lead ก่อน
- run test เท่าที่จำเป็น: fast test ของ batch ระหว่างแก้; heavy test ของ batch run ครั้งเดียวหลัง implement เสร็จ ห้าม run ทั้ง repo ห้าม run E2E
- **Commit:** เมื่อ QA แจ้งว่า commit unit เสร็จ → `git add` เฉพาะไฟล์ของ unit นั้น (ไม่รวม .state/) → commit ด้วย message ตาม Commit Plan (`type(scope): ขั้นตอนที่ทำ` + body บอก test ที่ครอบคลุม) → ไม่ push

## Partner
- `qa-{TASK_ID}` — รอข้อความ "tests ready" จาก qa-{TASK_ID} ก่อนเริ่ม implement (fast test ต้อง RED อยู่ตอนนั้น — ถ้ารันแล้วผ่านตั้งแต่ยังไม่ implement ให้แจ้ง qa-{TASK_ID} ว่า test ไม่ได้ทดสอบอะไร; heavy test ที่ QA ระบุ `RED assumed` ไม่ต้อง pre-run)
- implement เสร็จ + รัน test เองแล้ว → SendMessage qa-{TASK_ID}: ไฟล์ที่แก้ + ผล test
- ถ้า qa-{TASK_ID} แจ้งว่า fail → แก้แล้วแจ้งกลับ qa-{TASK_ID} โดยตรง (ไม่ต้องผ่าน Lead)
- วนได้สูงสุด 3 รอบ — ถ้ารอบที่ 3 ยังไม่ผ่าน หยุดแก้ แล้ว SendMessage ถึง Lead (main) สรุปว่าติดอะไร ลองอะไรไปแล้ว คิดว่าเป็น spec/plan gap ตรงไหน (qa-{TASK_ID} จะ escalate ด้วยเช่นกัน)
- หลังผ่านแล้ว Lead อาจส่ง review findings มาให้แก้เพิ่ม — แก้แล้วแจ้ง qa-{TASK_ID} ให้ re-verify
- ห้าม spawn subagent ห้ามสร้าง team

## สิ่งที่ต้องส่งกลับ (ถึง Lead เมื่อ QA ผ่านแล้ว)
รายการไฟล์ที่แก้ + สรุปสั้นๆ + verification command ที่รัน
จบด้วย status เดียว: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
```

### Solo Dev (WF-7 Infra, no QA)

Same Dev template with the Partner section replaced by: "ไม่มี QA คู่ — เสร็จแล้ว SendMessage ถึง Lead; Lead จะส่ง review findings จาก Sn Dev มาให้แก้ถ้ามี" Still a teammate (named), because reviewer findings need to reach the same Dev.

---

## Anti-Patterns

- Spawning Dev as a one-shot and then trying to `SendMessage` it review findings — there is no name to address, so the Lead ends up re-spawning a fresh Dev that re-reads everything.
- Making reviewers or researchers teammates "just in case". They return one report; a live context window sitting idle is pure cost.
- Lead reading every Dev↔QA message and replying — the pair was spawned to own that loop. Intervene only on escalation or after the QA report.
- Letting the pair run past 3 rounds. If they are still looping, the problem is upstream (spec/plan), and only the Lead can fix that.
- Forgetting `shutdown_request` at the end. Idle teammates keep the team alive until the session ends.
- Running the whole suite or E2E inside the loop. Heavy tests cost minutes each; the loop runs only the batch's tests and the touched module's tests, and Final Verification runs everything once.
- Two pairs running heavy tests at the same time. Shared DB/containers make both results meaningless — batches with heavy tests are sequential.
- Pre-running a heavy test that is certain to fail just to "see RED". Write it, implement, then run it once. (Fast unit tests are the opposite: RED costs a second and proves the test is real.)
- QA softening a test to end a loop. That converts a failing feature into a passing report; the Lead reads the test diff since RED precisely to catch this.
- One commit per loop round. Commit units follow the plan's Commit Plan — a readable step of the feature, which may span several tasks and rounds.
