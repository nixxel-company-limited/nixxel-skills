# TeamLead Research Protocol (Research Gate)

This file is the detailed protocol for the Research Gate summarized in `SKILL.md`.

The Research Gate exists because design decisions made on shallow context are the most expensive mistakes in the whole flow: a wrong assumption at brainstorm time survives into the spec, the plan, and every implementation wave. Wave 0 catches some of this, but Wave 0 runs *after* the Human has already approved a design and a plan — too late to cheaply change direction. The Research Gate front-loads real knowledge so the Lead asks informed questions and proposes approaches grounded in the actual codebase, not in guesses.

---

## When the Gate Applies

| Situation | Research Gate? |
|-----------|:--------------:|
| Any brainstorm-required work (per Brainstorming Trigger Matrix: new feature, new component/functionality, behavior/UX/API/data change, architecture-shaping refactor, new infra flow) | ✅ **Mandatory** |
| Bug fix from a concrete broken symptom | ❌ WF-3 Wave 0 (Sn Dev survey) already covers this |
| Read-only inspection / status / explanation | ❌ No gate |
| WF-6 Research/POC | ❌ The workflow itself is research |

If classification is uncertain, run the gate. A research agent costs minutes; a design built on a wrong assumption costs waves.

---

## Core Rule

**The Lead does not do deep research inline.** The Lead's context window is the scarcest resource in the session — it must hold the Human conversation, the gates, and the orchestration state. Reading dozens of files inline burns that context and degrades every later decision.

Instead, the Lead spawns research agents (background, parallel), merges their findings into `.state/{TASK_ID}/research.md`, and works from that artifact.

**Hard stop:** do not run the detailed Q&A loop (Brainstorm Step 3), propose approaches (Step 4), or present a proposed design (Step 5) until `research.md` exists and the Lead has read it. While research agents run, the Lead may ask the Human 1-2 high-level *intent* questions (goal, motivation, scope) — those don't depend on codebase knowledge and the waiting time is free.

---

## Who to Spawn

Researchers are **one-shot** agents: no `name`, `model: "opus"`, `run_in_background: true`, spawned in parallel where both apply. They read a lot and return one report; nobody messages them again, so a teammate context window would sit idle at full cost.

| Researcher | Role | When | Covers |
|------------|------|------|--------|
| **Codebase researcher** | Sn Dev (or Explore-type agent) | Always | Relevant files/routes/services/schemas/tests/configs, existing patterns and conventions, recent related changes, monorepo boundaries, existing docs/specs in the repo, **Convention Anchor** (the reference feature the new work must look like, with example paths per layer) |
| **External researcher** | Sn Dev with Context7 MCP / web search | When the work touches a framework, library, external API, or infra tool where current docs matter | Current best practices, version-specific APIs, known pitfalls, migration notes, recommended patterns for the exact versions used in the repo |

Scaling: for a small change in a well-known area, one combined researcher covering both scopes is enough. For larger work or framework-heavy work, spawn both in parallel. The Lead decides — but the default for brainstorm-required work is to spawn, not to skip.

The external researcher should verify versions against the repo (`package.json`, lockfiles, etc.) before researching — best practices for the wrong major version are worse than no research.

---

## Research Agent Prompt Template

