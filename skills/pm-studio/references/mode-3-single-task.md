# Mode 3: Single Task Deep Dive

วิเคราะห์ task เดียวให้ละเอียดสุด แล้วเขียนใหม่ในรูปแบบ **Context / Objective / Technical Details + Acceptance Criteria + Gherkin Scenario**

**เป้าหมาย:** ทำ task ให้ developer หยิบไปทำได้เลย ไม่ต้องถามเพิ่ม

**เมื่อไหร่ใช้:**
- ผู้ใช้มี task เดียวแล้ว (Notion URL / chat / file) บอกว่า "วิเคราะห์ task นี้" / "เขียน task ใหม่" / "rewrite"
- ผู้ใช้เลือก Mode 3 ที่ GATE-0
- ผู้ใช้เลือก "เขียน Task เดี่ยวใหม่" ที่ GATE-2 (หลัง Mode 1)
- input เป็น task เดียว ไม่ใช่ feature spec ขนาดใหญ่

---

## Workflow Overview

```
Read task + reference → GATE-1 focus → Analyze → GATE-2 fix gaps → Rewrite → GATE-ST-FINAL → (option) Update Notion
```

---

## Section 1: Read Task + Reference

### Step 1: อ่าน task หลัก
- ถ้าเป็น Notion URL → `notion-fetch`
- ถ้าเป็น chat text / file → อ่านตรงๆ

### Step 2: อ่าน reference tasks (ถ้ามี)
ถ้าผู้ใช้ระบุ "อ้างอิง THUN-69, THUN-68" → fetch มาอ่านด้วย

### Step 3: วิเคราะห์ pattern
ถ้ามี reference tasks ดู pattern ที่ใช้ร่วมกัน:
- Excel Import flow pattern
- History Log pattern
- BullMQ background process pattern
- Notification pattern

ใช้ pattern เดียวกันใน task ที่จะเขียนใหม่ — เพื่อให้ codebase consistent

---

## Section 2: Analyze (เหมือน Mode 1 แต่โฟกัส task เดียว)

ทำ **GATE-1** ถามโฟกัส (Functional/Technical/UX) เหมือน Mode 1

วิเคราะห์ task ตาม focus → output เป็น **Analysis Report** เหมือน Mode 1

### Hidden Requirement Detection (Task-level)

สำหรับ task เดียว ตรวจตามนี้:

**Input/Output**
- Input format ชัดไหม? (request body, query params)
- Output format ชัดไหม? (response shape, error codes)
- Validation rules ครบไหม?

**Edge cases**
- Empty / null / undefined values
- Concurrent users / race conditions
- Permission edge cases (role A vs role B)
- Timezone / locale handling

**Error handling**
- 4xx errors แต่ละแบบจัดการยังไง?
- 5xx errors มี retry/fallback ไหม?
- User-facing error messages เขียนยังไง?

**Backward compatibility**
- migration ของ existing data
- old API contracts ที่ยังต้อง support
- feature flag / phased rollout

---

## Section 3: GATE-2 → แก้ gaps

ใช้ GATE-2 เหมือน Mode 1 — ให้ผู้ใช้ตัดสินใจว่าจะแก้ gaps หรือไปเขียน task ใหม่เลย

```
question: "พบ X gaps — ต้องการทำอะไรต่อครับ?"
header: "Next Step"
options:
  - label: "เขียน task ใหม่เลย"
    description: "รวม gaps ที่พบเข้าไปใน task ใหม่"
  - label: "ถาม stakeholder ก่อน"
    description: "สรุปคำถามให้ส่งทีม"
  - label: "เลือกเฉพาะบางข้อ"
    description: "เลือก gaps ที่จะแก้"
```

---

## Section 4: Rewrite — เขียน Task ใหม่

ใช้ **Mode 3 Task Content Template** จาก `templates.md` (ละเอียด ครบทุก section)

### หลักการเขียน

1. **กระชับ** — ไม่อธิบายสิ่งที่ developer รู้อยู่แล้ว
2. **Context → Objective → Technical** — เรียง why → what → how
3. **AC ครอบคลุม** — happy path + edge cases + error handling + migration/compatibility
4. **Gherkin Scenario** — Given/When/Then สำหรับ use case สำคัญ 3-5 scenarios ไม่ต้องทำทุก AC เป็น scenario
5. **Out of Scope** — ระบุชัดเจนป้องกัน scope creep
6. **ไม่ใส่ code/schema** — เว้นแต่ผู้ใช้ขอ — เพราะ spec level ไม่ควรผูกกับ implementation detail
7. **ภาษาไทยเป็นหลัก** — ยกเว้น technical terms, Gherkin keywords (Given/When/Then), ชื่อ field/table/API

### Title format

```
[Platform/Screen] : Task Name
```

เช่น `[CS/BO] : ระบบจัดเรียง Attribute(Dimension) ส่วนของ Filter`

---

## Section 5: GATE-ST-FINAL — Confirm rewritten task

แสดง task ที่เขียนใหม่ใน chat ก่อน แล้วถาม:

```
question: "Task ที่เขียนใหม่ โอเคไหมครับ?"
header: "Task Review"
options:
  - label: "โอเค อัปเดต Notion"
    description: "เขียนทับ content เดิมบน Notion เลย"
  - label: "ปรับก่อน"
    description: "อยากแก้ไขบางส่วนก่อนอัปเดต"
  - label: "แสดงใน chat พอ"
    description: "ไม่ต้องอัปเดต Notion เก็บไว้ใน chat ก็พอ"
```

**ถ้าเลือก "ปรับก่อน"** → รอ input → แก้ → ถามอีกรอบ (loop)

**ถ้าเลือก "อัปเดต Notion"** → ใช้ `notion-update-page` เขียนทับ content ของ page

---

## Section 6: Update Notion

### วิธี update content

ใช้ `notion-update-page` กับ option ที่เหมาะสม:

- **replace_content_range** — แก้ section ที่ระบุได้ (แนะนำเมื่อรู้ block IDs)
- **insert_content_after** — เพิ่ม content หลัง block ที่ระบุ
- **replace_content** — เขียนทับทั้ง page (ใช้เมื่อ rewrite ทั้งหมด)

### Update properties

ใช้ `notion-update-page` กับ `update_properties`:
- Summary: สรุปสั้นๆ ของ task เป็น bullet points (ใช้ `<br>` คั่นบรรทัด)
- Estimate Point: อัพเดท SP ถ้าเปลี่ยน
- Effort level: Small / Medium / Large

### Verify

หลัง update:
- Fetch page อีกครั้ง → spot-check ว่า content ถูกต้อง
- แสดงลิงก์ให้ผู้ใช้กดไปดู:

```
✅ Updated: [Task Title]
[ลิงก์ไป Notion page]
```
