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
- PLAN.md status is `pending` or `in-progress`
- Resuming from pause (e.g., after human approval in Mode 1)

---

## Flow

1. **Load progress.json** - Find first non-complete step
2. **Update workflow status** - Set workflow_status to `in-progress` and set `started` timestamp if first run
3. **Ask user** - Confirm start/resume before executing
4. **Main loop** - Execute until all steps complete:
   - If status is `pending` or `needs-fix` → Dispatch implementer (as subagent)
   - If status is `implementation` → Dispatch implementer (subagent still working)
   - If status is `verification` → Dispatch verifier (as subagent)
   - If status is `awaiting-approval` → Ask user for approval (Mode 1 only)
   - If status is `complete` → Skip to next step
   - If all steps complete → Finalize (direct invocation)
5. **On completion** - Set workflow_status to `completed` and `completed` timestamp
6. **On pause** - Set workflow_status to `paused` (when awaiting approval)

## Implementation Details

### Load State

1. Read `.workflow/{TASK_NAME}/PLAN.md` for mode (1 or 2)
2. Read `.workflow/{TASK_NAME}/progress.json`
3. Read `.workflow/{TASK_NAME}/.workflow-config.json` - Extract: projectType, buildCommand, testCommand, migrateCommand
4. Find first step with status ≠ "complete"
5. Extract: task name, mode, next step, current status, and config info

### Update Workflow Status

If starting first execution (workflow_status == "initialized"):
1. Update progress.json: `workflow_status` = `in-progress`
2. Update progress.json: `started` = ISO 8601 current timestamp
3. Update PLAN.md: `status` = `in-progress`

If resuming after pause (workflow_status == "paused"):
1. Keep workflow_status as `in-progress` (execution continues)
2. Keep `started` timestamp unchanged
3. Keep PLAN.md status as `in-progress`

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

1. First, load `.workflow-config.json` to get projectType, buildCommand, testCommand, migrateCommand
2. Include this info in the subagent prompt

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
- Project Type: {PROJECT_TYPE}
- Build command: {BUILD_COMMAND}
- Test command: {TEST_COMMAND}
- Migrate command: {MIGRATE_COMMAND}

**DONE WHEN:**
step-{N}.md shows status: verification
"""
)
```

The skill will load .workflow-config.json and use these commands for building, testing, and migrations.

**After implementer returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter status field
3. If status == `verification` → Continue to verifier
4. If status != `verification` → Ask user for guidance (retry/abort)

### Dispatch Verifier (Subagent)

When a step is in `verification` status, dispatch verifier as subagent:

1. First, load `.workflow-config.json` to get projectType, buildCommand, testCommand, migrateCommand
2. Include this info in the subagent prompt

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
- Project Type: {PROJECT_TYPE}
- Build command: {BUILD_COMMAND}
- Test command: {TEST_COMMAND}
- Migrate command: {MIGRATE_COMMAND}

**DONE WHEN:**
step-{N}.md shows status: complete OR needs-fix
"""
)
```

The skill will load .workflow-config.json and use these commands for building, testing, and migrations.

**After verifier returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter status field
3. If status == `complete`:
   - Mode 1: Update progress.json step status to `awaiting-approval`, set `awaiting_approval_since` timestamp
   - Mode 2: Automatically move to `complete` (no user approval needed)
4. If status == `needs-fix`: Loop back to dispatch implementer again
5. If status != `complete` AND != `needs-fix`: Ask user for guidance

### Mode 1: Awaiting Approval

When a step is in `awaiting-approval` status:

1. Check if step status is `awaiting-approval` in progress.json
2. Set workflow_status to `paused`
3. Ask user for approval:

```
AskUserQuestion:
  question: "Step {N} verification passed. Approve and continue?"
  options:
    - "Approve and continue to next step"
    - "Request changes (back to implementation)"
```

**If Approve:**
1. Update progress.json: step status = `complete`, set `approval_date` timestamp
2. Add entry to `approvals.mode_1_manual_approvals` array
3. Continue to next step

**If Request changes:**
1. Update progress.json: step status = `needs-fix`
2. Loop back to dispatch implementer again

### Mode 2: Auto-Advance

When step is verified as complete in Mode 2:
1. Automatically set step status = `complete`
2. No user approval needed
3. Continue to next step immediately

### Finalize (Direct Invocation)

When all steps are complete:

1. Update progress.json: `workflow_status` = `completed`, set `completed` timestamp
2. Update PLAN.md: `status` = `completed`
3. Ask user: "All steps verified. Finalize workflow (create commit)?"

**If Yes:**
```python
Skill(skill: "workflow:finalize")
```

This runs **directly in main session** (not as subagent).

Result:
- Final commit created with comprehensive summary
- Workflow marked complete in both progress.json and PLAN.md
- Ready for review/merge

---

## Subagent Dispatch Rules

### implementer - ALWAYS Subagent

When dispatching implementer:
- Use `Agent(subagent_type="general-purpose")`
- Never use `Skill(skill: "workflow:implementer")` directly
- Include full step context in prompt
- Include projectType, buildCommand, testCommand, migrateCommand from .workflow-config.json
- Subagent updates step status to `verification` when complete
- You verify status changed before proceeding

### verifier - ALWAYS Subagent

When dispatching verifier:
- Use `Agent(subagent_type="general-purpose")`
- Never use `Skill(skill: "workflow:verifier")` directly
- Include full step context in prompt
- Include projectType, buildCommand, testCommand, migrateCommand from .workflow-config.json
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
- ✅ Read PLAN.md, progress.json, and .workflow-config.json first
- ✅ Set workflow_status and timestamps in progress.json properly
- ✅ Extract projectType, buildCommand, testCommand, migrateCommand from config
- ✅ Ask user via AskUserQuestion before dispatching first Agent()
- ✅ Use Agent(subagent_type="general-purpose") for implementer and verifier
- ✅ Include task context, step directory, AND project config info in Agent prompt
- ✅ Instruct Agent to `Invoke: Skill(skill: "workflow:...")`
- ✅ Wait for Agent() to return completely
- ✅ Read step-N.md to verify status changed
- ✅ Handle awaiting-approval status correctly (Mode 1 vs Mode 2)
- ✅ Set paused status when awaiting approval, resume when approval given
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
