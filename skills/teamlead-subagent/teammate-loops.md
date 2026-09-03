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
qa-{id}: write failing tests for the assigned AC slice
  -> run them, confirm RED
  -> SendMessage dev-{id}: "tests ready" + file list + how to run
  |
dev-{id}: implement to make those tests pass (stay inside the plan's task slice)
  -> run tests locally
  -> SendMessage qa-{id}: "implemented" + changed files + test output
  |
qa-{id}: run full relevant suite + AC coverage check
  -> PASS  -> write QA Verify Report -> SendMessage main (Lead) with report -> stop
  -> FAIL  -> SendMessage dev-{id}: exact failing tests + expected vs actual   (round 2, then 3)
  |
after 3 failed rounds:
  qa-{id} -> SendMessage main: what fails, what was tried, suspected cause (plan gap? missing context? spec ambiguity?)
  Lead decides: add context / adjust plan / ask Human — then messages dev-{id} or qa-{id} to resume
```

Frontend-only or other non-backend scope follows the plan's risk-based test strategy: QA may start with tests or with a verification checklist, but the message protocol and the 3-round cap are the same.

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
เขียน test สำหรับ AC: {AC list หรือ path ไป spec.md section}
ตาม test strategy ใน `.state/{TASK_ID}/plan.md` — backend/API/service tests ต้อง FAIL ก่อน (ยังไม่มี implementation)

## Context
- Repo: {repo path} — `cd {repo}` ก่อนทำงาน
- Spec: `.state/{TASK_ID}/spec.md`   Plan: `.state/{TASK_ID}/plan.md`
- Design: `.state/{TASK_ID}/wave-1-design.md`   Impact: `.state/{TASK_ID}/wave-0-impact.md`
- Test convention: {path/ตัวอย่าง test ที่มีอยู่ — verify แล้ว}
- Test command: `{command}`

## ข้อจำกัด
- ทำงานเฉพาะใน {repo} เท่านั้น แก้ได้เฉพาะ test files
- ห้ามแก้ implementation code — ถ้าเจอบั๊ก ส่งให้ dev-{TASK_ID}

## Partner
- `dev-{TASK_ID}` — คุณเริ่มก่อน: test RED แล้ว SendMessage บอก dev-{TASK_ID} ว่า test อยู่ไฟล์ไหน รันยังไง
- เมื่อ dev-{TASK_ID} แจ้งว่า implement แล้ว → รัน suite ที่เกี่ยวข้อง + ตรวจ AC coverage
  - ผ่าน → เขียน QA Verify Report (format ใน review-domains.md) แล้ว SendMessage ถึง Lead (main)
  - ไม่ผ่าน → SendMessage dev-{TASK_ID} ระบุ test ที่ fail, expected vs actual (ไม่ต้องผ่าน Lead)
- วนได้สูงสุด 3 รอบ ถ้ายังไม่ผ่านให้ SendMessage ถึง Lead สรุปว่าติดอะไร ลองอะไรไปแล้ว คิดว่าสาเหตุคืออะไร แล้วหยุดรอ
- ห้าม spawn subagent ห้ามสร้าง team

## สิ่งที่ต้องส่งกลับ (ถึง Lead)
QA Verify Report: AC coverage table, test results, regression, flagged issues
จบด้วย status เดียว: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
```

### Dev teammate

```
<!-- spawn: model=opus mode=teammate name=dev-{TASK_ID} -->
คุณคือ Dev — implement ตาม plan ให้ test ของ QA ผ่าน

## งาน
{Task N จาก plan.md — 1-2 tasks เท่านั้น, สรุป 2-5 บรรทัด}

## Context
- Repo: {repo path} — `cd {repo}` ก่อนทำงาน
- Spec: `.state/{TASK_ID}/spec.md`   Plan: `.state/{TASK_ID}/plan.md` (ทำเฉพาะ Task {N})
- Design: `.state/{TASK_ID}/wave-1-design.md`   Impact: `.state/{TASK_ID}/wave-0-impact.md`
- ไฟล์ที่คาดว่าต้องแก้: {list — verify แล้ว}
- Conventions: {inline หรือ path ไป .context/conventions.md}

## ข้อจำกัด
- ทำงานเฉพาะใน {repo} เท่านั้น ห้ามแก้ไฟล์นอก repo นี้
- ห้ามแก้ test ของ QA เพื่อให้ผ่าน — ถ้าคิดว่า test ผิด ให้ SendMessage qa-{TASK_ID} อธิบายเหตุผล
- อยู่ใน scope ของ Task {N} — ถ้าต้องแก้นอก scope ให้แจ้ง Lead ก่อน

## Partner
- `qa-{TASK_ID}` — รอข้อความ "tests ready" จาก qa-{TASK_ID} ก่อนเริ่ม implement (test ต้อง RED อยู่ตอนนั้น — ถ้ารันแล้วผ่านตั้งแต่ยังไม่ implement ให้แจ้ง qa-{TASK_ID} ว่า test ไม่ได้ทดสอบอะไร)
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
