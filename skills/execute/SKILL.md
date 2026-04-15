---
name: workflow:execute
description: Orchestrate workflow execution - choose Subagent Mode (automated) or Manual Mode (transparent)
---

# Workflow Execute

**⚠️ ORCHESTRATOR: Runs in MAIN SESSION**

You manage workflow state and coordinate step execution. Choose your execution mode:

## Execution Modes

| Mode | Implementer/Verifier | Best for |
|------|---------------------|----------|
| **Subagent** (✅ Recommended) | Dispatched via `Agent()` as isolated subagents | Speed, automation, hands-off |
| **Manual** | Invoked via `Skill()` in this session | Debugging, transparency, control |

**The ONLY difference:** How implementer and verifier are invoked (Agent vs Skill). Everything else is the same.

---

## When to Use

Use `/workflow:execute` when:
- ✅ Workflow has been bootstrapped (PLAN.md exists)
- ✅ progress.json exists (created by bootstrap)
- ✅ All step-N.md files exist
- ✅ User wants to start or resume execution
- ✅ PLAN.md status is `pending` or `in-progress`
- ✅ Resuming from pause (e.g., after Mode 1 approval)

**REQUIRED:** If progress.json doesn't exist, STOP and tell user to run `/workflow:bootstrap`

---

## Procedure

### Step 0: Validate progress.json Exists

Before doing anything, check if progress.json exists:

```bash
if [ ! -f ".workflow/{TASK_NAME}/progress.json" ]; then
  # progress.json missing - STOP, cannot proceed
  echo "❌ ERROR: progress.json not found"
  echo ".workflow/{TASK_NAME}/progress.json is required"
  echo "Ensure workflow was created with: /workflow:bootstrap"
  exit 1
fi
```

**If progress.json is missing:**
- ❌ STOP execution
- Tell user: "Workflow not properly initialized. Run `/workflow:bootstrap` first."
- Do NOT create progress.json yourself
- Do NOT continue

**If progress.json exists:** Continue to Step 1.

### Step 1: Load State

1. Read `.workflow/{TASK_NAME}/PLAN.md` (mode, status)
2. Read `.workflow/{TASK_NAME}/progress.json` (step statuses) ← Now guaranteed to exist
3. Read `.workflow/{TASK_NAME}/.workflow-config.json` (projectType, buildCommand, testCommand, migrateCommand)
4. Find first step with status ≠ "complete"

### Step 2: Update Workflow Status

If first execution (workflow_status == "initialized"):
1. Set progress.json: `workflow_status` = `in-progress`
2. Set progress.json: `started` = ISO 8601 timestamp
3. Set PLAN.md: `status` = `in-progress`

If resuming (workflow_status == "paused"):
1. Keep status as `in-progress`
2. Keep `started` timestamp unchanged

### Step 3: Ask User to Choose Mode

```
AskUserQuestion:
  question: "How should I run implementer and verifier?"
  header: "Execution Mode"
  options:
    - "Subagent Mode (Recommended) - I dispatch as Agent() subagents"
    - "Manual Mode - I run as Skill() in this session"
```

Store the choice and continue.

### Step 4: Main Loop

**Repeat until all steps complete:**

**If step status is `pending` or `needs-fix`:**

Subagent Mode:
```python
Agent(
  description: f"Implement {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are implementing step {N} of {TASK_NAME}.

Read: .workflow/{TASK_NAME}/steps/step-{N}.md
Invoke: Skill(skill: "workflow:implementer")
Follow the skill procedure.

Config:
- projectType: {PROJECT_TYPE}
- buildCommand: {BUILD_COMMAND}
- testCommand: {TEST_COMMAND}
- migrateCommand: {MIGRATE_COMMAND}
"""
)
```

Manual Mode:
```python
Skill(skill: "workflow:implementer")
```

