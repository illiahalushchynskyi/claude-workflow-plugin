---
name: workflow:execute
description: Orchestrate workflow execution - read state, confirm subagent execution, ask about workflow mode
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
6. Check current_step and its status from progress.json

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

**If workflow_status is NOT "initialized" (resuming):**
- Skip this step, continue to Step 2

---

### Step 2: Show What Will Happen Next

Based on current step status, tell user what comes next:

```
Next action will be:
- If status is pending/needs-fix → Implement step {N}
- If status is verification → Verify step {N}
- If status is awaiting-approval → Ask for approval (Step Manual Approve (mode = 1))
- If status is complete → Move to step {N+1}
- If all complete → Finalize workflow
```

**With current Mode {1 or 2}:**
```
Step Manual Approve (mode = 1): After implementer finishes → Verifier tests → You approve → Next step
Final Approve (mode = 2): After implementer finishes → Verifier tests → Automatic next step (no approval)
```

---

### Step 3: Ask About Workflow Mode (Every Execution)

**ALWAYS ask this step - user can choose/change mode:**

```
AskUserQuestion:
  question: "How should approval work for this execution?"
  header: "Workflow Approval Mode"
  options:
    - "Step Manual Approve - I will approve each step after verification"
    - "Final Approve - Approve only at the end after all steps verified"
```

**Modes Explained:**
- **Step Manual Approve**: After verifier tests each step → you approve before next step
- **Final Approve**: All steps run and verify automatically, you approve once at end

**User action:**
- Update progress.json: `mode` field = 1 (Step Manual Approve) or 2 (Final Approve)
- If workflow_status still "initialized", also set to "in-progress"

**Then continue to Step 4.**

---

### Step 4: Check If Step Approval Needed (Step Manual Approve Mode Only)

**ONLY if ALL conditions are true:**
- User chose "Step Manual Approve" mode in Step 3 (mode = 1)
- Current step status is `complete` (verifier just finished)

**If conditions met, ask for approval:**

```
AskUserQuestion:
  question: "Step {N} passed verification. Approve and continue to next step?"
  options:
    - "Approve - Mark complete and continue"
    - "Request changes - Back to implementation"
```

**If user approves:**
- Update progress.json: step status = `complete`
- Loop back to Step 1 to move to next step (if more steps exist)
- If all complete → Call finalize skill

**If user requests changes:**
- Update progress.json: step status = `needs-fix`
- Loop back to Step 1 (will go to Step 5 to implement again)

**If NOT Step Manual Approve mode:** Skip this step, continue to Step 5

**If step status is NOT `complete`:** Skip this step, continue to Step 5

---

### Step 5: Execute with Subagent (If Work Needed)

**Skip this step if:**
- Step status is `complete` and approval already handled in Step 4

**Execute ONLY if:**
- Step status is `pending` or `needs-fix` (need to implement)
- OR step status is `verification` (need to verify)

Dispatch isolated Agent subagent:

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
3. If Step Manual Approve (mode = 1) and step is now `complete`: set `awaiting_approval_since` to current ISO8601 timestamp
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
- ✅ Step 3: Ask workflow mode (EVERY execution - user can change)
- ✅ Step 4: Check approval (ONLY if Step Manual Approve AND step complete)
- ✅ Step 5: Execute with subagent (ONLY if work needed)

**NEVER skip or reorder steps.**

**NEVER:**
- ❌ Skip progress.json validation (Step 1)
- ❌ Skip subagent confirmation on first run (Step 1.5)
- ❌ Skip showing what's next (Step 2)
- ❌ Skip asking about mode (Step 3) - ASK EVERY TIME
- ❌ Dispatch implementer/verifier without Step 3 mode selection
- ❌ Ask approval (Step 4) unless Step Manual Approve mode AND step complete

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

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve
  → Set mode = 1, workflow_status = in-progress

Step 4: Check approval
  → Step 1 status = pending (not complete) → Skip

Step 5: Execute with subagent
  → Dispatch Agent(implementer step 1)
  → Agent returns, status = verification
  → Loop back to Step 1

---

/workflow:execute (Second Run)

Step 1: Read state
  → Step 1 verification, Mode = 1

Step 1.5: Skip (workflow_status is in-progress, not initialized)

Step 2: Show next
  → "Will verify step 1"

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve (can change if wanted)

Step 4: Check approval
  → Step 1 status = verification (not complete) → Skip

Step 5: Execute with subagent
  → Dispatch Agent(verifier step 1)
  → Agent returns, status = complete
  → Loop back to Step 1

---

/workflow:execute (Third Run)

Step 1: Read state
  → Step 1 complete, Mode = 1

Step 1.5: Skip (workflow_status is in-progress)

Step 2: Show next
  → "Step 1 verified - ask for approval"

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve

Step 4: Check approval (Mode = Step Manual Approve AND status = complete)
  → "Step 1 passed verification. Approve?"
  → User: Approve
  → Status = complete, loop to next step

[Repeat for steps 2 & 3...]

All steps complete → Finalize
```

---

See [EXECUTION_MODES.md](../docs/EXECUTION_MODES.md) for detailed execution method comparison.
