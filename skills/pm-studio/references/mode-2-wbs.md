# Mode 2: WBS Breakdown (Task-level)

แตก requirement / feature spec → Epic → Task

**สิ่งที่ Mode 2 ทำต่างจากของเดิม:**
- **Task ใหญ่ขึ้น (5-13 SP)** — ไม่แตกย่อยเกินไป ไม่แยก UI/API/DB เป็น 3 tasks
- **Task content สั้น** — แค่ **Context + Objective** ไม่มี Technical Design / AC / Implementation Details
- **ถ้าต้องการเจาะลึก → ใช้ Mode 3 แทน**

---

## Workflow Overview

```
Input + Analysis → GATE-3 Epic list confirm → loop {แสดง WBS ทีละ Epic → GATE-4 confirm} → WBS Summary → GATE-5 next step
```

---

## Section 1: Task Granularity Rule (สำคัญที่สุดของ Mode นี้)

### กฎเหล็ก 3 ข้อ

1. **1 Task = 1 Deliverable ใช้งานได้ครบ** — รวม UI + API + DB ในตัวเดียว ไม่แตกแยก
2. **ขนาด 5-13 SP** — ถ้าเล็กกว่า 5 SP รวมกับ task ใกล้เคียง ถ้าใหญ่กว่า 13 SP แตกย่อย
3. **1 Task = 1 PR ที่ reviewer ดูแล้วเข้าใจ end-to-end**

### ตัวอย่าง: ทำผิด vs ทำถูก

**❌ ผิด (แตกย่อยเกิน):**
```
- Task 1.1: Create Login UI (3 SP)
- Task 1.2: Create Login API endpoint (3 SP)
- Task 1.3: Create Login DB schema (2 SP)
- Task 1.4: Add JWT token handling (3 SP)
```

**✅ ถูก (รวมเป็น deliverable):**
```
- Task 1.1: Implement Login flow (UI + API + JWT) (8 SP)
```

**❌ ผิด:**
```
- Task: Add field "phone" to User table (1 SP)
- Task: Update Profile UI to show phone (2 SP)
- Task: Add phone validation (2 SP)
```

**✅ ถูก:**
```
- Task: Add phone number to User Profile (UI + DB + validation) (5 SP)
```

### เมื่อไหร่ควรแยก task

**แยกได้** เมื่อ:
- ขนาดรวมเกิน 13 SP (large feature ที่ใหญ่จริงๆ)
- มี dependency ชัดเจนระหว่าง 2 deliverables (ทำพร้อมกันไม่ได้)
- เป็น Feature vs Chore ที่ภาพรวมเหมือนแต่ scope แยกกัน (เช่น "Implement RBAC" vs "Migrate existing users to RBAC")

### INVEST Check

แต่ละ Task ที่แตกออกมา ต้องผ่าน:
- **I**ndependent — ทำได้โดยไม่ต้องรอ task อื่น (หรือระบุ dependency ชัด)
- **N**egotiable — scope ต่อรองได้
- **V**aluable — ส่งมอบ value ให้ user หรือ system
- **E**stimable — ประเมิน effort ได้
- **S**mall — ≤ 13 SP
- **T**estable — มี AC ที่ทดสอบได้ (แม้ Mode 2 จะไม่เขียน AC ก็ต้องคิดได้)

---

## Section 2: Epic Structure (อย่าลืมระบบหลังบ้าน)

เมื่อแตก Epic ต้องพิจารณา **ทุกฝั่ง** ของระบบ:

```
Epic ที่ต้องมีเสมอ (ถ้าเกี่ยวข้อง):
├── Epic: [Feature] — Front-end (user-facing)
├── Epic: [Feature] — Back-office / Admin Panel     ← มักถูกลืม!
├── Epic: [Feature] — Configuration & Settings      ← มักถูกลืม!
├── Epic: [Feature] — Reporting & Analytics          ← มักถูกลืม!
└── Epic: Infrastructure / Shared (DB, API, Auth)
```

### หลักการออกแบบ Epic (MECE)

1. **Mutually Exclusive** — ทุก task ต้องจัด Epic เดียว ไม่ซ้อนทับ
2. **Collectively Exhaustive** — ทุก task ต้องมี Epic ไม่มี orphan
3. **Domain-driven** — แบ่งตาม business domain ไม่ใช่ technical layer
4. **Balanced size** — แต่ละ Epic ควรมี tasks สมดุล (5-30 tasks ต่อ Epic)

---

## Section 3: GATE-3 — Confirm Epic List ก่อนแตก Task

**สำคัญ:** แสดง Epic overview ก่อนเสมอ ห้ามแตก task ทันที

```markdown
📋 Epic Overview: [Feature/Project Name]

| # | Epic | ฝั่ง | Scope สรุป | ~Tasks |
|---|------|-----|-----------|--------|
| 1 | [ชื่อ] | 👤 User | [1 บรรทัด] | ~X |
| 2 | [ชื่อ] | 🔧 Admin | [1 บรรทัด] | ~X |
| 3 | [ชื่อ] | ⚙️ Config | [1 บรรทัด] | ~X |
| 4 | [ชื่อ] | 📊 Report | [1 บรรทัด] | ~X |
| 5 | [ชื่อ] | 🏗️ Infra | [1 บรรทัด] | ~X |
```

