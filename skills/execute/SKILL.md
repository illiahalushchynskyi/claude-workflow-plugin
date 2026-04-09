---
name: workflow:execute
description: Orchestrate workflow execution - manages mode selection and step sequencing between implementer and verifier agents
---

# Workflow Execute Skill (Orchestrator)

## Overview

The `execute` skill orchestrates the entire workflow execution. It reads the PLAN.md to understand the workflow mode (1 or 2), step sequence, and current progress, then coordinates the timing and flow of implementer and verifier agents.

The execute skill is mode-aware and handles:

- **Mode 1:** Step-by-step with human approval pauses after each step
- **Mode 2:** End-to-end with verification gates between steps, single human approval at end

## When to Use

Use `execute` when:

- A workflow has been bootstrapped (PLAN.md and step files exist)
- You're ready to begin or resume implementation
- You want to continue from where the previous session left off
- You need the system to manage step sequencing automatically

## Input

The execute skill:

1. **Reads** `.workflow/TASK_NAME/PLAN.md` to determine:
   - Mode (1 or 2)
   - List of steps and their current statuses
   - Overall workflow status
   - Which step to start/resume

2. **Checks Step Files**:
   - Reads `.workflow/TASK_NAME/steps/step-N.md` for each step
   - Determines which steps are `pending`, `implementation`, `verification`, `complete`, or `needs-fix`
   - Identifies where to resume

3. **Reads Config**:
   - `.workflow/TASK_NAME/.workflow-config.json` for settings
   - Notification preferences
   - Auto-advance settings

## Mode 1: Step-by-Step Verification Flow

```
[Start execute]
  ↓
Check PLAN.md status
  ↓
For each step (in order):
  ├─ Step in "pending" or "implementation"?
  │   ├─ Yes: Signal implementer to work on step-N
  │   │       Wait for status: verification
  │   │
  │   └─ No: Continue to next check
  │
  ├─ Step in "verification"?
  │   ├─ Yes: Signal verifier to test step-N
  │   │       Wait for status: complete OR needs-fix
  │   │
  │   ├─ If complete:
  │   │   ├─ Update PLAN.md step table: step-N status → complete
  │   │   ├─ **PAUSE & SIGNAL HUMAN**
  │   │   ├─ Wait for human approval:
  │   │   │   ├─ [Approve] → Advance to next step
  │   │   │   └─ [Request Changes] → Signal implementer to fix
  │   │   └─ Continue loop
  │   │
  │   └─ If needs-fix:
  │       ├─ Update PLAN.md with issue notes
  │       └─ Signal implementer to fix (mark step: implementation)
  │
  └─ Step in "complete"?
      ├─ Yes: Move to next step
      └─ No: End (shouldn't reach here)

[All steps complete]
  ↓
Update PLAN.md status: ready-for-review
  ↓
**PAUSE & SIGNAL HUMAN**
  ↓
Wait for final human approval:
  ├─ [Approve] → Signal finalize skill
  └─ [Request Changes] → Go back to identified problematic step
```

## Mode 2: End-to-End with Verification Gates

```
[Start execute]
  ↓
Check PLAN.md status
  ↓
For each step (in order):
  ├─ Check previous step (N-1):
  │   ├─ If N = 1: No previous, continue
  │   ├─ If N > 1 and N-1 status ≠ "complete": **PAUSE**
  │   │   ├─ Reason: Mode 2 requires sequential verification
  │   │   ├─ Wait for verifier to complete step N-1
  │   │   └─ [Resume execute when N-1 completes]
  │   │
  │   └─ If N-1 complete: Proceed
  │
  ├─ Step in "pending"?
  │   ├─ Yes: Signal implementer to work on step-N
  │   │       Wait for status: verification
  │   │
  │   └─ No: Continue to next check
  │
  ├─ Step in "implementation"?
  │   ├─ Yes: Continue waiting for status: verification
  │   │
  │   └─ No: Continue to next check
  │
  ├─ Step in "verification"?
  │   ├─ Yes: Signal verifier to test step-N
  │   │       Wait for status: complete OR needs-fix
  │   │
  │   ├─ If complete:
  │   │   ├─ Update PLAN.md step table
  │   │   ├─ **NO HUMAN PAUSE HERE**
  │   │   └─ Continue to next step
  │   │
  │   └─ If needs-fix:
  │       ├─ Update PLAN.md with issue notes
  │       └─ Signal implementer to fix (mark: implementation)
  │           └─ Loop back to this step (don't advance)
  │
  └─ Step in "complete"?
      ├─ Yes: Move to next step
      └─ No: End (shouldn't reach here)

[All steps complete]
  ↓
Update PLAN.md status: ready-for-review
  ↓
**PAUSE & SIGNAL HUMAN** (first pause in Mode 2)
  ↓
Wait for final human approval:
  ├─ [Approve] → Signal finalize skill
  └─ [Request Changes] → Go back to identified problematic step
```

