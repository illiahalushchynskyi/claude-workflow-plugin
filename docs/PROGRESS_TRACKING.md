# Progress Tracking System

The workflow system uses two complementary files to track execution progress:

- **`PLAN.md`** — Human-readable status summary
- **`progress.json`** — System source of truth with detailed timestamps and state

## PLAN.md (Human-Facing Summary)

Minimal, easy to read. Only contains:

```yaml
---
status: pending
mode: 1
created: 2026-04-15
---

# Feature: User Authentication

Add JWT-based authentication system with login/logout endpoints.
```

### Status Values in PLAN.md

- **`pending`** - Not started yet
- **`in-progress`** - Currently executing (or paused waiting for human approval)
- **`completed`** - All steps verified and finalized

Update frequency: execute skill updates when workflow starts, pauses, and completes.

---

## progress.json (System Source of Truth)

Comprehensive tracking with timestamps, iterations, and approvals.

### Top-Level Fields

```json
{
  "task_name": "feature-auth",
  "mode": 1,
  "workflow_status": "in-progress",
  "created": "2026-04-15",
  "started": "2026-04-15T10:30:00Z",
  "completed": null,
  "current_step": 1,
  "steps": { ... },
  "approvals": { ... }
}
```

#### Workflow Status Values

- **`initialized`** - Created by bootstrap, not started
- **`in-progress`** - Currently executing steps
- **`paused`** - Waiting (human approval in Mode 1, or error)
- **`completed`** - All steps verified and finalized

**Timeline:**
```
bootstrap
   ↓
initialized → in-progress → paused ↔ in-progress → completed
                              ↑                         ↓
                    (Mode 1: awaiting approval)    finalize called
```

---

## Step Status Lifecycle

Each step transitions through statuses as it's implemented and verified:

### Full Lifecycle (Both Modes)

```
pending
  ↓
implementation (implementer working)
  ↓
verification (verifier testing)
  ↓
[Mode 1: awaiting-approval]  [Mode 2: complete]
  ↓
[Mode 1 approval decision]
  ├→ complete (approved)
  └→ needs-fix (changes requested)
        ↓
    implementation (retry)
```

### Step Status Values

- **`pending`** - Not started
- **`implementation`** - Implementer agent is working on this step
- **`verification`** - Verifier agent is testing implementation
- **`awaiting-approval`** - *(Mode 1 only)* Verification passed, waiting for human approval
- **`needs-fix`** - Verifier found issues or human requested changes
- **`complete`** - Verified and approved (or auto-approved in Mode 2)

### Step State Fields

```json
{
  "1": {
    "name": "Add JWT generation",
    "status": "awaiting-approval",
    "iteration": 1,
    "implementation_start": "2026-04-15T10:30:00Z",
    "implementation_end": "2026-04-15T10:45:00Z",
    "verification_start": "2026-04-15T10:45:30Z",
    "verification_end": "2026-04-15T11:00:00Z",
    "awaiting_approval_since": "2026-04-15T11:00:05Z",
    "approval_date": null
  }
}
```

**Field meanings:**
- `name` - Step name from step-N.md
- `status` - Current state (see values above)
- `iteration` - How many times this step has been implemented (increments on `needs-fix`)
- `implementation_start` - When implementer agent started
- `implementation_end` - When implementer agent finished
- `verification_start` - When verifier agent started
- `verification_end` - When verifier agent finished
- `awaiting_approval_since` - When verification passed and approval wait started (Mode 1 only)
- `approval_date` - When human approved (Mode 1 only)

---

## Mode 1 vs Mode 2 Flow

### Mode 1: Step-by-Step with Human Approval

```
User: /workflow:execute
  ↓
execute loads progress.json, sets workflow_status = in-progress
  ↓
For each step:
  ├→ Step starts: status = implementation
  ├→ Dispatch implementer
  ├→ When done: status = verification
  ├→ Dispatch verifier
  ├→ Verification passes: status = awaiting-approval
  ├→ workflow_status = paused
  ├→ AskUserQuestion: "Approve step?"
  │   ├→ Approve: status = complete, approval_date set, continue
  │   └→ Changes: status = needs-fix, iteration++, loop back
  └→
  (repeat until all complete)
  ↓
All complete: workflow_status = completed
  ↓
User: /workflow:finalize → final commit
```

