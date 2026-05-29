# Mode 4: Presale Analysis

วิเคราะห์ requirement แบบ **คร่าวๆ** สำหรับขั้นตอน presale / quotation — ก่อนทำสัญญา / commit project

**เป้าหมาย:** ตอบลูกค้าเร็วๆ ว่าระบบนี้ทำอะไรได้บ้าง ใช้เวลาคร่าวๆ ประมาณเท่าไหร่ — เพื่อให้ตัดสินใจไปต่อหรือไม่

**ความแตกต่างจาก Mode 1:**

| มิติ | Mode 1: Requirement Analysis | Mode 4: Presale Analysis |
|------|------------------------------|--------------------------|
| **Audience** | Internal team / dev | ลูกค้า / sales / decision maker |
| **Depth** | ลึก ครบ hidden requirements | คร่าวๆ ภาพรวม |
| **Output** | Analysis Report พร้อม gaps, edge cases | Quotation-ready Summary |
| **Technical** | มี (architecture, integration) | ❌ ไม่มี |
| **Time** | 30-60 นาที | 5-15 นาที |
| **Effort estimate** | Story Points | T-shirt size / weeks |

---

## เมื่อไหร่ใช้ Mode 4

- ลูกค้าถาม "ทำได้ไหม กี่เดือน"
- Sales ต้องการ rough quote
- Internal pre-commit decision (รับ project นี้ไหม)
- "อยากได้ rough estimate"
- "ขอ scope คร่าวๆ ส่งลูกค้า"

**ห้ามใช้ Mode 4 เมื่อ:**
- ต้องการ requirement ครบ พร้อมแตก WBS → ใช้ Mode 1 + Mode 2
- ต้องการ technical detail → ใช้ Mode 1 หรือ Mode 3
- ลูกค้า confirm จะทำแล้ว ต้องการ contract spec → ใช้ Mode 1 (deep)

---

## Workflow Overview

```
Input → Quick Discovery (1-2 คำถาม) → Feature List → T-shirt Estimate → Risks → Presale Summary → ส่งลูกค้า
```

**สั้น เร็ว ตรงประเด็น** — ไม่ต้อง interactive หลาย gates เหมือน Mode 1

---

## Section 1: Quick Discovery (สั้นๆ 1-2 คำถาม)

ถ้า input ระบุชัดอยู่แล้ว → ข้ามไป Section 2

ถ้ายังไม่ชัด ใช้ `AskUserQuestion` **ครั้งเดียว** (ห้ามถามยาว — presale ต้องเร็ว):

```
question: "ขอ context นิดหน่อย — ระบบ [ชื่อ] นี้:"
header: "Quick Context"
multiSelect: true
options:
  - label: "B2C — ลูกค้าใช้เอง"
    description: "End user / consumer"
  - label: "B2B — องค์กรใช้"
    description: "Business / company internal"
  - label: "มี Admin Panel"
    description: "ต้องมีระบบหลังบ้าน"
  - label: "เชื่อมระบบเดิม"
    description: "Integration กับระบบที่ลูกค้ามีอยู่"
```

**ถ้า input ครบแล้ว** ข้ามไปทำ Section 2 ทันที — อย่า over-ask

---

## Section 2: High-Level Feature List

แตกเป็น **กลุ่ม feature คร่าวๆ** (5-10 bullet points) — ไม่ลงรายละเอียด

```markdown
🎯 Scope (ภาพรวม)

### Front-end (ผู้ใช้)
- [Feature group 1] — [คำอธิบาย 1 ประโยค]
- [Feature group 2] — [คำอธิบาย 1 ประโยค]
- [Feature group 3] — [คำอธิบาย 1 ประโยค]

### Back-office (ถ้ามี)
- [Feature group 1] — [คำอธิบาย 1 ประโยค]
- [Feature group 2] — [คำอธิบาย 1 ประโยค]

### Integration & Other
- [item 1]
- [item 2]
```

**กฎ:**
- 1 bullet = 1 feature group (ไม่ใช่ task ย่อย)
- คำอธิบาย 1 ประโยค ห้ามยาว
- ไม่ใส่ technical detail (database, API, architecture)
- ไม่ใส่ hidden requirements deep dive

---

## Section 2.5: Rough WBS Breakdown (Optional)

ถ้าลูกค้าอยากเห็น **"จะได้อะไรบ้าง"** ละเอียดขึ้น (แต่ยังไม่ใช่ Mode 2 level) แตกเป็น sub-feature

