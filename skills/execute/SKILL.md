---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute

**⚠️ ORCHESTRATOR: Runs in MAIN SESSION**

Execute manages the workflow state machine and coordinates:
- **implementer** → ALWAYS dispatched as subagent (never direct)
- **verifier** → ALWAYS dispatched as subagent (never direct)
- **finalize** → Runs in main session (direct invocation)

## When to Use

Use `workflow:execute` when:
- Workflow has been bootstrapped (PLAN.md exists)
- User wants to start or resume execution
- PLAN.md status is `planning` or `executing`

---

## Flow

1. **Load progress.json** - Find first non-complete step
2. **Ask user** - Confirm start/resume before executing
3. **Main loop** - Execute until all steps complete:
   - If status is `pending` or `needs-fix` → Dispatch implementer (as subagent)
   - If status is `verification` → Dispatch verifier (as subagent)
   - If status is `complete` → Skip to next step
   - If all steps complete → Finalize (direct invocation)

## Implementation Details

### Load State

1. Read `.workflow/{TASK_NAME}/PLAN.md` for mode (1 or 2)
2. Read `.workflow/{TASK_NAME}/progress.json`
3. Find first step with status ≠ "complete"
4. Extract: task name, mode, next step, current status

### Ask User Confirmation

Before dispatching any subagents:

```
AskUserQuestion:
  question: "Start workflow from step {N}?"
  options:
    - "Yes, proceed"
    - "No, stop"
```

**If No:** STOP. **If Yes:** Continue to main loop.

### Main Loop Logic

```bash
# Find next incomplete step
NEXT_STEP=$(jq '.steps | to_entries[] | select(.value.status != "complete") | .key' progress.json | head -1)

if [ -z "$NEXT_STEP" ]; then
  # All steps complete → finalize
else
  STEP_STATUS=$(jq ".steps[\"$NEXT_STEP\"].status" progress.json)
  
  case "$STEP_STATUS" in
    "pending" | "needs-fix")
      # Dispatch implementer as subagent
      ;;
    "verification")
      # Dispatch verifier as subagent
      ;;
  esac
fi
```

### Dispatch Implementer (Subagent)

When a step is `pending` or `needs-fix`, dispatch implementer as subagent:

```python
Agent(
  description: f"Implement {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are implementing step {N} of {TASK_NAME}.

**TASK:**
1. Invoke: Skill(skill: "workflow:implementer")
2. Follow the skill procedure exactly
3. The skill guides you through implementation

**CONTEXT:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: {TASK_DIR_PATH}
- Mode: {MODE} (1=step-by-step, 2=continuous)

**DONE WHEN:**
step-{N}.md shows status: verification
"""
)
```

**After implementer returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter status field
3. If status == `verification` → Continue to verifier
4. If status != `verification` → Ask user for guidance (retry/abort)

### Dispatch Verifier (Subagent)

When a step is in `verification` status, dispatch verifier as subagent:

```python
Agent(
  description: f"Verify {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are verifying step {N} of {TASK_NAME}.

**TASK:**
1. Invoke: Skill(skill: "workflow:verifier")
2. Follow the skill procedure exactly
3. The skill guides you through verification

**CONTEXT:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: {TASK_DIR_PATH}
- Mode: {MODE} (1=step-by-step, 2=continuous)

**DONE WHEN:**
step-{N}.md shows status: complete OR needs-fix
"""
)
```

**After verifier returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter status field
3. If status == `complete`:
   - Mode 1: Ask user for approval before next step
   - Mode 2: Continue to next step automatically
4. If status == `needs-fix`: Loop back to dispatch implementer again
5. If status != `complete` AND != `needs-fix`: Ask user for guidance

### Mode 1: Step Complete - Ask Approval

When step passes verification in **Mode 1 only**:

```
AskUserQuestion:
  question: "Step {N} complete. Continue?"
  options:
    - "Approve and continue"
    - "Request changes (mark needs-fix)"
```

**If Approve:** Continue to next step. **If Request changes:** Mark needs-fix and re-dispatch implementer.

### Finalize (Direct Invocation)

When all steps are complete:

1. Set PLAN.md status = `ready-for-review`
2. Ask user: "All steps verified. Finalize workflow?"

**If Yes:**
```python
Skill(skill: "workflow:finalize")
```

This runs **directly in main session** (not as subagent).

Result:
- PLAN.md status = `complete`
- Final commit created with comprehensive summary
- Workflow complete

---

## Subagent Dispatch Rules

### implementer - ALWAYS Subagent

When dispatching implementer:
- Use `Agent(subagent_type="general-purpose")`
- Never use `Skill(skill: "workflow:implementer")` directly
- Include full step context in prompt
- Subagent updates step status to `verification` when complete
- You verify status changed before proceeding

### verifier - ALWAYS Subagent

When dispatching verifier:
- Use `Agent(subagent_type="general-purpose")`
- Never use `Skill(skill: "workflow:verifier")` directly
- Include full step context in prompt
- Subagent updates step status to `complete` or `needs-fix`
- You verify status changed before proceeding

### finalize - Main Session (Direct)

When all steps complete:
- Use `Skill(skill: "workflow:finalize")` directly
- Runs in main session (NOT as subagent)
- Updates PLAN.md status to `complete`
- Creates final commit

---

## Critical Rules

**MUST DO:**
- ✅ Read PLAN.md and progress.json first
- ✅ Ask user via AskUserQuestion before dispatching first Agent()
- ✅ Use Agent(subagent_type="general-purpose") for implementer and verifier
- ✅ Include task context and step directory in Agent prompt
- ✅ Instruct Agent to `Invoke: Skill(skill: "workflow:...")`
- ✅ Wait for Agent() to return completely
- ✅ Read step-N.md to verify status changed
- ✅ Handle failures gracefully

**NEVER DO:**
- ❌ Skip initial AskUserQuestion
- ❌ Write code or make changes yourself
- ❌ Run tests or build yourself
- ❌ Modify step files directly (agents own this)
- ❌ Dispatch implementer/verifier directly via Skill() (always Agent())
- ❌ Assume success without reading updated files
- ❌ Dispatch multiple agents in parallel (go sequentially)

---

## Mode 2: Continuous Execution

If PLAN.mode == 2:
- NO human approval pauses between steps
- Verifier gates advancement (must complete before next step)
- Loop continuously until all steps complete
- Then ask: "Finalize workflow?"