```
question: "Epic ที่แบ่งมา X กลุ่มนี้โอเคไหมครับ? (รวมระบบหลังบ้านแล้ว)"
header: "Epic Review"
options:
  - label: "โอเค ไปต่อ"
    description: "Epic ถูกต้อง ไปแตก Task ทีละ Epic ได้เลย"
  - label: "ปรับ Epic"
    description: "อยากเพิ่ม/ลด/เปลี่ยนชื่อ/รวม Epic บางตัว"
  - label: "แตกละเอียดกว่านี้"
    description: "อยากให้ Epic ย่อยกว่านี้"
  - label: "รวมให้น้อยลง"
    description: "Epic เยอะเกิน อยากรวมให้กระชับ"
```

ถ้าเลือก "ปรับ" → ปรับ → แสดงใหม่ → ถามอีกรอบ (loop)

---

## Section 4: GATE-4 — แสดง WBS ทีละ Epic (loop)

**สำคัญ:** แสดงทีละ Epic ไม่ใช่ทั้งหมดทีเดียว

สำหรับแต่ละ Epic:

```markdown
### Epic 1: [ชื่อ Epic] (👤 User)
| # | Task | Context (สั้นๆ) | Priority | SP | Type |
|---|------|----------------|----------|-----|------|
| 1.1 | [ชื่อ Task] | [1 ประโยค ทำไมต้องมี] | High | 8 | Feature |
| 1.2 | [ชื่อ Task] | [...] | Medium | 5 | Feature |
| 1.3 | [ชื่อ Task] | [...] | Medium | 5 | Chore |
```

**Type:** Feature, Bug, Chore

**Priority:**
- High = core feature, blocker, ผลกระทบมาก
- Medium = สำคัญแต่ไม่ block
- Low = nice-to-have, polish, optimization

**SP:** ใช้ Modified Fibonacci (1, 2, 3, 5, 8, 13) — ตามกฎ Mode 2 ส่วนใหญ่จะเป็น 5, 8, 13

```
question: "Epic 1: [ชื่อ] — X tasks นี้โอเคไหมครับ?"
header: "Epic 1 Review"
options:
  - label: "โอเค → Epic ถัดไป"
    description: "Tasks ถูกต้อง ไปดู Epic ถัดไปได้เลย"
  - label: "ปรับ tasks"
    description: "อยากเพิ่ม/ลด/แก้ task บางตัว"
  - label: "แตกย่อยกว่านี้"
    description: "บาง task ยังใหญ่เกิน อยากให้แตก"
  - label: "รวมให้น้อยลง"
    description: "Tasks เยอะเกิน อยากรวมบางตัว"
```

**Loop:** ปรับ → แสดงใหม่ → ถามอีกรอบ จนกว่าจะ "โอเค"

ทำซ้ำสำหรับทุก Epic จนครบ

---

## Section 5: WBS Summary (หลัง Epic สุดท้าย)

```markdown
## WBS Summary: [Feature/Project Name]

| Epic | ฝั่ง | Tasks | High | Med | Low | SP รวม |
|------|-----|-------|------|-----|-----|--------|
| [Epic 1] | 👤 | X | X | X | X | YY |
| [Epic 2] | 🔧 | X | X | X | X | YY |
| **รวม** | | **Y** | **X** | **X** | **X** | **ZZ** |
```

---

## Section 6: GATE-5 — Next Step

```
question: "WBS เสร็จแล้ว — ต้องการทำอะไรต่อครับ?"
header: "Next Step"
multiSelect: true
options:
  - label: "สร้างบน Notion"
    description: "สร้าง Tasks ทั้งหมดเป็น pages บน Notion"
  - label: "Effort Estimation"
    description: "ประเมิน Story Points ละเอียดให้แต่ละ Task (ดู cross-cutting.md)"
  - label: "Dependency Map"
    description: "วิเคราะห์ dependency chain + risk (ดู cross-cutting.md)"
  - label: "เสร็จแค่นี้"
    description: "ได้ WBS table ก็พอ"
```

---

## Section 7: สร้าง Tasks บน Notion

ถ้าผู้ใช้เลือก "สร้างบน Notion":

### Step 1: ตรวจ config
- ตรวจว่ามี `notion.backlog_datasource` ยัง — ถ้าไม่มีให้ถาม URL
- ใช้ `notion-fetch` ดึง schema จริงของ database (อย่าสมมติ properties)

### Step 2: Confirm

```
question: "ต้องการสร้าง Tasks ทั้งหมด X ตัว บน Notion — confirm?"
header: "Create on Notion"
options:
  - label: "สร้างทั้งหมด"
    description: "สร้าง X tasks ใน Notion backlog"
  - label: "เลือกบาง Epic"
    description: "สร้างเฉพาะบาง Epic ก่อน"
  - label: "ยกเลิก"
    description: "ไม่สร้าง เก็บแค่ WBS table"
```

### Step 3: สร้างทีละ task

ใช้ `notion-create-pages` กับ backlog data source

**Task properties:**
- Task name: ชื่อ task
- Status: "Backlog"
- Task type: Feature / Bug / Chore
- Priority: High / Medium / Low
- Epic: (ตาม Epic ที่แตกไว้ — ต้องตั้งเสมอ)
- Estimate Point: SP
- Effort level: Small (1-3), Medium (5-8), Large (13)

**Task content:** ใช้ **Mode 2 Task Content Template** จาก `templates.md` — แค่ Context + Objective

### Step 4: Verify

หลังสร้างเสร็จ:
- Spot-check 2-3 tasks สุ่ม
- ตรวจ orphan (ไม่มี Epic)
- แสดงสรุป:

```
✅ สร้างเสร็จ: X/Y tasks
[ลิงก์ไป Notion board]
```