**ใช้เมื่อ:**
- ลูกค้าขอ "scope ละเอียดกว่านี้"
- ลูกค้าจะเอาไป compare กับ vendor อื่น
- ต้อง justify timeline (ทำไม 3 เดือน?)

**ห้ามใช้เมื่อ:**
- ลูกค้าแค่ถามคร่าวๆ (อย่ายัดให้)
- ยังไม่ confirm จะทำ (เก็บไว้ Discovery)

### Rough WBS Template

```markdown
🎯 Scope Breakdown (Rough)

### 📱 Customer Side
1. **Authentication** — LINE LIFF login + profile sync
2. **Point Wallet** — ดูยอด point + ประวัติการสะสม/แลก
3. **Coupon Catalog** — ดู coupon ที่แลกได้ + filter
4. **Coupon Redemption** — แลก point เป็น coupon + show QR
5. **Notification** — ดู LINE OA messages + push setting

### 🏪 Restaurant Admin
1. **Dashboard Login** — auth + role-based access (manager/staff)
2. **Point Rule Config** — กำหนดอัตรา point ต่อยอดสั่ง
3. **Coupon Builder** — สร้าง/แก้ไข coupon + เงื่อนไข
4. **Campaign Manager** — schedule promotion + target group
5. **Reports** — usage, redemption rate, top customers, ROI

### 🔌 Integration
1. **POS Sync** — webhook/polling integrate กับ POS ปัจจุบัน
2. **LINE OA** — Messaging API + LIFF setup
3. **Authentication Flow** — LINE Login + customer linking
```

**กฎสำหรับ Rough WBS:**
- 3-7 sub-items ต่อ feature group (ไม่เกิน 7)
- 1 sub-item = 1 ประโยคสั้น
- **ไม่ใส่ story points** — ใช้แค่ T-shirt size รวมที่ Section 3
- **ไม่ใส่ priority** — presale ยังไม่ถึงขั้น prioritize
- **ไม่ใส่ Acceptance Criteria** — เก็บไว้ Mode 2/3 ตอน Discovery จริง
- **ไม่ใส่ dependency** — too detailed for presale

### เทียบกับ Mode 2

| | Mode 2 WBS | Mode 4 Rough WBS |
|---|-----------|------------------|
| Depth | Epic → Task (5-13 SP) | Group → Sub-feature (no SP) |
| Per task content | Context + Objective | แค่ชื่อ + 1 ประโยค |
| Story Points | ✅ มี | ❌ ไม่มี |
| Priority | ✅ High/Med/Low | ❌ ไม่มี |
| Dependency | ✅ มี | ❌ ไม่มี |
| AC | ❌ (เก็บไว้ Mode 3) | ❌ ไม่มี |
| Audience | Internal dev | Customer / sales |
| Purpose | Sprint planning | Show "what's included" |

---

## Section 2.5 Output (เพิ่มใน Presale Summary)

ถ้าทำ Rough WBS แล้ว ใส่เป็น section ใหม่ใน Final Summary:

```markdown
## 🎯 Scope Breakdown (Rough)
[ตามที่แตกไว้ Section 2.5]

## ⏱️ Effort Estimate
[ตามเดิม]
```

---

## Section 3: T-shirt Size Estimate

ใช้ T-shirt size แทน story points — เพราะ presale ไม่ต้องการความแม่นยำสูง

| Size | Timeline | Team | ตัวอย่าง |
|------|----------|------|---------|
| **S** | 2-4 weeks | 1-2 devs | Landing page + 1-2 simple forms |
| **M** | 1-2 months | 2-3 devs | CRUD app + auth + basic reports |
| **L** | 2-4 months | 3-5 devs | Multi-role system + workflow + integrations |
| **XL** | 4-8 months | 5+ devs | Enterprise system + multi-tenant + complex business logic |
| **XXL** | 8+ months | full team | Platform + ecosystem + heavy integrations |

### Presale Estimate Output

```markdown
⏱️ Effort Estimate
- **Size:** M (Medium)
- **Timeline:** ~6-8 weeks
- **Team:** 2-3 devs (full-stack) + 1 PM + 1 designer (part-time)
- **Confidence:** Medium — ขึ้นกับ integration ที่ยังไม่ชัด
```

**Confidence levels:**
- 🟢 **High** — เคยทำคล้ายๆ มาแล้ว, scope ชัด
- 🟡 **Medium** — มี unknown แต่จัดการได้
- 🔴 **Low** — มี risk สูง ต้อง discovery เพิ่ม

---

## Section 4: Risks & Assumptions

