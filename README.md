# Nixxel Skills

Skills for Claude Code by [Nixxel](https://github.com/nixxel-company-limited).

## Installation

### Install All Skills

```bash
npx skills add https://github.com/nixxel-company-limited/nixxel-skills
```

### Install Individual Skills

#### project-onboard

Onboard an AI agent into any project — auto-detect stack, research up-to-date docs, audit features (Page → API → DB mapping), interview the developer, and generate structured context files with task-based routing.

```bash
npx skills add https://github.com/nixxel-company-limited/nixxel-skills --skill project-onboard
```

**Commands:**

| Command | Description |
|---------|-------------|
| `/onboard` | Full onboard (all phases) |
| `/onboard --quick` | Phase 1+2 only (skip audit & interview) |
| `/onboard Thai` | Full onboard, output in Thai |
| `/onboard-update` | Incremental update (changed files only) |
| `/onboard-update --full` | Re-scan everything, keep interview answers |
| `/onboard-audit` | Run feature audit only |
| `/onboard-suggest` | Suggest skills for detected stack |

---

#### teamlead-subagent

TeamLead orchestration — วิเคราะห์งาน แล้ว spawn SA/BA/Sn Dev/Dev/QA agents ทำงานแทน. Lead ไม่เขียนโค้ดเอง delegate เสมอ รองรับ monorepo.

```bash
npx skills add https://github.com/nixxel-company-limited/nixxel-skills --skill teamlead-subagent
```

**Roles:**

| Role | Responsibility |
| ---- | -------------- |
| SA | Architecture, API contract, data model |
| BA | Requirement analysis, acceptance criteria |
| Sn Dev | Code review, debug, research |
| Dev | Implement feature, fix bug, refactor |
| QA | Write tests, verify behavior |

**Workflows:** Feature (with/without spec), Bug Fix, Refactor, Cross-Repo, Research/POC, Infra/CI

---

#### cloudflare

Set up a Next.js project for deployment on Cloudflare Workers using @opennextjs/cloudflare — install dependencies, configure wrangler, add build scripts, and verify with local workerd runtime.

```bash
npx skills add https://github.com/nixxel-company-limited/nixxel-skills --skill cloudflare
```

**Commands:**

| Command | Description |
|---------|-------------|
| `/cloudflare` | Full setup (steps 1-7, no deploy) |
| `/cloudflare --deploy` | Full setup + deploy to Cloudflare |

---

## How It Works

Each skill is a self-contained directory under `skills/` with a `SKILL.md` that defines its behavior. When installed, Claude Code loads the skill and makes its commands available in your session.

## License

MIT
