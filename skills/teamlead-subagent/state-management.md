# State Management & Resume Flow

Persistent state for TeamLead-SubAgent. Loaded at: conversation start (resume check), after each wave (state update), on error.

---

## 1. Directory Structure

```
.state/
├── teamlead.json              <- state tracker (1 active task)
└── THUN-XX/                   <- folder per task
    ├── research.md               <- merged Research Report (Research Gate, before brainstorm)
    ├── spec.md                   <- canonical spec (after Human approval)
    ├── plan.md                   <- canonical plan
    ├── wave-0-impact.md          <- merged Impact Report
    ├── wave-1-design.md          <- AC + architecture
    ├── wave-2-implementation.md  <- QA Verify Report + Dev changed files
    ├── wave-3-review.md          <- merged review reports
    ├── final-verification.md     <- Final Verification wave results
    └── ...                       <- wave-N-{label}.md
```

**Local ignore requirement:** `.state/` is local session state and must never be committed. Prefer adding `.state/` to `.git/info/exclude`, which keeps the ignore rule local to the working copy and avoids product config churn.

Do not require Lead to edit product config before writing state. Only edit `.gitignore` for `.state/` if the Human explicitly approves it or delegates that config change to the appropriate agent.

---

## 2. State File Schema

File: `.state/teamlead.json`

### Complete Example

