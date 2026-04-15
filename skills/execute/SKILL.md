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
2. Read `.workflow/{TASK_NAME}/PLAN.md` (mode: 1 or 2)
3. Read `.workflow/{TASK_NAME}/progress.json` (step statuses, workflow_status)
4. Read `.workflow/{TASK_NAME}/.workflow-config.json` (projectType, buildCommand, testCommand, migrateCommand)
5. Find first step with status ≠ "complete"
6. Determine what was last done and what's next

**Output to user:**
```
Current state:
- Workflow: {TASK_NAME}
- Mode: {1 or 2}
- Current step: {N} ({STEP_NAME})
- Current status: {pending/implementation/verification/awaiting-approval/needs-fix/complete}
- Workflow status: {initialized/in-progress/paused/completed}
```

---

### Step 2: Show What Will Happen Next

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

### Step 3: Ask About Workflow Mode (only if needed)

**Ask user:**
```
AskUserQuestion:
  question: "Is Mode {1 or 2} correct?"
  header: "Workflow Mode"
  options:
    - "Yes, Mode {current} is correct"
    - "Change to Mode {other}"
```

**If user wants to change:**
- Update PLAN.md: mode field
- Update progress.json: mode field
- Explain: "Mode 1 = step-by-step approval, Mode 2 = continuous"

**Then continue with chosen mode.**

---

### Step 4: Check If Approval Needed (Mode 1 Only)

**If Mode 1 AND step status is `awaiting-approval`:**

```
AskUserQuestion:
  question: "Step {N} passed verification. Approve and continue?"
  options:
    - "Approve and continue to next step"
    - "Request changes (back to implementation)"
```

**If approve:**
- Update progress.json: step status = `complete`
- Update PLAN.md: step row status = `complete`
- Continue to next step (loop back to Step 1 if more steps)
- If all complete → Finalize

**If request changes:**
- Update progress.json: step status = `needs-fix`
- Loop back to Step 5 (to implement again)

**If NOT Mode 1 or NOT awaiting-approval:** Skip this, continue to Step 5.

---

### Step 5: Choose Execution Method (ONLY if implementer or verifier needed)

**Only ask Step 5 if:**
- Step status is `pending` or `needs-fix` (need to implement)
- Step status is `verification` (need to verify)

**Ask user:**
```
AskUserQuestion:
  question: "How should I run implementer/verifier?"
  header: "Execution Method"
  options:
    - "Subagent Mode - I dispatch as Agent() (faster, hands-off)"
    - "Manual Mode - I invoke Skill() in this session (transparent)"
```

**THEN execute based on choice:**

#### Step 5a: Subagent Mode

Dispatch as isolated Agent subagent:

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
- Code changes implemented
- Tests pass
- step-{N}.md status = verification
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
- step-{N}.md status = complete or needs-fix
"""
)
```

**After Agent returns:**
1. Read `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
3. If implementation: status should be `verification` now
4. If verification: status should be `complete` or `needs-fix` now
5. Loop back to Step 1 (to handle new status)

#### Step 5b: Manual Mode

Invoke skill in this session:

**If implementing (step status = pending/needs-fix):**
```python
Skill(skill: "workflow:implementer")
```

The skill will guide user to:
- Read step-{N}.md goal
- Implement code changes
- Run tests (must pass)
- Commit changes
- Update step-{N}.md Implementation section
- Set status = `verification`

**If verifying (step status = verification):**
```python
Skill(skill: "workflow:verifier")
```

The skill will guide user to:
- Build project
- Run tests (must all pass)
- Verify acceptance criteria
- Update step-{N}.md Verification section
- Set status = `complete` or `needs-fix`

**After Skill returns:**
1. Read `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
3. Verify status changed correctly
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
- ✅ Step 2: Show what comes next
- ✅ Step 3: Confirm/change workflow mode
- ✅ Step 4: Handle approval if Mode 1
- ✅ Step 5: Choose execution method (Subagent or Manual) ONLY if implementer/verifier needed

**NEVER skip or reorder steps.**

**NEVER:**
- ❌ Skip progress.json validation (Step 1)
- ❌ Skip showing what's next (Step 2)
- ❌ Skip asking about mode (Step 3)
- ❌ Skip approval question if Mode 1 (Step 4)
- ❌ Choose execution method before showing state (Step 5)
- ❌ Dispatch implementer/verifier without asking how (Step 5)

---

## Example Flow: 3 Steps, Mode 1

```
Start: /workflow:execute

Step 1: Read state
  → Step 1 pending, Mode 1, Workflow initialized

Step 2: Show next
  → "Will implement step 1, then verify, then ask for approval"

Step 3: Confirm mode
  → "Is Mode 1 correct?" → User: Yes

Step 4: Check approval
  → Step 1 is pending, so skip approval

Step 5: Ask execution method
  → "Subagent or Manual?"
  → User: Subagent Mode
  → Dispatch Agent(implementer step 1)
  → Agent returns, step status = verification

Step 1 again: Read state
  → Step 1 verification, Mode 1

Step 2: Show next
  → "Will verify step 1, then ask for approval"

Step 3: Confirm mode
  → Already confirmed, skip

Step 4: Check approval
  → Step 1 is verification (not awaiting-approval yet), skip

Step 5: Ask execution method
  → "Subagent or Manual?"
  → User: Subagent Mode
  → Dispatch Agent(verifier step 1)
  → Agent returns, step status = complete

Step 1 again: Read state
  → Step 1 complete, Mode 1

Step 2: Show next
  → "Step 1 verified, will ask for approval"

Step 3: Confirm mode
  → Already confirmed, skip

Step 4: Check approval
  → Step 1 complete → Ask approval
  → User: Approve
  → Step 1 status = complete, move to step 2

[Repeat for steps 2 & 3...]

All steps complete → Finalize
```

---

See [EXECUTION_MODES.md](../docs/EXECUTION_MODES.md) for detailed execution method comparison.
