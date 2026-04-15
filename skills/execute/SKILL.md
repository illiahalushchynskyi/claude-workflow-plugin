---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute

**⚠️ ORCHESTRATOR: Runs in MAIN SESSION**

## CRITICAL: Agent vs Skill

When you see `implementer` or `verifier` mentioned here:
- ❌ These are NOT skills to call via `Skill(skill: "workflow:implementer")`
- ✅ These are SUBAGENTS to dispatch via `Agent(subagent_type="general-purpose")`
- ✅ The Agent prompt must contain `Invoke: Skill(skill: "workflow:implementer")` for the subagent to execute

**You are the orchestrator. You dispatch agents. You do NOT run implementer or verifier yourself.**

**See [EXECUTE_AGENT_DISPATCH.md](../docs/EXECUTE_AGENT_DISPATCH.md) for detailed patterns and common mistakes.**

---

Execute manages the workflow state machine and coordinates:
- **implementer** → ALWAYS dispatched as Agent subagent (NEVER direct Skill)
- **verifier** → ALWAYS dispatched as Agent subagent (NEVER direct Skill)
- **finalize** → Runs in main session (direct Skill invocation)

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

### Dispatch Implementer (Subagent) - CRITICAL RULE

**You are the orchestrator. You do NOT write code. You dispatch the implementer subagent.**

When a step is `pending` or `needs-fix`, use Agent() tool to dispatch implementer:

```python
Agent(
  description: "Implement {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are a subagent implementing step {N} of {TASK_NAME}.

**YOUR TASK:**
1. Load and read: .workflow/{TASK_NAME}/steps/step-{N}.md
2. Invoke: Skill(skill: "workflow:implementer")
3. Follow the skill procedure exactly
4. The skill teaches you how to implement this step

**CONTEXT FOR YOUR WORK:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: .workflow/{TASK_NAME}/
- Mode: {MODE} (1=step-by-step, 2=continuous)
- Project Type: {PROJECT_TYPE}
- Build command: {BUILD_COMMAND}
- Test command: {TEST_COMMAND}
- Migrate command: {MIGRATE_COMMAND}

**SUCCESS CRITERIA:**
- step-{N}.md status is set to 'verification'
- Code changes committed
- All tests pass
- Implementation section filled with notes
"""
)
```

**Key points:**
- You call Agent() with the prompt above
- The subagent reads workflow:implementer skill and follows it
- You wait for Agent() to complete
- The subagent updates the step file

**After implementer subagent returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
3. If status == `verification` → Continue to verifier dispatch
4. If status != `verification` → Ask user for guidance (retry/abort)

### Dispatch Verifier (Subagent) - CRITICAL RULE

**You are the orchestrator. You do NOT test code. You dispatch the verifier subagent.**

When a step is in `verification` status, use Agent() tool to dispatch verifier:

```python
Agent(
  description: "Verify {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are a subagent verifying step {N} of {TASK_NAME}.

**YOUR TASK:**
1. Load and read: .workflow/{TASK_NAME}/steps/step-{N}.md
2. Invoke: Skill(skill: "workflow:verifier")
3. Follow the skill procedure exactly
4. The skill teaches you how to verify this step

**CONTEXT FOR YOUR WORK:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: .workflow/{TASK_NAME}/
- Mode: {MODE} (1=step-by-step, 2=continuous)
- Project Type: {PROJECT_TYPE}
- Build command: {BUILD_COMMAND}
- Test command: {TEST_COMMAND}
- Migrate command: {MIGRATE_COMMAND}

**SUCCESS CRITERIA:**
- step-{N}.md status is set to 'complete' (all tests pass, criteria met)
- OR status is set to 'needs-fix' (issues documented)
- Verification section filled with test results and evidence
"""
)
```

**Key points:**
- You call Agent() with the prompt above
- The subagent reads workflow:verifier skill and follows it
- You wait for Agent() to complete
- The subagent updates the step file with pass/fail status

**After verifier subagent returns:**
1. Read: `.workflow/{TASK_NAME}/steps/step-{N}.md`
2. Check frontmatter `status:` field
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

## Subagent Dispatch Rules - STRICT ENFORCEMENT

### implementer - ALWAYS Subagent (NEVER Direct Skill)

**CORRECT:**
```python
Agent(
  description: "Implement step X",
  subagent_type: "general-purpose",
  prompt: "...\nInvoke: Skill(skill: 'workflow:implementer')\n..."
)
```