```json
{
  "stateVersion": 1,
  "taskId": "THUN-48",
  "workflow": "WF-1",
  "branch": "feature/THUN-48-product-filter",
  "currentWave": 2,
  "totalWaves": 4,
  "batchOrder": "1,2,3",
  "currentBatch": "2",
  "commits": [
    { "unit": 1, "hash": "abc1234", "message": "feat(filter): add product filter query schema + route", "batch": "1" }
  ],
  "waves": {
    "0": {
      "status": "completed",
      "agents": [
        { "role": "SA", "name": null, "mode": "one-shot", "model": "opus", "mission": "impact check", "status": "completed" },
        { "role": "Sn Dev", "name": null, "mode": "one-shot", "model": "opus", "mission": "impact check", "status": "completed" }
      ],
      "outputFile": ".state/THUN-48/wave-0-impact.md"
    },
    "1": {
      "status": "completed",
      "agents": [
        { "role": "BA", "name": null, "mode": "one-shot", "model": "fable", "mission": "analyze AC from spec", "status": "completed" },
        { "role": "SA", "name": null, "mode": "one-shot", "model": "fable", "mission": "design API + data model", "status": "completed" }
      ],
      "outputFile": ".state/THUN-48/wave-1-design.md"
    },
    "2": {
      "status": "in_progress",
      "agents": [
        { "role": "QA", "name": "qa-THUN-48", "mode": "teammate", "model": "opus", "mission": "write tests from AC, verify Dev until green", "status": "running" },
        { "role": "Dev", "name": "dev-THUN-48", "mode": "teammate", "model": "opus", "mission": "implement Task 1-2 until QA tests pass", "status": "running" }
      ],
      "outputFile": null
    },
    "3": {
      "status": "pending",
      "agents": [],
      "outputFile": null
    }
  },
  "updatedAt": "2026-03-30T14:30:00Z"
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `stateVersion` | number | Schema version, always `1` for now. Increment on breaking changes. The `name`/`mode`/`model` agent fields and the `batchOrder` / `currentBatch` / `commits` fields are additive, so `1` still applies. |
| `taskId` | string | Task folder ID. Default to a short slug (e.g., `"product-filter"`) unless external tracking or branch naming requires an exact ID such as `"THUN-48"`. |
| `workflow` | string | Active workflow (e.g., `"WF-1"`, `"WF-3"`). |
| `branch` | string | Git branch created for this task. |
| `currentWave` | number | Index of the wave currently being executed (0-based). |
| `totalWaves` | number | Total number of waves planned for this workflow. |
| `batchOrder` | string | The plan's Batch Order as a comma-separated list of batch ids (e.g. `"1,2,3"`). Additive field. |
| `currentBatch` | string | Id of the batch the pair is working on right now (e.g. `"2"`). Lets a resume know where the loop stopped. Additive field. |
| `commits` | array | Commits made so far, one entry per Commit Plan unit: `{ "unit": 1, "hash": "abc1234", "message": "...", "batch": "1" }`. Lead appends an entry when Dev reports a commit. Additive field. |
| `waves` | object | Keyed by wave index (`"0"`, `"1"`, ...). Each wave has `status`, `agents`, `outputFile`. |
| `waves[N].status` | string | Wave status: `pending` / `in_progress` / `completed` / `failed` |
| `waves[N].agents` | array | Array of agent entries with `role`, `name`, `mode`, `model`, `mission`, `status`. |
| `waves[N].agents[].role` | string | Agent role: `SA`, `BA`, `Sn Dev`, `Dev`, `QA` |
| `waves[N].agents[].name` | string or null | Teammate name (`dev-{taskId}`, `qa-{taskId}`, or cross-repo `dev-api` / `qa-web`). `null` for one-shot agents. Required for re-spawn after a context reset. |
| `waves[N].agents[].mode` | string | Spawn mode: `one-shot` (no `name`, `run_in_background: true`) or `teammate` (has `name`, no `run_in_background`). |
| `waves[N].agents[].model` | string | Model passed on the spawn: `opus` or `fable`, per the Model Policy in `SKILL.md`. Never empty. |
| `waves[N].agents[].mission` | string | Short description of the agent's task. |
| `waves[N].agents[].status` | string | Agent status: `pending` / `running` / `completed` / `failed` |
| `waves[N].outputFile` | string or null | Path to the wave output markdown file. `null` until wave completes. |
| `updatedAt` | string | ISO 8601 timestamp of last state write. |

### Status Values

**Wave statuses:**
- `pending` -- not started yet
- `in_progress` -- at least one agent is running
- `completed` -- all agents completed, output file written
- `failed` -- at least one agent failed and was not recovered

**Agent statuses:**
- `pending` -- not spawned yet
- `running` -- spawned, awaiting result
- `completed` -- returned output successfully
- `failed` -- crashed, timed out, or killed by conversation reset

---

## 3. Write Rules

### Atomic Write

Always write state files atomically to prevent corruption, in **one** Bash command (write the temp file and rename it in the same invocation, so it costs one turn, not two):

```bash
cat > .state/teamlead.json.tmp <<'EOF'
{ ...json... }
EOF
mv .state/teamlead.json.tmp .state/teamlead.json
```

Never write directly to `teamlead.json` -- a crash mid-write would corrupt the file. When a wave output file is written at the same time, put it in the same command.

### When to Write

| Trigger | Action |
|---------|--------|
| Wave starts | Set `waves[N].status = "in_progress"`, set agents to `running`, write state |
| Agent returns | Update agent `status` to `completed` or `failed`, write state |
| Dev reports a commit | Append the entry to `commits` (`unit`, `hash`, `message`, `batch`), advance `currentBatch`, write state |
| Wave completes (all agents done) | Set `waves[N].status = "completed"`, write output file, write state |
| Wave fails (any agent failed, not recoverable) | Set `waves[N].status = "failed"`, write state |

### Wave Output Files

When a wave completes, Lead summarizes all agent outputs into a single markdown file:

- File path: `.state/{taskId}/wave-{N}-{label}.md`
- Label convention: `impact`, `design`, `implementation`, `review`, plus `root-cause` for WF-3, `fix` (WF-3), `api` / `frontend` (WF-5), `final-verification`
- Content: Lead-written summary combining all agent outputs for that wave
- This file becomes context input for subsequent waves

### Cleanup (Delete)

When to delete state:

1. Task is fully complete
2. Validation Gate passed
3. Every teammate has received `shutdown_request` (the Validation Gate does this after all items PASS; if the Human cancels mid-task, shut the teammates down first)
4. Summary delivered to Human
5. Human acknowledges

Only after Human acknowledgement: delete the entire task folder (`.state/{taskId}/`) and the state file (`.state/teamlead.json`). Do not delete state immediately after delivering the summary.

---

## 4. Resume Flow

On every conversation start, check for existing state:

```
Start conversation
  |
  v
