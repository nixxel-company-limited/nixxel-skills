Shared rules (Wave 0, Validation Gate, summary format) are in `common.md` — read it first.

## WF-6: Research / POC

**Total waves: 1** (Wave 1 only, NO Wave 0, NO Validation Gate)

### NO Wave 0

This workflow skips Wave 0 entirely. The workflow itself IS research -- a separate impact check would be redundant.

---

### Wave 1 -- Research

**Agents:**
- **Sn Dev** (one-shot, `opus`, background) -- research the topic, analyze options, produce recommendation

**Context for agents:** Research question / POC scope from Human

**Lead action after wave:**
- Return Sn Dev's recommendation directly to Human for decision
- No further waves -- Human decides next steps

---

### NO Validation Gate

No code changes are produced, so Validation Gate does not apply.

**Notes:**
- Simplest workflow -- single wave, one one-shot `opus` agent, no teammates
- Research reads a lot and edits nothing, so it is never a teammate
- Output goes directly to Human for decision-making
- If Human decides to proceed with implementation after research, Lead starts a new workflow (WF-1, WF-2, etc.) using the research output as input context
