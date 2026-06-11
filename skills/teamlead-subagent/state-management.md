# State Management & Resume Flow

Persistent state for TeamLead-SubAgent. Loaded at: conversation start (resume check), after each wave (state update), on error.

---

## 1. Directory Structure

```
.state/
├── teamlead.json              <- state tracker (1 active task)
└── THUN-XX/                   <- folder per task
    ├── research.md            <- merged Research Report (Research Gate, before brainstorm)
    ├── wave-0-impact.md       <- merged Impact Report
    ├── wave-1-design.md       <- AC + architecture
    ├── wave-2-tests.md        <- test plan / test file list
    └── ...                    <- wave-N-{label}.md
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
  "totalWaves": 5,
  "waves": {
    "0": {
      "status": "completed",
      "agents": [
        { "role": "SA", "mission": "impact check", "status": "completed" },
        { "role": "Sn Dev", "mission": "impact check", "status": "completed" }
      ],
      "outputFile": ".state/THUN-48/wave-0-impact.md"
    },
    "1": {
      "status": "completed",
      "agents": [
        { "role": "BA", "mission": "analyze AC from spec", "status": "completed" },
        { "role": "SA", "mission": "design API + data model", "status": "completed" }
      ],
      "outputFile": ".state/THUN-48/wave-1-design.md"
    },
    "2": {
      "status": "in_progress",
      "agents": [
        { "role": "QA", "mission": "write tests from AC", "status": "running" }
      ],
      "outputFile": null
    },
    "3": {
      "status": "pending",
      "agents": [],
      "outputFile": null
    },
    "4": {
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
| `stateVersion` | number | Schema version, always `1` for now. Increment on breaking changes. |
| `taskId` | string | Task folder ID. Default to a short slug (e.g., `"product-filter"`) unless external tracking or branch naming requires an exact ID such as `"THUN-48"`. |
| `workflow` | string | Active workflow (e.g., `"WF-1"`, `"WF-3"`). |
| `branch` | string | Git branch created for this task. |
| `currentWave` | number | Index of the wave currently being executed (0-based). |
| `totalWaves` | number | Total number of waves planned for this workflow. |
| `waves` | object | Keyed by wave index (`"0"`, `"1"`, ...). Each wave has `status`, `agents`, `outputFile`. |
| `waves[N].status` | string | Wave status: `pending` / `in_progress` / `completed` / `failed` |
| `waves[N].agents` | array | Array of agent entries with `role`, `mission`, `status`. |
| `waves[N].agents[].role` | string | Agent role: `SA`, `BA`, `Sn Dev`, `Dev`, `QA` |
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

Always write state files atomically to prevent corruption:

```
1. Write content to .state/teamlead.json.tmp
2. Rename .state/teamlead.json.tmp to .state/teamlead.json
```

Never write directly to `teamlead.json` -- a crash mid-write would corrupt the file.

### When to Write

| Trigger | Action |
|---------|--------|
| Wave starts | Set `waves[N].status = "in_progress"`, set agents to `running`, write state |
| Agent returns | Update agent `status` to `completed` or `failed`, write state |
| Wave completes (all agents done) | Set `waves[N].status = "completed"`, write output file, write state |
| Wave fails (any agent failed, not recoverable) | Set `waves[N].status = "failed"`, write state |

### Wave Output Files

When a wave completes, Lead summarizes all agent outputs into a single markdown file:

- File path: `.state/{taskId}/wave-{N}-{label}.md`
- Label convention: `impact`, `design`, `tests`, `implementation`, `review`
- Content: Lead-written summary combining all agent outputs for that wave
- This file becomes context input for subsequent waves

### Cleanup (Delete)

When to delete state:

1. Task is fully complete
2. Validation Gate passed
3. Summary delivered to Human
4. Human acknowledges

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

This ensures agents have full context even after a conversation reset.

---

## 5. Error Handling

| Case | Action |
|------|--------|
| **State file parse fail** (invalid JSON) | Notify Human: "State file corrupted." Backup as `teamlead.json.bak`. Start fresh -- ask Human for task context. |
| **Agent timeout / crash** | Mark agent `status = "failed"` in state. Mark wave `status = "failed"`. Ask Human: "Agent {role} ({mission}) crashed -- re-spawn?" |
| **Human cancel mid-task** | Ask Human: "Cleanup? Delete state files + task folder?" If yes, delete `.state/{taskId}/` and `teamlead.json`. If branch should be reverted, confirm with Human before any destructive git operation. |
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
