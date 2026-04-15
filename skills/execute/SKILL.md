---
name: workflow:execute
description: Orchestrate workflow execution - read state, show plan, ask about mode and execution method
---

# Workflow Execute

**⚠️ ORCHESTRATOR: Runs in MAIN SESSION**

Execute manages workflow state and coordinates step execution in strict order.

---

## Procedure - Exact Order (Non-Negotiable)

### Step 1: Read Current State

1. Check progress.json exists (if not: STOP, tell user to run `/workflow:bootstrap`)
2. Read `.workflow/{TASK_NAME}/PLAN.md` (title, description, created date)
3. Read `.workflow/{TASK_NAME}/progress.json` — **THIS IS THE SOURCE OF TRUTH**
   - Extract: task_name, mode, workflow_status, current_step, all step statuses
4. Read `.workflow/{TASK_NAME}/.workflow-config.json` (projectType, buildCommand, testCommand, migrateCommand)
5. Find current step (progress.json current_step) and its status
6. Check if mode is set in progress.json:
   - If mode is null → will ask user in Step 3
   - If mode is set → continue to Step 1.5

### Step 1.5: Confirm Subagent Execution (First Run Only)

**If workflow_status is "initialized" (first execution):**

This workflow uses subagents for implementation and verification. Subagents are isolated AI agents that will execute steps independently.

```
AskUserQuestion:
  question: "This workflow will use subagents for implementation and verification. Continue?"
  header: "Subagent Confirmation"
  options:
    - "Agree - Continue with subagent execution"
    - "Stop - Cancel workflow execution"
```

**If user agrees:**
- Continue to Step 2

**If user stops:**
- Exit workflow, user can run `/workflow:execute` again later if they change their mind

**Output to user:**
```
Current state:
- Workflow: {TASK_NAME}
- Mode: {Step Manual Approve or Final Approve} (or 'Not set' if null)
- Current step: {N} ({STEP_NAME})
- Current status: {pending/implementation/verification/awaiting-approval/needs-fix/complete}
- Workflow status: {initialized/in-progress/paused/completed}
- Iteration: {current iteration count}
```

---

### Step 2: Show What Will Happen Next

(Updated to Step 2 after Step 1.5)

Based on current step status, tell user what comes next:

```
Next action will be:
- If status is pending/needs-fix → Implement step {N}
- If status is verification → Verify step {N}
- If status is awaiting-approval → Ask for approval (Mode 1)
- If status is complete → Move to step {N+1}
- If all complete → Finalize workflow
```

**With current Mode {1 or 2}:**
```
Mode 1: After implementer finishes → Verifier tests → You approve → Next step
Mode 2: After implementer finishes → Verifier tests → Automatic next step (no approval)
```

---

### Step 3: Ask About Workflow Mode (if mode not set)

**If progress.json mode is null (first execution):**

```
AskUserQuestion:
  question: "How should approval work?"
  header: "Workflow Mode"
  options:
    - "Step Manual Approve - I approve each step before moving to next"
    - "Final Approve - I approve once at the end"
```

**Modes Explained:**
- **Step Manual Approve**: After verifier tests each step → you approve → next step
- **Final Approve**: After all steps verified → you approve all at once

**If user chooses a mode:**
- Update progress.json: `mode` field = 1 or 2
- Update progress.json: `workflow_status` = "in-progress"

**If progress.json mode already set (resuming):**
- Skip this step, use existing mode

**Then continue with chosen/existing mode.**

---

### Step 4: Check If Step Approval Needed (Step Manual Approve Only)

**If Step Manual Approve mode AND step status is `awaiting-approval`:**

```
AskUserQuestion:
  question: "Step {N} passed verification. Approve and continue?"
  options:
    - "Approve and continue to next step"
    - "Request changes (back to implementation)"
```

**If approve:**
- Update progress.json: step status = `complete`
- Continue to next step (loop back to Step 1 if more steps)
- If all complete → Finalize
- (PLAN.md status updated only by finalize, not here)

**If request changes:**
- Update progress.json: step status = `needs-fix`
- Loop back to Step 5 (to implement again)

**If NOT Step Manual Approve mode or NOT awaiting-approval:** Skip this, continue to Step 5.

---

### Step 5: Execute with Subagent

**Execute ONLY if:**
- Step status is `pending` or `needs-fix` (need to implement)
- Step status is `verification` (need to verify)

Workflow uses subagents for execution. Dispatch isolated Agent subagent:

