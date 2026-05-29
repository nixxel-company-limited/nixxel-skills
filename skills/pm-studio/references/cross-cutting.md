# Cross-cutting Operations

ความสามารถที่เรียกใช้ได้จากทุก mode — Dependency, Estimation, Traceability, Impact Analysis

---

## 1. Dependency & Risk Mapping

### Step 1: ดึงข้อมูล
- `notion-fetch` Tasks ใน Sprint ที่ต้องการวิเคราะห์
- ดู `Blocked by` / `Blocking` relations
- ดู Roadmap dependencies

### Step 2: Dependency Map

```
🔗 Dependency Map: [Sprint Name]

Critical Path (ทำก่อนทั้งหมด):
  [PREFIX]-1 → [PREFIX]-3 → [PREFIX]-7 → [PREFIX]-10

Parallel Tracks (ทำพร้อมกันได้):
  Track A: [PREFIX]-2 → [PREFIX]-5
  Track B: [PREFIX]-4 → [PREFIX]-6
  Track C: [PREFIX]-8, [PREFIX]-9 (independent)

Bottlenecks:
  ⚠️ [PREFIX]-3 — block 4 tasks ถัดไป (critical!)
  ⚠️ [PREFIX]-1 — prerequisite ของทุก track

External Dependencies:
  🔌 [ระบุ external dependency] (ต้องเสร็จก่อน Sprint เริ่ม)
```

### Step 3: Risk Assessment

```
⚠️ Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [task X] ล่าช้า block ทุก task | สูง | Critical | เริ่มก่อน + daily check-in |
| [tech Y] ไม่เคยใช้ | กลาง | High | spike 1 วันก่อนเริ่ม |
| scope creep จาก [feature Z] | กลาง | Medium | lock scope ก่อน sprint |

🔴 Critical: X  🟡 High: X  🟢 Medium: X
```

### Step 4: AskUserQuestion

```
question: "Dependency map ตรงไหม? มี dependency ที่ขาดไหม?"
header: "Dep Review"
options:
  - label: "ถูกต้อง"
    description: "Dependency ครบ"
  - label: "มีเพิ่ม"
    description: "อยากเพิ่ม dependency ที่ขาด"
  - label: "ไม่แน่ใจ"
    description: "ต้องถาม tech lead — สรุปคำถามให้"
```

---

## 2. Effort Estimation

### Framework

ใช้ Modified Fibonacci: 1, 2, 3, 5, 8, 13

| SP | Effort Level | ลักษณะ | ตัวอย่าง |
|----|-------------|--------|---------|
| 1–2 | Small | config, UI tweak | เพิ่ม field ใน form |
| 3 | Small-Medium | minor feature | edit existing form |
| 5 | Medium | feature ครบ logic ไม่ซับซ้อน | CRUD พื้นฐาน |
| 8 | Large | logic ซับซ้อน หลาย component | Approval workflow |
| 13 | X-Large | feature ใหญ่ end-to-end | RBAC พื้นฐาน + UI |
| 21+ | ❌ | แตกย่อยก่อน! | ระบบ RBAC ทั้งหมด |

### Stack-specific Modifiers

ถ้ารู้ tech stack ให้พิจารณาเพิ่ม:

| ปัจจัย | Modifier | เหตุผล |
|--------|---------|--------|
| RLS / Permission policies | +2 | ซับซ้อน ยากทดสอบ |
| Real-time / WebSocket | +2 | debug ยาก |
| File upload/storage | +1 | edge cases เยอะ |
| Multi-role logic | +2 | permission matrix |
| Production migration | +1 | rollback plan |
| Offline support | +3 | queue, retry, idempotency |

### GATE: ถาม tech stack ก่อน (ถ้ายังไม่มี)

```
question: "ต้องการบอก tech stack เพื่อ estimate แม่นยำขึ้นไหมครับ?"
header: "Tech Stack"
options:
  - label: "บอก stack"
    description: "frontend/backend/language ให้ — ช่วยเพิ่ม modifiers"
  - label: "ข้าม"
    description: "Estimate แบบ generic ก็พอ"
```

### Process

1. ดู requirement → ระบุ base complexity
2. บวก modifiers ตามปัจจัย stack
3. เทียบกับ reference tasks (ถ้ามี)
4. ปัดขึ้นเป็น Fibonacci ใกล้สุด
5. อัพเดท `Estimate Point` + `Effort level` ใน Task properties

### แสดงผลทีละ Epic

