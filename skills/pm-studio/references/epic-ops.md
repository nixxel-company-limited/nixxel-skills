# Epic Operations

จัดการ Epic ครบวงจร — สร้าง/แก้, bulk assign, dashboard, health analysis

**Epic คืออะไร:** Product capability — หน่วยส่งมอบ value ระดับสูงสุดที่รวมกลุ่ม tasks ที่เกี่ยวข้องกันเชิง domain

---

## 1. สร้าง/แก้ไข Epic Options

### เมื่อไหร่ควรจัดการ Epic

- **โปรเจคใหม่** — วิเคราะห์ scope ทั้งหมดแล้วเสนอ Epic structure
- **Scope เปลี่ยน** — มี feature ใหม่ที่ไม่เข้า Epic ไหน → สร้าง Epic ใหม่หรือ restructure
- **Epic ใหญ่เกิน** — Epic มี tasks >30 → แตกเป็น sub-Epics
- **Epic ไม่ใช้แล้ว** — tasks ใน Epic หมดหรือถูก cancel → ลบ option

### หลักการออกแบบ (MECE)

1. **Mutually Exclusive** — ทุก task อยู่ใน Epic เดียวเท่านั้น
2. **Collectively Exhaustive** — ทุก task ต้องมี Epic ไม่มี orphan
3. **Domain-driven** — แบ่งตาม business domain ไม่ใช่ technical layer
4. **Balanced size** — 5-30 tasks ต่อ Epic
5. **Meaningful naming** — ชื่อสื่อ domain ชัดเจน

### Step 1: วิเคราะห์ scope
- `notion-fetch` database schema → ดู Epic options ปัจจุบัน
- `notion-search` tasks ทั้งหมด → ดู feature landscape
- วิเคราะห์ว่า tasks จัดกลุ่มตาม domain ได้อย่างไร

### Step 2: ออกแบบ Epic structure

```markdown
🏗️ Epic Structure Proposal

| Epic | Color | Scope Definition | ตัวอย่าง Tasks |
|------|-------|-----------------|----------------|
| Auth & Security | red | ระบบ authentication, authorization | Login, 2FA, RBAC |
| User Management | blue | จัดการ user profiles, roles | CRUD Users, CSV Import |
| Leave Management | green | การลา ยอดวันลา การอนุมัติ | Leave Request, Leave Balance |
| Infrastructure | gray | DevOps, utilities, cross-cutting | Backup, i18n, Search |
```

**Notion colors:** red, blue, green, yellow, orange, pink, purple, gray, brown, default

### Step 3: Update Database Schema

ใช้ `notion-update-data-source` กับ `ALTER COLUMN` DDL:

```sql
ALTER COLUMN "Epic" SET SELECT(
  'Auth & Security':red,
  'User Management':blue,
  'Leave Management':green,
  'OT Management':yellow,
  'Approval & Workflow':orange,
  'Notification':pink,
  'Reports & Analytics':purple,
  'Infrastructure':gray
)
```

**สำคัญ:** ก่อนลบ options เก่า ตรวจ tasks ที่ใช้ option นั้น — ต้อง migrate ก่อน

### Step 4: AskUserQuestion confirm

```
question: "Epic structure นี้โอเคไหมครับ?"
header: "Epic Structure"
options:
  - label: "โอเค ใช้เลย"
    description: "Execute ALTER COLUMN ใน Notion"
  - label: "ปรับก่อน"
    description: "อยากแก้ไข Epic บางตัว"
  - label: "อธิบายเพิ่ม"
    description: "อยากรู้เหตุผลของแต่ละ Epic"
```

---

## 2. Bulk Assign Epic ให้ Tasks

เมื่อต้องการ assign Epic ให้ tasks จำนวนมาก

### Step 1: ค้นหา tasks ทั้งหมด

- `notion-search` กับ `data_source_url` → ดึง tasks
- อาจต้อง search หลายรอบด้วย keyword ต่างๆ
- ตรวจว่ามี orphan tasks (ไม่มี Epic) ไหม

### Step 2: จัดหมวด tasks → Epic

ใช้ semantic analysis:
- อ่านชื่อ task + description + content
- Match กับ scope definition ของแต่ละ Epic
- ถ้าก้ำกึ่ง 2 Epics → ดู primary purpose

### หลักการตัดสิน
- **Primary domain wins** — task เกี่ยวหลาย domain → ดู primary purpose
- **User-facing vs system** — task ที่ user เห็นตรง → ดู user perspective
- **Cross-cutting → Infrastructure** — task ที่ใช้ร่วมทั้ง system → Infrastructure
- **อธิบายได้** — ทุกการจัดต้องมีเหตุผล

### Step 3: เสนอ mapping ให้ผู้ใช้ review

```markdown
📋 Epic Assignment Plan

🔴 Auth & Security (22 tasks):
  - Login/Logout, 2FA, Password *, Account Lockout, ...

🔵 User Management (12 tasks):
  - CRUD Users, User Profile, CSV Import, ...

🟢 Leave Management (16 tasks):
  - Leave Request, Leave Balance, Calendar View, ...

⬜ Infrastructure (24 tasks):
  - RBAC Schema, Backup, i18n, Clock In/Out, ...

ทั้งหมด: XX tasks
Orphan (ไม่มี Epic): 0 tasks
```

