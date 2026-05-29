# Mode 1: Requirement Analysis

วิเคราะห์ feature spec / PRD / requirement document แล้วชี้ gaps, hidden requirements, ambiguities, edge cases, conflicts

**เป้าหมาย:** ทำให้ requirement พร้อมแตกเป็น WBS หรือ implement ได้ — ไม่มีจุดคลุมเครือ ไม่มีระบบหลังบ้านที่ขาด

---

## Workflow Overview

```
Input → GATE-PRE (ถ้าจำเป็น) → GATE-1 focus → Analyze → GATE-1.5 hidden requirements → Report → GATE-2 next step
```

---

## Section 1: Context Discovery (GATE-PRE)

**เมื่อไหร่ trigger เสมอ:**

1. **กว้างเกินไป** — ผู้ใช้บอกแค่ชื่อ feature/ระบบ ("อยากได้ระบบ Promotion", "ทำระบบ HR")
2. **ไม่รู้ context** — ไม่รู้ว่าเป็นระบบอะไร, มีกี่ role, ใครเป็นผู้ใช้
3. **มาเป็น keyword/phase** — รายการสั้นๆ ("Phase 2: Reporting + Notification + Settings")
4. **Spec ไม่ครบ flow** — มีแต่ฝั่ง user แต่ไม่มี admin/operational flow

### Deep Discovery (กรณี 🔴 Vague)

**ชุดที่ 1: ใครใช้ — ใช้ AskUserQuestion**

```
question: "ระบบ [ชื่อ] นี้ ใครเป็นผู้ใช้หลักบ้างครับ?"
header: "User Roles"
multiSelect: true
options:
  - label: "End User / ลูกค้า"
    description: "ผู้ใช้งานทั่วไปที่เห็น/ใช้ feature นี้"
  - label: "Admin / Back-office"
    description: "คนที่ตั้งค่า จัดการ ดูแลระบบหลังบ้าน"
  - label: "Staff / พนักงาน"
    description: "พนักงานที่ใช้ระบบในการทำงานประจำวัน"
  - label: "Manager / หัวหน้า"
    description: "คนที่ดู report, approve, ตัดสินใจ"
```

**ชุดที่ 2: Scope & Boundary (open-ended ใน chat)**

ถามต่อ:
- "[Role A] จะทำอะไรได้บ้างในระบบนี้?"
- "[Role B - Admin] ต้องจัดการอะไรบ้าง?"
- "มีระบบอื่นที่ต้องเชื่อมต่อไหม? (payment, inventory, CRM)"

**ชุดที่ 3: ยืนยัน Scope Statement**

สรุปแล้ว AskUserQuestion ขอ confirm:

```
📋 Scope Statement: [ชื่อระบบ]

ผู้ใช้:
- [Role 1]: [สิ่งที่ทำได้]
- [Role 2]: [สิ่งที่ทำได้]

ขอบเขต: [module 1, 2, 3]
ไม่รวม (Out of scope): [สิ่งที่ไม่ทำ]
```

```
question: "Scope นี้ตรงกับที่ต้องการไหมครับ?"
header: "Scope Confirm"
options:
  - label: "ตรงแล้ว ไปต่อ"
    description: "Scope ถูกต้อง เริ่ม Analyze ได้เลย"
  - label: "ปรับ scope"
    description: "อยากเพิ่ม/ลดบางส่วน"
  - label: "ยังไม่ชัด"
    description: "อยากให้ถามเพิ่มอีก"
```

วน loop จนกว่า scope จะ confirm

---

## Section 2: GATE-1 — โฟกัสการวิเคราะห์

```
question: "ต้องการเน้นวิเคราะห์ด้านไหนครับ?"
header: "Analysis Focus"
multiSelect: true
options:
  - label: "Functional"
    description: "User flow, business rules, edge cases, error handling"
  - label: "Technical"
    description: "Performance, security, data validation, API contracts"
  - label: "UX/UI"
    description: "UI states, responsive, accessibility"
  - label: "ทุกด้าน (Recommended)"
    description: "วิเคราะห์ครบทุกมิติ"
```

---

## Section 3: Analyze — ตรวจตาม focus ที่เลือก

### Functional Requirements
- User flow ครบไหม (happy path + error path)
- Input/Output ชัดเจนไหม
- Business rules ระบุครบไหม
- Edge cases ระบุแล้วหรือยัง
- Error handling กำหนดแล้วหรือยัง

### Non-Functional / Technical
- Performance expectations
- Security requirements
- Data validation rules
- Concurrency / race conditions

### UX & UI
- UI states ครบไหม (loading, empty, error, success)
- Mobile/responsive considerations
- Accessibility (keyboard, screen reader)

### Integration & Dependencies
- API contracts ชัดเจนไหม
- Third-party dependencies ระบุแล้วหรือยัง
- Data migration needs
- Backward compatibility

### Acceptance & Testing
- Acceptance criteria ทดสอบได้จริงไหม
- Test scenarios ครอบคลุมไหม

---

## Section 4: GATE-1.5 — Hidden Requirement Detection (สำคัญมาก)

**ทำทุกครั้งหลัง Analyze** — ลูกค้ามักระบุเฉพาะ "ฝั่งหน้าบ้าน" (user-facing) แต่ลืม operational flow ทั้งหมด

### Hidden Requirement Checklist

สำหรับทุก feature ที่มี ตรวจตามนี้:

**🏗️ การสร้าง/ตั้งค่า (Creation & Configuration)**
- ใครเป็นคนสร้าง/ตั้งค่า feature นี้? (admin? system? auto?)
- UI สำหรับสร้าง/แก้ไข/ลบ อยู่ที่ไหน?
- เงื่อนไข/กฎ ตั้งค่าอย่างไร?
- มี draft/publish flow ไหม?
- ต้อง approve ก่อน publish ไหม?

**📊 การดูแล/ติดตาม (Monitoring & Reporting)**
- Admin ดู dashboard/report อะไรบ้าง?
- มี notification เมื่อเกิด event สำคัญไหม?
- มี audit log ไหม? (ใครทำอะไร เมื่อไหร่)
- Export data ได้ไหม?

**🔄 Lifecycle Management**
- feature นี้มี status อะไรบ้าง? (active/inactive/expired/archived)
- หมดอายุ/ปิดใช้งาน ทำอย่างไร? (manual? auto?)
- ถ้าปิดแล้ว ข้อมูลเก่าจะเป็นยังไง?
- มี versioning ไหม?

**⚙️ Hidden Business Rules**
- เงื่อนไขซ้อนกันได้ไหม? (เช่น promotion 2 ตัวใช้พร้อมกัน)
- มี limit อะไรบ้าง? (ต่อคน, ต่อวัน, ต่อ campaign)
- กรณีพิเศษ? (refund แล้ว promotion คืนไหม?)
- timezone / scheduling — ตั้งเวลาล่วงหน้าได้ไหม?

**🔗 Hidden Integrations**
- ส่ง notification (email/push/SMS) เมื่อไหร่?
- เชื่อมกับระบบอื่น? (payment, inventory, CRM, analytics)
- มี webhook / event ที่ต้อง fire ไหม?

### แสดงสิ่งที่พบ

แสดงเฉพาะสิ่งที่ **ขาดหายจาก requirement เดิม**:

```
🔎 Hidden Requirements ที่พบ
━━━━━━━━━━━━━━━━━━━━━━━━━━━

🏗️ Back-office ที่ขาด:
1. ❌ ไม่มี UI สำหรับ admin สร้าง/จัดการ [feature]
2. ❌ ไม่มีระบบ approval

📊 Monitoring ที่ขาด:
3. ❌ ไม่มี dashboard สำหรับ admin
4. ❌ ไม่มี audit log

⚙️ Business Rules ที่ไม่ได้ระบุ:
5. ❓ ไม่ได้ระบุว่า [rule X] ทำงานยังไง
6. ❓ ไม่ได้ระบุ refund policy
```

### GATE-1.5: ถาม confirm

```
question: "พบ X สิ่งที่ requirement ไม่ได้ระบุ — ต้องการจัดการอย่างไรครับ?"
header: "Hidden Requirements"
options:
  - label: "เพิ่มเข้า scope ทั้งหมด"
    description: "เอาทุกข้อเข้า scope แล้ว Claude ช่วยร่าง requirement เพิ่ม"
  - label: "เลือกเฉพาะบางข้อ"
    description: "ดูทีละข้อแล้วเลือกว่าจะเอาเข้า scope หรือไม่"
  - label: "ถาม stakeholder ก่อน"
    description: "สรุปเป็นคำถาม list ให้ส่งถามลูกค้า/ทีม"
  - label: "ข้ามไปก่อน"
    description: "เก็บไว้เป็น note แต่ไม่เอาเข้า scope ตอนนี้"
```

**ถ้าเลือก "เลือกเฉพาะบางข้อ"** — AskUserQuestion แบบ multiSelect ให้เลือกแต่ละข้อ

**ถ้าเลือก "ถาม stakeholder"** — generate Stakeholder Questions List ที่พร้อมส่งต่อได้เลย

---

## Section 5: Output — Requirement Analysis Report

```markdown
📝 Requirement Analysis Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📌 สรุปภาพรวม
- จำนวน features: X
- ความสมบูรณ์โดยรวม: XX%
- ระดับ readiness: พร้อม / ต้องแก้ไข / ต้องทำเพิ่ม

🔴 Gaps (สิ่งที่ขาดหาย)
1. [requirement ที่ขาด] — ผลกระทบ: [อธิบาย]

🟣 Hidden Requirements (ระบบหลังบ้าน/operational flow ที่ขาด)
1. [ระบบที่ขาด] — ทำไมต้องมี: [เหตุผล]

🟡 Ambiguities (สิ่งที่ไม่ชัดเจน)
1. [ข้อที่คลุมเครือ] — คำถามที่ควรถาม: [...]

🟠 Edge Cases (กรณีพิเศษที่ไม่ได้ระบุ)
1. [edge case] — แนะนำ: [วิธีจัดการ]

🔵 Conflicts (ข้อขัดแย้ง)
1. [requirement A] ขัดกับ [requirement B]

✅ Strengths (จุดแข็งของ spec)
1. [สิ่งที่ทำได้ดี]

💡 Recommendations
1. [คำแนะนำ]
```

---

## Section 6: GATE-2 — Next Step

```
question: "พบ X gaps, Y hidden requirements, Z risks — ต้องการทำอะไรต่อครับ?"
header: "Next Step"
options:
  - label: "แตก WBS ต่อ"
    description: "ไป Mode 2 — แตก Epic → Task (รวม hidden requirements ที่ confirm แล้ว)"
  - label: "เขียน Task เดี่ยวใหม่"
    description: "ไป Mode 3 — รวมทุกอย่างเขียนเป็น 1 task ละเอียด"
  - label: "แก้ gaps ก่อน"
    description: "อยากเสริม spec ให้ครบก่อนค่อยทำต่อ"
  - label: "ถาม stakeholder"
    description: "สรุปคำถามให้ส่งทีม"
```