**If implementing (step status = pending/needs-fix):**
```python
Agent(
  description: f"Implement {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are implementing step {N} of {TASK_NAME}.

Read: .workflow/{TASK_NAME}/steps/step-{N}.md
Invoke: Skill(skill: "workflow:implementer")
Follow the skill procedure exactly.

Config:
- projectType: {PROJECT_TYPE}
- buildCommand: {BUILD_COMMAND}
- testCommand: {TEST_COMMAND}
- migrateCommand: {MIGRATE_COMMAND}

Your step is complete when:
- Code changes implemented and committed
- Tests pass
- step-{N}.md Implementation section filled with changes and results
"""
)
```

**If verifying (step status = verification):**
```python
Agent(
  description: f"Verify {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are verifying step {N} of {TASK_NAME}.

Read: .workflow/{TASK_NAME}/steps/step-{N}.md
Invoke: Skill(skill: "workflow:verifier")
Follow the skill procedure exactly.

Config:
- projectType: {PROJECT_TYPE}
- buildCommand: {BUILD_COMMAND}
- testCommand: {TEST_COMMAND}
- migrateCommand: {MIGRATE_COMMAND}

Your step is complete when:
- All tests pass
- All criteria verified
- step-{N}.md Verification section filled with results and pass/fail status
"""
)
```

**After Agent returns:**
1. Read `.workflow/{TASK_NAME}/steps/step-{N}.md` Implementation or Verification section to see what agent did
   - Check if implementation section is filled (code completed)
   - Check if verification section is filled with PASS/FAIL results
2. Update progress.json step status:
   - If implementing: set status = `verification`
   - If verifying: set status = `complete` (if all criteria pass) or `needs-fix` (if any fail)
3. If Mode 1 and step is now `complete`: set `awaiting_approval_since` to current ISO8601 timestamp
4. Loop back to Step 1 (to handle new status)

---

## Loop Until Complete

Repeat Steps 1-5 until:
- All steps have status = `complete`
- All criteria verified

Then finalize:
```python
Skill(skill: "workflow:finalize")
```

---

## Critical Rules

**ALWAYS follow Step 1-5 in EXACT order:**
- ✅ Step 1: Read state first
- ✅ Step 1.5: Confirm subagent execution (first run only)
- ✅ Step 2: Show what comes next
- ✅ Step 3: Confirm/change workflow mode (Step Manual Approve or Final Approve)
- ✅ Step 4: Handle step approval if Step Manual Approve mode
- ✅ Step 5: Execute with subagent ONLY if implementer/verifier needed

**NEVER skip or reorder steps.**

**NEVER:**
- ❌ Skip progress.json validation (Step 1)
- ❌ Skip subagent confirmation on first run (Step 1.5)
- ❌ Skip showing what's next (Step 2)
- ❌ Skip asking about mode (Step 3)
- ❌ Skip approval question if Step Manual Approve mode (Step 4)
- ❌ Dispatch implementer/verifier without using subagent (Step 5)

---

## Example Flow: 3 Steps, Step Manual Approve

```
Start: /workflow:execute

Step 1: Read state
  → Step 1 pending, Mode not set, Workflow initialized

Step 1.5: Confirm subagent execution
  → "This workflow will use subagents. Continue?"
  → User: Agree
  → Continue to Step 2

Step 2: Show next
  → "Will implement step 1, then verify, then ask for approval"

Step 3: Confirm mode
  → "How should approval work?" → User: Step Manual Approve
  → Set mode = 1, workflow_status = in-progress

Step 4: Check approval
  → Step 1 is pending, so skip approval

Step 5: Execute with subagent
  → Dispatch Agent(implementer step 1)
  → Agent returns, read step-1.md Implementation section
  → Set status = verification, loop back to Step 1

Step 1 again: Read state
  → Step 1 verification, Mode = 1 (Step Manual Approve)

Step 1.5: Confirm subagent execution
  → Skip (workflow_status is in-progress, not initialized)

Step 2: Show next
  → "Will verify step 1, then ask for approval"

Step 3: Confirm mode
  → Skip (mode already set)

Step 4: Check approval
  → Step 1 is verification (not awaiting-approval yet), skip

Step 5: Execute with subagent
  → Dispatch Agent(verifier step 1)
  → Agent returns, read step-1.md Verification section
  → Set status = complete, set awaiting_approval_since, loop back to Step 1

Step 1 again: Read state
  → Step 1 complete, awaiting approval

Step 1.5: Skip (workflow_status is in-progress)

Step 2: Show next
  → "Step 1 verified, will ask for approval"

Step 3: Skip (mode already set)

Step 4: Check approval (Mode 1 AND status = complete)
  → Ask user approval
  → User: Approve
  → Set status = complete, move to step 2

[Repeat for steps 2 & 3...]

All steps complete → Finalize
```

---

See [EXECUTION_MODES.md](../docs/EXECUTION_MODES.md) for detailed execution method comparison.