## State Management

### PLAN.md Status Transitions

```
planning 
  → executing (execute marks this when starting)
  → ready-for-review (execute marks when all steps complete)
  → approved (human approves)
  → complete (finalize marks after commit)
```

### Step-N.md Status Transitions

Execute doesn't directly update step statuses; it:

- **Reads** step statuses
- **Signals agents** based on status
- **Waits** for agents to update status
- **Updates PLAN.md step table** based on current step status

Status transitions happen by:
- **Implementer:** pending → implementation → verification
- **Verifier:** verification → complete or needs-fix
- **Execute:** Updates PLAN.md after each agent completes

## Decision Logic

Execute uses these rules to determine next action:

### Check: Which steps are in which status?

```
Loop through steps 1 to N:
  1. Count complete steps
  2. Find first pending/incomplete step
  3. Check if previous step complete (Mode 2 requirement)
  4. Determine what to signal next
```

### Rule: Mode 1 Sequential Execution

```
Previous step:
  - Must be complete before advancing
  - If complete: human approval required before signaling next implementer

Current step:
  - Signal implementer when ready
  - Signal verifier when implementation done
  - Signal human for approval when verification done
```

### Rule: Mode 2 Sequential Execution

```
Previous step:
  - Must be complete before signaling next implementer
  - No human pause between steps (but verifier gates advancement)

Current step:
  - Signal implementer immediately when previous verified
  - Signal verifier when implementation done
  - NO human pause here (continue to next step)

Final:
  - After all steps verified: single human approval
```

## Signaling Agents

Execute signals agents by:

1. **Writing status to step-N.md** (if needed)
2. **Printing clear messages** to user/claude-code
3. **Waiting for status changes** in step files
4. **Monitoring PLAN.md** for overall progress

Signal examples:

```
Mode 1:
  → "Ready for implementer: step-1-auth-setup"
  → "[Waiting for implementer to complete...]"
  → "Step 1 verification ready. Calling verifier..."
  → "[Waiting for verifier...]"
  → "✓ Step 1 verified. Ready for human approval."
  → "[Waiting for human approval...]"
  → "✓ Step 1 approved. Ready to proceed to step-2."

Mode 2:
  → "Ready for implementer: step-1-auth-setup"
  → "[Waiting for implementer...]"
  → "Step 1 verification ready. Calling verifier..."
  → "[Waiting for verifier...]"
  → "✓ Step 1 verified. Implementer can now work on step-2."
  → "Ready for implementer: step-2-database"
  → "[Waiting for implementer...]"
  → "[... loop continues for all steps ...]"
  → "All steps verified! Ready for final human approval."
```

## Progress Display

Execute prints progress:

```
Workflow: feature-auth-system
Mode: 1 (Step-by-Step Verification)
Status: executing

Steps Progress:
  [x] Step 1: Add JWT token endpoint (complete)
  [x] Step 2: Add token validation middleware (complete)
  [ ] Step 3: Add logout endpoint (needs-fix)
  [ ] Step 4: Add tests (pending)

Current: Awaiting implementer fixes for Step 3
Next: Verifier will re-test Step 3 when ready
```

## Resume & Checkpointing

Execute can resume from any state:

- **First run:** Starts at step 1
- **Resume:** Checks PLAN.md status
  - If "executing": resumes from current incomplete step
  - If "ready-for-review": waits for human decision
  - If "approved": signals finalize
  - If "complete": nothing to do

Example:

```
Previous session:
  Step 1: complete ✓
  Step 2: verification (waiting for verifier)
  Step 3: pending

New session runs execute:
  Reads PLAN.md: status = executing
  Finds step 2 in verification → calls verifier for step-2
  No need to re-run step 1
```

## Error Handling

If execute encounters issues:

1. **Agent didn't update status**
   - Wait timeout → signal human "agent appears stuck"
   - May need to manually update step file or restart agent

2. **Step missing verification criteria**
   - Can't signal verifier
   - Return to human: "Step N needs clearer criteria before verification"

3. **Mode 2: Previous step not complete**
   - Print: "Cannot proceed to Step N: Step N-1 not yet verified"
   - Wait for verifier to complete N-1

4. **PLAN.md corrupted or missing**
   - Error: "PLAN.md not found or invalid"
   - Cannot proceed; need to restore from git

## Related Skills

- `workflow:bootstrap` — Creates PLAN.md that execute reads
- `workflow:implementer` — Works when execute signals
- `workflow:verifier` — Works when execute signals
- `workflow:finalize` — Called by execute after final human approval