```
question: "Epic assignment นี้โอเคไหมครับ?"
header: "Bulk Assign"
options:
  - label: "ถูกต้อง execute เลย"
    description: "Update ทั้งหมด XX tasks"
  - label: "ปรับบาง task"
    description: "บาง task ควรอยู่ Epic อื่น"
  - label: "ขอ review ก่อน"
    description: "ดูทีละ Epic ละเอียดก่อน"
```

### Step 4: Execute bulk update
- Update parallel (10 tasks/batch)
- ใช้ `notion-update-page` กับ `update_properties` → `{"Epic": "Epic Name"}`
- Report progress ทุก batch

### Step 5: Verify
- Spot-check tasks สุ่มจากแต่ละ Epic
- ตรวจไม่มี orphan เหลือ

---

## 3. Epic Progress Dashboard

เมื่อผู้ใช้ขอดู progress ของ Epic / ภาพรวมโปรเจค

### Step 1: ดึงข้อมูล
- `notion-search` tasks ทั้งหมดจาก backlog data source
- จัดกลุ่มตาม Epic
- นับ status ของแต่ละ task

### Step 2: Dashboard

```
📊 Epic Progress Dashboard
📅 As of: [วันที่]

| Epic                  | Total | Done | Active | Backlog | Progress |
|-----------------------|-------|------|--------|---------|----------|
| 🔴 Auth & Security    |   22  |  10  |    5   |    7    | ██████░░ 45% |
| 🔵 User Management    |   12  |   4  |    3   |    5    | ████░░░░ 33% |
| 🟢 Leave Management   |   16  |   0  |    2   |   14    | █░░░░░░░ 13% |
| 🟡 OT Management      |    4  |   0  |    0   |    4    | ░░░░░░░░  0% |
| 🟠 Approval & Workflow |   12  |   0  |    1   |   11    | █░░░░░░░  8% |
| 🩷 Notification        |    2  |   0  |    0   |    2    | ░░░░░░░░  0% |
| 🟣 Reports & Analytics |   12  |   0  |    0   |   12    | ░░░░░░░░  0% |
| ⬜ Infrastructure      |   24  |   8  |    4   |   12    | ████░░░░ 33% |
| 📈 TOTAL              |  104  |  22  |   15   |   67    | ███░░░░░ 21% |

🏆 Top Performing: Auth & Security (45%)
🐌 Needs Attention: OT/Notification/Reports (0%)
🔥 Most Active: Auth & Security (5 in progress)
```

### Step 3: Insights

```
💡 PM Insights:

1. ⚖️ Balance Issue: Auth ไปไกลแต่ OT/Notification/Reports ยังไม่เริ่ม
   → แนะนำ: Sprint ถัดไป focus OT + Notification

2. 🎯 Sprint Velocity: เฉลี่ย X tasks/sprint → เสร็จทั้งหมดภายใน Sprint N

3. 🚧 Risk: Leave Mgmt มี 16 tasks เพิ่งเริ่ม 2 → อาจ delay
   → แนะนำ: priority Leave Mgmt ใน Sprint 3-5

4. 📊 Effort: Infrastructure ใหญ่ที่สุด → อาจต้อง rebalance
```

---

## 4. Epic Health Analysis

วิเคราะห์เชิงลึก: Epic structure ดีอยู่ไหม ต้องปรับไหม

### Section 1: Structural Health

```
🏥 Epic Health Report

📐 Size Balance:
| Epic | Tasks | SP | Health |
|------|-------|-----|--------|
| Auth & Security | 22 | 85 | ✅ Good (5-30) |
| Infrastructure | 24 | 110 | ⚠️ Large — พิจารณาแตก |
| OT Management | 4 | 15 | ⚠️ Small — อาจรวม |
| Notification | 2 | 8 | 🔴 Too small — ควรรวม |

📊 Balance Score: 7/10
- Largest: Infrastructure (24)
- Smallest: Notification (2)
- Ratio: 12:1 → ⚠️ ไม่ balance (ควร < 5:1)
```

### Section 2: Coverage Check

```
🔍 Coverage Analysis:

✅ All tasks have Epic: 103/104 (99%)
⚠️ Orphan tasks: 1 (empty template)
✅ No duplicate assignments
✅ All Epics have tasks: 8/8

🎯 MECE Score: 9.5/10
```

### Section 3: Recommendations

```
💡 Restructuring Recommendations:

1. 🔄 Merge "Notification" (2) → "Approval & Workflow"
   เหตุผล: Notification ในโปรเจคนี้ส่วนใหญ่เกี่ยว approval
   Impact: Approval & Workflow → 14 tasks

2. 🔀 Split "Infrastructure" (24) → "Infrastructure" + "Time & Attendance"
   เหตุผล: Clock In/Out + Work Hours แยก domain
   Impact: Infrastructure → 14, Time & Attendance → 10

3. ✅ Keep ที่เหลือเหมือนเดิม
```

ใช้ AskUserQuestion ขอ confirm ก่อน execute:

```
question: "Restructuring plan นี้โอเคไหมครับ?"
header: "Restructure"
options:
  - label: "ตกลง execute"
    description: "1. Update schema 2. Migrate tasks 3. Verify"
  - label: "ทำบางส่วน"
    description: "เลือก recommendation ที่จะทำ"
  - label: "ขอคิดดูก่อน"
    description: "ไม่ทำตอนนี้ เก็บ recommendation ไว้"
```