### Mode 2: Continuous Execution (No Approval Pauses)

```
User: /workflow:execute
  ↓
execute loads progress.json, sets workflow_status = in-progress
  ↓
For each step:
  ├→ Step starts: status = implementation
  ├→ Dispatch implementer
  ├→ When done: status = verification
  ├→ Dispatch verifier
  ├→ Verification passes: status = complete (auto)
  ├→ No human pause, continue immediately
  └→
  (repeat until all complete)
  ↓
All complete: workflow_status = completed
  ↓
User: /workflow:finalize → final commit
```

**Key difference:** Mode 1 pauses at `awaiting-approval`, Mode 2 skips to `complete`.

---

## Pause and Resume

The system supports pausing and resuming execution:

### Pausing

When workflow is in Mode 1 and a step is `awaiting-approval`:
- User doesn't approve yet, closes Claude Code
- progress.json is left with: `workflow_status = paused`, step `status = awaiting-approval`
- PLAN.md still shows `status = in-progress`

### Resuming

User runs `/workflow:execute` again later:
1. Execute loads progress.json
2. Finds first non-complete step (the one in awaiting-approval)
3. Asks user to approve/request-changes
4. Continues from where it left off
5. No re-execution of previous steps

**Important:** All timestamps and iteration counts preserved across resume/pause cycles.

---

## Approval Tracking

Mode 1 workflows track all user approvals:

```json
"approvals": {
  "mode_1_manual_approvals": [
    {
      "step": 1,
      "approved_at": "2026-04-15T11:05:00Z",
      "approved_by": "user"
    },
    {
      "step": 2,
      "approved_at": "2026-04-15T11:30:00Z",
      "approved_by": "user"
    }
  ]
}
```

This provides a complete audit trail of who approved what and when.

---

## Timestamps

All timestamps use ISO 8601 format with UTC timezone: `2026-04-15T11:30:00Z`

| Timestamp | Set By | When |
|-----------|--------|------|
| `created` | bootstrap | When workflow created |
| `started` | execute | When first step begins (initialization→in-progress) |
| `completed` | execute (during finalize) | When last step verified |
| `implementation_start` | implementer | When agent starts work |
| `implementation_end` | implementer | When agent finishes |
| `verification_start` | verifier | When agent starts tests |
| `verification_end` | verifier | When agent finishes |
| `awaiting_approval_since` | execute | When verification passes, approval wait starts |
| `approval_date` | execute | When user approves step |

---

## Files Modified By Each Skill

| Skill | PLAN.md | progress.json | step-N.md |
|-------|---------|---------------|-----------|
| bootstrap | Create | Create | Create |
| execute | Update status | Update workflow_status, step status, timestamps | — |
| implementer | — | — | Update Implementation section |
| verifier | — | — | Update Verification section |
| finalize | Update status | Set completed timestamp | — |

**Key rule:** Only `execute` manages workflow and step status fields in progress.json.

---

## Common Questions

### Q: My workflow is in Mode 1, step 2 is awaiting-approval, and I close Claude Code. How do I resume?

A: Run `/workflow:execute` again. Execute loads progress.json, sees workflow_status = paused and the awaiting-approval step, and prompts you to approve/request-changes. The workflow continues from there.

### Q: I approved step 2 but didn't see the approval_date get set. Did it work?

A: Check progress.json: if `approval_date` is set and the step `status` is `complete`, it worked. The date will be an ISO 8601 timestamp like `2026-04-15T11:05:00Z`.

### Q: What if I reject a step 3 times? Does iteration keep incrementing?

A: Yes. Each time `needs-fix` is set, `iteration` increments. So if a step fails verification 3 times, iteration will be 4 (started at 1, failed 3 times = iterations 1, 2, 3, 4).

### Q: In Mode 2, does the workflow ever pause?

A: No. Mode 2 steps go `pending → implementation → verification → complete` automatically. The workflow only pauses if there's an error (e.g., verifier fails tests).

### Q: Can I manually edit progress.json to change status?

A: You shouldn't. Execute manages status transitions. If you need to manually intervene, close the workflow and contact support.