**WRONG - DO NOT DO THIS:**
```python
❌ Skill(skill: "workflow:implementer")  # FORBIDDEN - Direct skill call
❌ # You writing code yourself  # FORBIDDEN - Orchestrator never codes
```

**Rules:**
- ✅ Use `Agent(subagent_type="general-purpose")` to dispatch
- ✅ Include `Invoke: Skill(skill: "workflow:implementer")` in the prompt
- ✅ Include project config (projectType, buildCommand, testCommand, migrateCommand)
- ✅ Subagent reads skill, implements, updates step status to `verification`
- ✅ You wait for Agent() to return, then check step-{N}.md status

### verifier - ALWAYS Subagent (NEVER Direct Skill)

**CORRECT:**
```python
Agent(
  description: "Verify step X",
  subagent_type: "general-purpose",
  prompt: "...\nInvoke: Skill(skill: 'workflow:verifier')\n..."
)
```

**WRONG - DO NOT DO THIS:**
```python
❌ Skill(skill: "workflow:verifier")  # FORBIDDEN - Direct skill call
❌ # You running tests yourself  # FORBIDDEN - Orchestrator never verifies
```

**Rules:**
- ✅ Use `Agent(subagent_type="general-purpose")` to dispatch
- ✅ Include `Invoke: Skill(skill: "workflow:verifier")` in the prompt
- ✅ Include project config (projectType, buildCommand, testCommand, migrateCommand)
- ✅ Subagent reads skill, tests, updates step status to `complete` or `needs-fix`
- ✅ You wait for Agent() to return, then check step-{N}.md status

### finalize - Main Session (Direct Skill ONLY)

**CORRECT:**
```python
Skill(skill: "workflow:finalize")  # OK - Main session direct
```

**WRONG:**
```python
❌ Agent(...prompt="...workflow:finalize...")  # FORBIDDEN - Not a subagent task
```

**Rules:**
- ✅ Use `Skill(skill: "workflow:finalize")` directly (you orchestrate this)
- ✅ Runs in main session only (not as subagent)
- ✅ Updates PLAN.md status to `complete`
- ✅ Creates final commit

---

## Critical Rules - YOU ARE THE ORCHESTRATOR, NOT THE WORKER

**YOUR ROLE:**
- You manage state files (PLAN.md, progress.json)
- You dispatch agents (implementer, verifier)
- You ask users for approval
- You orchestrate the workflow

**AGENTS' ROLE:**
- implementer agent: writes code, runs tests, commits, updates step file
- verifier agent: builds project, runs tests, verifies criteria, updates step file
- You do NONE of this work yourself

---

**MUST DO:**
- ✅ Read PLAN.md, progress.json, and .workflow-config.json first (YOU do this)
- ✅ Set workflow_status and timestamps in progress.json (YOU do this)
- ✅ Extract projectType, buildCommand, testCommand, migrateCommand from config (YOU do this)
- ✅ Ask user via AskUserQuestion before dispatching first Agent()
- ✅ Use Agent(subagent_type="general-purpose") for implementer and verifier
- ✅ Include `Invoke: Skill(skill: "workflow:implementer")` in implementer Agent prompt
- ✅ Include `Invoke: Skill(skill: "workflow:verifier")` in verifier Agent prompt
- ✅ Wait for Agent() to return completely
- ✅ Read step-N.md to verify subagent updated status field
- ✅ Handle awaiting-approval status correctly (Mode 1 vs Mode 2)
- ✅ Set paused status when awaiting approval, resume when approval given
- ✅ Handle failures gracefully

**NEVER DO:**
- ❌ Skip initial AskUserQuestion
- ❌ Write code or make changes yourself (that's implementer's job)
- ❌ Run tests or build yourself (that's verifier's job)
- ❌ Modify step files directly (agents update them via skills)
- ❌ Call `Skill(skill: "workflow:implementer")` directly—always dispatch via Agent()
- ❌ Call `Skill(skill: "workflow:verifier")` directly—always dispatch via Agent()
- ❌ Assume success without reading updated step files
- ❌ Dispatch multiple agents in parallel (go sequentially)
- ❌ Include full skill procedure inline in prompt (use `Invoke: Skill()` instead)

---

## Mode 2: Continuous Execution

If PLAN.mode == 2:
- NO human approval pauses between steps
- Verifier gates advancement (must complete before next step)
- Loop continuously until all steps complete
- Then ask: "Finalize workflow?"