```
<!-- spawn: model=opus mode=one-shot -->
คุณคือ Sn Dev — research ก่อนออกแบบ ไม่แก้ไฟล์ใดๆ ทั้งสิ้น (read-only)

## งาน
Research เพื่อเตรียม design discussion สำหรับ: {คำขอของ Human 1-3 บรรทัด}

## Scope
- Repo: {repo path}
- Codebase: หาไฟล์/route/service/schema/test ที่เกี่ยวข้อง, pattern และ convention ที่ใช้อยู่, การเปลี่ยนแปลงล่าสุดที่เกี่ยวข้อง (git log), ข้อจำกัดจาก architecture
- Convention Anchor: หา **feature ต้นแบบ** ที่งานใหม่ต้องทำให้เหมือน ไล่ตามลำดับ: `.context/` (conventions/standards/feature docs) → `CLAUDE.md`/`AGENTS.md` ของ repo → ถ้าไม่มี ให้เลือก feature ล่าสุดที่สมบูรณ์ที่สุดในพื้นที่เดียวกับงานนี้ ส่งกลับชื่อ feature + เหตุผลที่เลือก + path ตัวอย่างแยกตามชั้น: route, schema, service, repository, test, component/text ถ้าหาไม่ได้จริงๆ ให้บอกตรงๆ ว่าไม่เจอ ห้ามแต่งขึ้นมา
- External (ถ้าเกี่ยว): ใช้ Context7/web search หา best practices + API ของ {framework/library} ตาม version ที่ repo ใช้จริง — เช็ค version จาก package.json ก่อน

## สิ่งที่ต้องส่งกลับ
Markdown ตาม Research Report template (Lead จะ merge เป็น research.md):
- Relevant Code Map, Existing Patterns, Convention Anchor, Recent Changes, External Findings, Constraints, Open Questions
- อ้าง file:line ทุกครั้งที่อ้างโค้ด
- ถ้าหาไม่เจอ/ไม่แน่ใจ ให้บอกตรงๆ ว่าไม่เจอ ห้ามเดา
```

Run the Prompt Validation Checklist (`validation.md`) before spawning, like every other spawn.

---

## Research Report Template

Each researcher returns this; the Lead merges into one `.state/{TASK_ID}/research.md`:

```markdown
# Research — {TASK_ID}

## Relevant Code Map
- path/to/file.ts:line — what it does, why it matters to this task

## Existing Patterns & Conventions
- How the repo already solves similar problems (with file references)

## Convention Anchor
- Reference feature: {name} — {why it is the standard for this area: `.context/` doc, repo `CLAUDE.md`/`AGENTS.md`, or the most complete recent feature in the same area}
- Example paths per layer: route `{path}`, schema `{path}`, service `{path}`, repository `{path}`, test `{path}`, component/text `{path}`
- Say so plainly if no anchor could be found — do not invent one

## Recent Related Changes
- Commits/PRs touching this area recently, if relevant

## External Findings (if applicable)
- Library/framework: {name}@{version actually in repo}
- Best practices / pitfalls / version-specific notes, with sources

## Constraints
- Framework, monorepo boundary, architecture, or policy constraints found

## Open Questions for the Human
- Things research could NOT answer — these feed the brainstorm Q&A
```

---

## Lead Action After Research

1. Read the reports, merge into `.state/{TASK_ID}/research.md`
   - If no Convention Anchor could be found, ask the Human one question about which existing feature should be the standard — send it together with the 1-2 intent questions, not as a separate round trip
2. Update the state file (`state-management.md`) so the gate survives a context reset
3. Use it immediately in the Brainstorm Protocol:
   - **Step 2 summary to the Human** comes from `research.md`, not from a quick inline scan
   - **Lead the Q&A with the highest-risk finding.** If research surfaced an ambiguity or contradiction that could invalidate the whole request (e.g. the data/model the Human referred to does not exist in the repo, or the request conflicts with an existing contract), raise it in the very first message — before any preference-style question. A wrong premise discovered late wastes every gate after it.
   - **Step 3 Q&A** asks only what research could not answer — never ask the Human something `research.md` already answers
   - **Step 4 approaches** must reference real patterns/constraints found in research
4. Pass `research.md` as supporting context to Wave 0 agents and later waves

`research.md` is supporting context, like `wave-0-impact.md`. It is never canonical — requirements and design intent live in `.state/{TASK_ID}/spec.md` after Human approval.

---

## Anti-Patterns

Do not skip the gate because the task "sounds simple" or the area "feels familiar." Familiarity from a previous session is not knowledge of the current state of the repo.

Do not research inline as the Lead "just to be quick." That trades the whole session's decision quality for a few minutes.

Do not ask the Human questions that `research.md` already answers — it signals the research was wasted and burns the Human's patience.

Do not let the external researcher answer from memory. The point of the external scope is *current* docs for the *exact versions* in the repo; require Context7/web verification.

Do not treat `research.md` as a spec. It informs the design; it does not decide it.