Then:
1. Read `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
3. If `verification` → Continue loop
4. If not → Ask user for guidance

---

**If step status is `verification`:**

Subagent Mode:
```python
Agent(
  description: f"Verify {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are verifying step {N} of {TASK_NAME}.

Read: .workflow/{TASK_NAME}/steps/step-{N}.md
Invoke: Skill(skill: "workflow:verifier")
Follow the skill procedure.

Config:
- projectType: {PROJECT_TYPE}
- buildCommand: {BUILD_COMMAND}
- testCommand: {TEST_COMMAND}
- migrateCommand: {MIGRATE_COMMAND}
"""
)
```

Manual Mode:
```python
Skill(skill: "workflow:verifier")
```

Then:
1. Read `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
3. If `complete`:
   - Mode 1: Update progress.json `awaiting-approval`, ask user for approval
   - Mode 2: Continue to next step
4. If `needs-fix` → Loop back to implementer
5. If other → Ask user for guidance

---

**If step status is `awaiting-approval` (Mode 1 only):**

```
AskUserQuestion:
  question: "Step {N} passed verification. Approve and continue?"
  options:
    - "Approve and continue"
    - "Request changes"
```

If approve:
- Update progress.json: step `status` = `complete`
- Continue loop

If request changes:
- Update progress.json: step `status` = `needs-fix`
- Loop back to implementer

---

**If step status is `complete`:**

Move to next step. Repeat loop.

---

**If all steps complete:**

```python
Skill(skill: "workflow:finalize")
```

Then:
- Update progress.json: `workflow_status` = `completed`
- Update PLAN.md: `status` = `completed`
- Exit

---

## Critical Rules

**BEFORE ANYTHING ELSE:**
- ✅ Check Step 0: **progress.json MUST exist**
- ✅ If missing → STOP and tell user to run `/workflow:bootstrap`
- ❌ Do NOT create progress.json yourself
- ❌ Do NOT try to proceed without progress.json

**YOU ARE THE ORCHESTRATOR:**
- ✅ Read state files (PLAN.md, progress.json, .workflow-config.json)
- ✅ Coordinate step execution (Subagent or Manual mode)
- ✅ Ask users for approval
- ✅ Verify step completion by reading files

**IMPLEMENTER & VERIFIER (in BOTH modes):**
- ✅ Implement code / Verify implementation (skill guides them)
- ✅ Update step-{N}.md with status
- ❌ Not your job (implement/verify yourself)

**MODE CHOICE MATTERS:**
- Subagent: `Agent(implementer)` then `Agent(verifier)`
- Manual: `Skill(workflow:implementer)` then `Skill(workflow:verifier)`

**NEVER:**
- ❌ Skip progress.json validation (Step 0)
- ❌ Create progress.json if missing
- ❌ Skip mode selection question
- ❌ Write code yourself
- ❌ Run tests yourself
- ❌ Assume success without reading step files after execution

---

## Example: Subagent Mode (3 steps, Mode 1)

```
/workflow:execute
↓
Step 0: Check progress.json exists? ✓ Yes
↓
Load state: Step 1 pending, Mode 1
↓
Ask: "Subagent Mode or Manual Mode?"
User chooses: Subagent Mode
↓
Agent(implementer step 1)
↓
Read step-1.md: status = verification ✓
↓
Agent(verifier step 1)
↓
Read step-1.md: status = complete ✓
↓
Ask: "Approve step 1?"
User approves
↓
Step 2 pending
↓
[Repeat for step 2...]
↓
[Repeat for step 3...]
↓
All steps complete
↓
Skill(finalize)
↓
Done
```

---

## Example: Manual Mode (3 steps, Mode 1)

```
/workflow:execute
↓
Step 0: Check progress.json exists? ✓ Yes
↓
Load state: Step 1 pending, Mode 1
↓
Ask: "Subagent Mode or Manual Mode?"
User chooses: Manual Mode
↓
Skill(workflow:implementer) [I run it here]
[Step 1 implementation happens...]
↓
Read step-1.md: status = verification ✓
↓
Skill(workflow:verifier) [I run it here]
[Step 1 verification happens...]
↓
Read step-1.md: status = complete ✓
↓
Ask: "Approve step 1?"
User approves
↓
[Repeat for steps 2 & 3...]
↓
All steps complete
↓
Skill(finalize) [I run it here]
↓
Done
```

---

## Example: Error - progress.json Missing

```
/workflow:execute
↓
Step 0: Check progress.json exists? ❌ NO
↓
❌ ERROR: progress.json not found
.workflow/{TASK_NAME}/progress.json is required

Ensure workflow was created with: /workflow:bootstrap

[STOP - Do NOT continue]
```

**User sees:**
```
Workflow not properly initialized.

Please run: /workflow:bootstrap

This will create:
- PLAN.md
- progress.json
- step-1.md, step-2.md, etc.
- .workflow-config.json
```

---

See [EXECUTION_MODES.md](../docs/EXECUTION_MODES.md) for detailed mode comparison and when to choose each.