```markdown
⚠️ Key Risks
1. [Risk 1] — เช่น "integration กับ POS ที่ลูกค้าใช้ ยังไม่รู้ API"
2. [Risk 2] — เช่น "scope อาจขยายเมื่อเริ่ม discovery จริง"

✅ Assumptions (สิ่งที่สมมติว่าจริง)
1. ลูกค้ามี domain + hosting พร้อม
2. มีคนฝั่งลูกค้า dedicate ตอบคำถาม dev
3. Design system / branding ใช้แบบ standard (ไม่ต้อง custom)
```

---

## Section 5: Presale Summary (Final Output)

```markdown
# 📋 Presale Summary: [Project Name]

**Prepared for:** [Customer name]
**Date:** [วันที่]
**Prepared by:** [PM name / Nixxel]

---

## 🎯 Scope (ภาพรวม)

### Front-end (ผู้ใช้งาน)
- [Feature group 1]
- [Feature group 2]
- [Feature group 3]

### Back-office (Admin)
- [Feature group 1]
- [Feature group 2]

### Integration
- [integration 1]
- [integration 2]

---

## ⏱️ Effort Estimate
- **Size:** [T-shirt size]
- **Timeline:** ~X weeks
- **Team:** X devs + 1 PM
- **Confidence:** [High/Medium/Low]

---

## ✅ What's Included
- [feature group 1]
- [feature group 2]
- [feature group 3]
- Project management + weekly sync
- Bug fixes ระหว่าง development
- Basic documentation

---

## ❌ What's NOT Included (Out of Scope)
- [item 1] — เช่น "Mobile app (web responsive only)"
- [item 2] — เช่น "Custom branding (ใช้ template)"
- [item 3] — เช่น "Data migration จากระบบเดิม"
- Maintenance หลัง go-live (แยกเป็น contract ต่างหาก)

---

## ⚠️ Risks & Assumptions

**Risks:**
- [risk 1]
- [risk 2]

**Assumptions:**
- [assumption 1]
- [assumption 2]

---

## 📝 Next Steps
1. ลูกค้า review summary นี้
2. ถ้าตกลง → ทำ Discovery workshop (1 week) เพื่อทำ requirement ละเอียด
3. หลัง discovery จะได้ formal SOW + final timeline + price
```

---

## Section 6: ส่งให้ลูกค้า

หลังเขียน summary เสร็จ ใช้ `AskUserQuestion`:

```
question: "Presale summary พร้อมแล้ว — ต้องการทำอะไรต่อ?"
header: "Next Action"
options:
  - label: "Export เป็น .docx"
    description: "สร้างไฟล์ Word ส่งลูกค้าได้เลย"
  - label: "Export เป็น .pdf"
    description: "สร้าง PDF ส่งลูกค้า"
  - label: "ส่ง Notion page"
    description: "สร้าง Notion page (shareable link)"
  - label: "Copy text ไปอีเมลเอง"
    description: "แค่ copy ข้อความไป paste"
```

ถ้าเลือก Export → ใช้ docx skill หรือ pdf skill (จาก nixxel-doc-generator) สร้างไฟล์

---

## หลักการของ Mode 4

1. **เร็ว** — 5-15 นาที จบ ห้ามถามนาน
2. **คร่าวๆ** — ไม่ลงรายละเอียด เก็บไว้ Mode 1 ทำ
3. **ลูกค้าเข้าใจง่าย** — ไม่ใช้ jargon ทาง dev (story points, microservice, etc.)
4. **ระบุ what's in / what's out ชัดเจน** — ป้องกัน scope creep
5. **มี confidence level** — บอกตรงๆ ว่ามั่นใจระดับไหน อย่า over-commit
6. **เป็น stepping stone** — บอกชัดว่าต้องทำ Discovery ต่อก่อน contract จริง

---

## Anti-pattern (อย่าทำ)

❌ **อย่าทำ Hidden Requirement Detection ลึก** — เก็บไว้ Mode 1 ตอน discovery จริง

❌ **อย่าใส่ Technical Architecture** — ลูกค้าไม่สนใจ tech stack ตอนนี้

❌ **อย่าทำ Effort Estimation เป็น Story Points** — ใช้ T-shirt size

❌ **อย่าถามนาน** — presale ต้องเร็ว ถ้าต้องถามเยอะ = ควรเป็น Discovery workshop ไม่ใช่ presale

❌ **อย่า commit timeline แน่นอน** — ใช้ "~X weeks" + confidence level เสมอ
