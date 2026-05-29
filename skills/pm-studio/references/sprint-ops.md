# Sprint Operations

จัดการ Sprint บน Notion ครบวงจร — สร้าง, เปลี่ยน status, สรุป

---

## 1. สร้าง Sprint ใหม่

### Step 0: ตรวจ config
ต้องมี:
- `notion.sprints_datasource`
- `notion.backlog_datasource`
- `notion.roadmap_url`

ถ้าไม่ครบให้ถามผู้ใช้ก่อน

### Step 1: อ่าน Roadmap
- `notion-fetch` หน้า Roadmap
- หา Sprint ถัดไปที่ยังไม่ได้สร้าง
- ดึง feature list, dependencies, sprint goal, definition of done

### Step 2: สร้าง Sprint page

ใช้ `notion-create-pages` ไปยัง sprints data source:
- **Sprint name:** `Sprint N — Theme Name` (ตาม Roadmap)
- **Dates:** start → end (2 สัปดาห์ default)
- **Sprint status:** "Future" (หรือ "Next" ถ้าเป็น Sprint ถัดจาก Current)

### Step 3: สร้าง Tasks ทั้งหมด

สำหรับแต่ละ feature ใน Sprint:
- สร้าง Task ใน backlog data source
- **Task name** = ชื่อ feature จาก Roadmap
- **Status** = "Backlog"
- **Task type** = "💬 Feature" (default), "🐞 Bug", หรือ "💅 Chore"
- **Priority** = ตาม Roadmap (P0/Critical → High, Important → Medium, Nice-to-have → Low)
- **Sprint** = URL ของ Sprint page ที่เพิ่งสร้าง
- **Epic** = ตาม feature category (อ่าน `epic-ops.md` Section 1 หลักการจัดหมวด)
- **Content** = ตาม Mode 2 Task Content Template (`templates.md`)

### Step 4: ตั้ง Dependencies
- ตรวจ Blocked by / Blocking ตาม Roadmap
- อัพเดท tasks ที่เกี่ยวข้อง

### Step 5: เขียน Sprint Planning notes
เพิ่ม Sprint Goal + Definition of Done ลงใน Sprint page content

### Step 6: Confirm กับผู้ใช้
ก่อนสร้างให้สรุปแผน:

```
📋 จะสร้าง: Sprint N — [Theme]
   - X tasks (Y points รวม)
   - Dates: [start] → [end]
   - Sprint Goal: [...]

Confirm สร้างเลยไหม?
```

---

## 2. เปลี่ยน Status

### Status Flow (บังคับ — ห้ามข้าม)

```
Backlog → In design → Ready For Dev → In progress → In review
       → Waiting for test → Testing → Ready for demo → Done
                                    ↘ Reject → Fixing ↗
```

### กฎ
1. **ตรวจ valid transition ก่อนเสมอ** — ถ้า user ขอ Backlog → Done ให้แนะนำ path ที่ถูก
2. **In progress** → ตั้ง `Started at` = วันนี้ (ถ้ามี field)
3. **Done** → ตั้ง `Completed at` = วันนี้
4. **Reject** → ต้องระบุเหตุผลใน task content (เพิ่ม block ใหม่)
5. **Fixing** → reset assignee เป็นคนเดิม

### Assign งาน

- หา user ID ด้วย `notion-get-users` หรือ `notion-search` query_type: "user"
- ตั้ง `Assignee` ด้วย user ID

---

## 3. Sprint Planning

เมื่อผู้ใช้ขอวางแผน Sprint:

### Step 1: ดู Backlog
- ดูที่ยังไม่มี Sprint (filter Sprint = empty)
- ดู Priority และ Effort level

### Step 2: ดู Roadmap
- หา tasks ที่ควรอยู่ใน Sprint นี้
- ตรวจ dependencies

### Step 3: แนะนำ tasks
แสดงเป็นตาราง พร้อม priority, effort, dependencies:

```markdown
📋 แนะนำ Tasks สำหรับ Sprint N

| Task | Priority | SP | Epic | Dependencies |
|------|----------|-----|------|--------------|
| ... | High | 8 | Auth | - |
| ... | Medium | 5 | User Mgmt | #1 |

รวม: X tasks, Y points
แนะนำ capacity: ~30-40 points/sprint
```

### Step 4: Confirm
ใช้ AskUserQuestion ให้ผู้ใช้เลือก/ปรับ → แล้ว assign tasks เข้า Sprint

---

## 4. Sprint Summary

เมื่อผู้ใช้ขอสรุป Sprint:

### Step 1: ดึงข้อมูล
- `notion-fetch` Sprint page เพื่อดู Tasks ที่เชื่อม
- `notion-fetch` แต่ละ Task เพื่อดู Status, Assignee, Priority

### Step 2: สรุปเป็นรายงาน

```markdown
📊 Sprint Summary: [Sprint Name]
📅 ระยะเวลา: [start] → [end]

✅ Done: X tasks
🔄 In Progress: X tasks
⏳ Waiting: X tasks (In review / Waiting for test / Testing)
📋 Backlog: X tasks
🚫 Blocked: X tasks
❌ Rejected: X tasks

📈 Progress: X/Y tasks (XX%)
📊 Points: X/Y SP (XX%)

🔥 High Priority Pending:
- [task name] — [status] — [assignee]

⛔ Blockers:
- [task name] blocked by [task name]

📝 Notes:
- [ข้อสังเกตหรือคำแนะนำ]
```

### Step 3: Action Items
- Tasks ที่ควรเร่ง (High priority + ยังไม่เริ่ม)
- Blockers ที่ต้องแก้
- Tasks ที่ overdue

ใช้ AskUserQuestion ถาม:

```
question: "ต้องการ action อะไรต่อครับ?"
header: "Sprint Action"
options:
  - label: "Export เป็นรายงาน"
    description: "สร้างเอกสาร .md / .docx เพื่อส่ง stakeholder"
  - label: "วิเคราะห์ blockers"
    description: "วิเคราะห์ blockers หาทางแก้"
  - label: "วางแผน Sprint ถัดไป"
    description: "ไป Sprint Planning"
  - label: "ดูแค่นี้พอ"
    description: "ไม่ทำต่อ"
```