Check: does .state/teamlead.json exist?
  |
  +-- NOT FOUND --> Normal start. No state to resume.
  |
  +-- FOUND --> Read state file
        |
        v
      Read all completed wave output files
      (files listed in waves[N].outputFile where status=completed)
        |
        v
      Mark recovery: any agent with status=running --> set to "failed"
      (conversation reset = agent was killed, cannot be running)
        |
        v
      Mark every agent with mode="teammate" as "failed" regardless of
      recorded status -- teammates never survive a context reset or /resume
        |
        v
      Identify stuck point:
        - Which wave failed?
        - Which agents failed?
        - What was their mission?
        |
        v
      Tell Human:
        "THUN-{taskId} stuck at Wave {N} ({wave mission summary}) -- resume?"
        Show: completed waves, failed agents, what remains
        |
        v
      Human says YES (resume):
        - Re-spawn only failed agents
        - Re-spawn teammates with the SAME name recorded in state, and
          re-spawn the partner too -- a Dev/QA pair is always re-created
          together, never one half of it
        - Tell the re-spawned pair which commit units already exist (from
          `commits`: unit, hash, message) and which batch was in progress
          (`currentBatch` within `batchOrder`), so they continue from the
          next batch instead of redoing committed work
        - Pass all completed wave output files as context
        - Continue from the failed wave
        |
        v
      Human says NO:
        - Ask: "Cleanup? (delete state files + revert branch changes?)"
        - If cleanup YES: delete .state/{taskId}/ + teamlead.json
        - If cleanup NO: leave files in place, start fresh task
```

### Resume Context Loading

When resuming, provide re-spawned agents with:

1. All completed wave output files (read each `.md` file)
2. The original task description (from Human's previous request, reconstructed from state + output files)
3. Any partial output from the failed wave (if the output file was partially written)
4. For a re-spawned teammate: its recorded `name` and `mission`, plus the Partner section pointing at the freshly re-spawned partner
5. The commit units already landed (`commits`) and the batch that was in progress (`currentBatch` / `batchOrder`), so the pair resumes at the next batch

This ensures agents have full context even after a conversation reset.

---

## 5. Error Handling

| Case | Action |
|------|--------|
| **State file parse fail** (invalid JSON) | Notify Human: "State file corrupted." Backup as `teamlead.json.bak`. Start fresh -- ask Human for task context. |
| **Agent timeout / crash** | Mark agent `status = "failed"` in state. Mark wave `status = "failed"`. Ask Human: "Agent {role} ({mission}) crashed -- re-spawn?" |
| **Human cancel mid-task** | Ask Human: "Cleanup? Delete state files + task folder?" If yes, delete `.state/{taskId}/` and `teamlead.json`. If branch should be reverted, confirm with Human before any destructive git operation. |
| **Teammate idle notification never arrives / teammate errored** | Mark that agent `status = "failed"` in state. Ask Human: "Teammate {name} ({mission}) is not responding -- re-spawn the pair?" Re-spawn both halves together, never one alone. |
| **Output file missing** (state says completed but file not found) | Wave is incomplete. Set wave `status = "failed"`. Re-run the entire wave. |
| **State vs output files mismatch** (output files exist but state disagrees) | **Trust output files.** Read all existing `wave-*.md` files in the task folder. Rebuild state from output files. Notify Human of the discrepancy. |
| **Task folder missing** (state exists but no folder) | Treat as fresh start. Delete stale state file. Notify Human. |
| **Disk write failure** | Retry once. If still fails, notify Human: "Cannot write state -- proceeding without persistence." |

---

## 6. Impact Report Template

Used by SA and Sn Dev in Wave 0. Both roles use the same template:

```markdown
## Impact Report -- {Role}

### Files/Modules Affected
- path/to/file.ts -- reason this file is affected
- path/to/another.ts -- reason

### Risks / Blockers
- [BLOCKER] description (must resolve before starting implementation)
- [RISK] description (must be careful during implementation)

### Constraints
- constraint description

### Recommendations
- recommendation description
```

### Lead Merge Process

After Wave 0 completes, Lead merges SA + Sn Dev reports:

1. Read both Impact Reports
2. Combine into single `wave-0-impact.md` in the task folder
3. Tag each item with its source: `(SA)` or `(Sn Dev)`
4. Deduplicate items that both roles flagged -- keep the more detailed version, note both sources
5. Blockers go to the top

### Merged Output Format

```markdown
# Wave 0 -- Impact Report (Merged)

Task: THUN-{id}
Workflow: WF-{n}
Sources: SA, Sn Dev

## Blockers
- [BLOCKER] (SA) description
- [BLOCKER] (Sn Dev) description

## Files/Modules Affected
- path/to/file.ts -- reason (SA)
- path/to/other.ts -- reason (Sn Dev)
- path/to/shared.ts -- reason (SA + Sn Dev)

## Risks
- [RISK] (SA) description
- [RISK] (Sn Dev) description

## Constraints
- (SA) constraint
- (Sn Dev) constraint

## Recommendations
- (SA) recommendation
- (Sn Dev) recommendation
```

This merged file is sent as context to **all agents in every subsequent wave**.
