# Task Content Templates

Templates สำหรับ task content บน Notion — แยกตาม mode

---

## Template 1: Mode 2 Task Content (สั้น — Context + Objective เท่านั้น)

ใช้ตอนสร้าง tasks ใน Mode 2 (WBS Breakdown)

**หลักการ:** Task ขนาดใหญ่ 5-13 SP — content แค่ Context + Objective พอ ไม่ต้องลง Technical Design / AC / Implementation Details เพราะจะใหญ่เกินจำเป็น ถ้าต้องการเจาะลึก → ใช้ Mode 3

```markdown
## Context
[ทำไมต้องทำ task นี้ — 1-2 ประโยค บอกปัญหา/ความต้องการ/บริบทผู้ใช้]

## Objective
1. [Objective 1 — สิ่งที่ต้องทำ 1 ประโยค]
2. [Objective 2 — สิ่งที่ต้องทำ 1 ประโยค]
3. [Objective 3 — สิ่งที่ต้องทำ 1 ประโยค]
```

### ตัวอย่าง

**Task name:** Implement Login flow (UI + API + JWT)

```markdown
## Context
ระบบยังไม่มีการ authentication ผู้ใช้ — ต้องสามารถ login/logout เข้าระบบได้
ก่อนใช้ feature อื่น ส่งผลกระทบกับทุก module ในระบบ

## Objective
1. สร้าง Login UI พร้อม validation (email/password)
2. สร้าง API endpoint สำหรับ authenticate + ออก JWT token
3. Handle session/token lifecycle (login → refresh → logout)
```

### Summary property (Notion)

ตั้ง property `Summary` (text field) เป็น bullet points สั้นๆ ใช้ `<br>` คั่นบรรทัด:

```
• Login UI + validation
• JWT auth API
• Session lifecycle
```

---

## Template 2: Mode 3 Task Content (เต็มรูปแบบ — Context/Objective/Technical + AC + Gherkin)

ใช้ตอนเขียน task ใหม่ใน Mode 3 (Single Task Deep Dive)

**Title format:** `[Platform/Screen] : Task Name`
เช่น `[CS/BO] : ระบบจัดเรียง Attribute(Dimension) ส่วนของ Filter`

```markdown
## Task Description

### Context
[ทำไมต้องทำ task นี้ — อธิบายปัญหาหรือที่มา 2-3 ประโยค ลึกกว่า Mode 2]

### Objective
1. [Objective 1]
2. [Objective 2]
3. [Objective 3]

### Technical Details
[รายละเอียดเทคนิคที่ developer ต้องรู้ — จัดเป็นหัวข้อย่อย]

**[หัวข้อ 1 เช่น DB Schema Changes]**
- [รายละเอียด]

**[หัวข้อ 2 เช่น API Endpoint]**
- [รายละเอียด]

**[หัวข้อ 3 เช่น UI Component]**
- [รายละเอียด]

## Acceptance Criteria
- [ ] [AC 1 — happy path]
- [ ] [AC 2 — happy path]
- [ ] [AC 3 — edge case]
- [ ] [AC 4 — edge case]
- [ ] [AC 5 — error handling]
- [ ] [AC 6 — migration / backward compat ถ้ามี]

## Scenario

### Scenario 1: [ชื่อ scenario]
**Given** [เงื่อนไขเริ่มต้น]
**When** [action ที่ทำ]
**Then** [ผลลัพธ์ที่คาดหวัง]

### Scenario 2: [ชื่อ scenario]
**Given** [เงื่อนไขเริ่มต้น]
**When** [action ที่ทำ]
**Then** [ผลลัพธ์ที่คาดหวัง]

### Scenario 3: [edge case scenario]
**Given** [เงื่อนไขเริ่มต้น]
**When** [action ที่ทำ]
**Then** [ผลลัพธ์ที่คาดหวัง]

## Out of Scope
- [สิ่งที่ไม่ทำใน task นี้ — ป้องกัน scope creep]

## Related
- [Link ไป task/ticket ที่เกี่ยวข้อง]
```

### หลักการเขียน Mode 3

- **กระชับ** — ไม่อธิบายสิ่งที่ developer รู้อยู่แล้ว
- **Context → Objective → Technical** — why → what → how
- **AC ครอบคลุม** — happy + edge + error + migration
- **Gherkin Scenario** — 3-5 scenarios ครอบคลุม use case สำคัญ (ไม่ต้องทำทุก AC เป็น scenario)
- **Out of Scope** — ระบุชัด ป้องกัน scope creep
- **ไม่ใส่ code/schema** — เว้นแต่ผู้ใช้ขอ
- **ภาษาไทยเป็นหลัก** — ยกเว้น technical terms, Gherkin keywords (Given/When/Then), field/API names

---

## Template 3: Sprint Page Content

ใช้เมื่อสร้าง Sprint page

```markdown
## Sprint Goal
[1-2 ประโยค บอกเป้าหมายหลักของ Sprint นี้]

## Definition of Done
- [ ] All tasks moved to "Done"
- [ ] Code reviewed & merged
- [ ] Tests passing
- [ ] Demo recorded / shown
- [ ] [เกณฑ์เฉพาะของ Sprint นี้]

## Tasks
[Notion จะ auto-link จาก relation]

## Notes
- [ข้อสังเกตหรือ context สำคัญ]
```

---

## Template 4: Stakeholder Questions List

ใช้เมื่อพบ Hidden Requirements / Ambiguities ที่ต้องถาม stakeholder

```markdown
📋 คำถามสำหรับ Stakeholder

เรื่อง: [Feature Name]

### 1. ระบบหลังบ้าน
- Admin ต้องสร้าง/จัดการ [feature] ผ่านหน้าจอไหม หรือทำผ่าน database โดยตรง?
- ต้องมีระบบ approve ก่อน publish ไหม?

### 2. Business Rules
- [feature A] กับ [feature B] ใช้พร้อมกันได้ไหม?
- มี limit ต่อ user ไหม? (จำกัดกี่ครั้ง/วัน/เดือน)
- ถ้า refund เกิดขึ้น [feature] จะจัดการอย่างไร?

### 3. Monitoring
- Admin ต้องดู report อะไรบ้าง?
- ต้อง export data ได้ไหม?

### 4. Integration
- ต้องเชื่อมกับระบบไหนบ้าง?
- มี webhook / event ที่ต้อง fire ไหม?

---
กรุณาตอบแล้วส่งกลับมา จะได้นำไปอัพเดท requirement ต่อ
```

---

## Template 5: Sprint Summary Report

```markdown
📊 Sprint Summary: [Sprint Name]
📅 ระยะเวลา: [start] → [end]

## ภาพรวม
✅ Done: X tasks (Y SP)
🔄 In Progress: X tasks (Y SP)
⏳ Waiting: X tasks (Y SP) — In review / Testing / etc.
📋 Backlog: X tasks (Y SP)
🚫 Blocked: X tasks
❌ Rejected: X tasks

📈 Progress: X/Y tasks (XX%)
📊 Points: X/Y SP (XX%)

## 🔥 High Priority Pending
- [task name] — [status] — [assignee]

## ⛔ Blockers
- [task name] blocked by [task name]

## 📝 Notes
- [ข้อสังเกต]

## Action Items
1. [item 1]
2. [item 2]
```