```markdown
## Effort Estimation — Epic 1: [ชื่อ]

| # | Task | Base | Modifiers | Total | Level |
|---|------|------|-----------|-------|-------|
| 1.1 | [task] | 3 | +2 (RLS) | 5 | Medium |
| 1.2 | [task] | 5 | — | 5 | Medium |

Subtotal: X points
```

ใช้ AskUserQuestion หลังแต่ละ Epic:

```
question: "Estimation ของ Epic 1 โอเคไหม?"
header: "Est Review"
options:
  - label: "โอเค → Epic ถัดไป"
  - label: "ปรับบาง task"
```

---

## 3. Traceability Matrix

### Step 1: ดึงข้อมูล
- `notion-fetch` Roadmap → ดู feature list พร้อม Ref numbers
- `notion-search` Tasks ทั้งหมดจาก Backlog
- Map tasks กลับไปหา requirement ผ่าน Ref number ใน AC หรือ content

### Step 2: Matrix

```markdown
📊 Requirement Traceability Matrix

| Ref | Requirement | Task ID | Epic | Sprint | Status | Coverage |
|-----|------------|---------|------|--------|--------|----------|
| X.1.1 | [Feature A] | [PREFIX]-1 | Auth | S1 | Backlog | ✅ Full |
| X.1.2 | [Feature B] | [PREFIX]-2 | User | S1 | Backlog | ✅ Full |
| Y.1.1 | [Feature C] | — | — | — | — | ❌ Missing |
| Z.1.1 | [Feature D] | [PREFIX]-20 | Infra | S3 | Backlog | 🟡 Partial |

📊 Coverage: XX/YY (XX%)
✅ Covered: X | 🟡 Partial: X | ❌ Missing: X
```

### Step 3: Action

ถ้ามี ❌ Missing:

```
question: "พบ X requirements ที่ยังไม่มี task — สร้างเพิ่มไหม?"
header: "Missing Tasks"
options:
  - label: "สร้าง task เพิ่ม"
    description: "แตก task เพิ่มให้ cover requirements ที่ขาด"
  - label: "ข้ามไปก่อน"
    description: "ค่อยทำทีหลัง"
  - label: "Out of scope"
    description: "Requirements เหล่านี้ไม่อยู่ใน scope"
```

---

## 4. Impact Analysis (Change Request)

เมื่อมี CR / requirement เปลี่ยน

### Step 1: ทำความเข้าใจ CR
- อะไรเปลี่ยน
- จากอะไรเป็นอะไร
- ทำไมเปลี่ยน

### Step 2: วิเคราะห์ 4 มิติ

```
🔄 Impact Analysis: [CR Title]

📝 สรุป CR:
- เดิม: [...]
- ใหม่: [...]
- เหตุผล: [...]

1️⃣ Task Impact
| Task ID | Task Name | Epic | ผลกระทบ | Action |
|---------|-----------|------|---------|--------|
| [P]-X | ... | Auth | แก้ scope | แก้ AC + TD |
| [P]-Y | ... | User | สร้างใหม่ | สร้าง task ใหม่ |
| [P]-Z | ... | Infra | ยกเลิก | status → Done (cancelled) |

2️⃣ Sprint Impact
- Sprint N: +X tasks, +Y points → อาจ overflow
- Sprint M: dependency เปลี่ยน → ต้อง replan

3️⃣ Technical Impact
- Database: [migration ไหม]
- API: [endpoint ที่กระทบ]
- UI: [component ที่ต้องแก้]
- Security: [permission กระทบ]

4️⃣ Effort Impact
- Effort: +X SP
- เวลา: ประมาณ X วัน
- Risk: [ระบุ]

📊 Impact Level: 🔴 High / 🟡 Medium / 🟢 Low
```

### Step 3: เสนอ Action Plan

```
question: "ต้องการจัดการ CR นี้อย่างไรครับ?"
header: "CR Decision"
options:
  - label: "รับ CR เต็ม"
    description: "ปรับ sprint plan + เพิ่ม/แก้ tasks ทั้งหมด"
  - label: "รับบางส่วน"
    description: "ทำ core changes ใน sprint นี้ ส่วนอื่น defer"
  - label: "Defer ทั้งหมด"
    description: "ใส่ backlog แล้วจัด sprint ใหม่ทีหลัง"
  - label: "ต้องคุยทีมก่อน"
    description: "สรุป impact เป็น brief ส่งทีม"
```

### Step 4: Execute
หลัง confirm → สร้าง/แก้ tasks, อัพเดท sprint, ตั้ง dependencies ตามที่ตกลง
