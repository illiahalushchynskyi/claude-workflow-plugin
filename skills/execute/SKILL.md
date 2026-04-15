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

**For detailed patterns and common mistakes, see:**
- **[EXECUTION_MODES.md](../docs/EXECUTION_MODES.md)** — Choose between Subagent Mode and Manual Mode
- **[EXECUTE_AGENT_DISPATCH.md](../docs/EXECUTE_AGENT_DISPATCH.md)** — Agent vs Skill dispatch patterns

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
3. **Ask user** - Choose execution mode:
   - **Subagent Mode:** Automatic Agent dispatch (recommended, no manual work)
   - **Manual Mode:** Guide user to invoke skills directly (more control)
4. **Main loop** - Execute until all steps complete (behavior depends on mode):
   - **Subagent Mode:**
     - If status is `pending` or `needs-fix` → Dispatch implementer Agent subagent
     - If status is `verification` → Dispatch verifier Agent subagent
     - If status is `awaiting-approval` → Ask user for approval (Mode 1 only)
     - If status is `complete` → Skip to next step
     - If all steps complete → Finalize
   - **Manual Mode:**
     - If status is `pending` or `needs-fix` → Guide user: "Run `/workflow:implementer` to implement step"
     - If status is `verification` → Guide user: "Run `/workflow:verifier` to verify step"
     - If status is `awaiting-approval` → Ask user for approval (Mode 1 only)
     - If status is `complete` → Skip to next step
     - If all steps complete → Guide user: "Run `/workflow:finalize` to complete"
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

### Step 3a: Ask Execution Mode

First, ask user about execution mode:

```
AskUserQuestion:
  question: "How should implementer and verifier work?"
  header: "Execution Mode"
  options:
    - label: "Subagent Mode (Recommended)"
      description: "I dispatch implementer/verifier as isolated subagents via Agent(). You handle orchestration only."
    - label: "Manual Mode"
      description: "I guide you to run workflow:implementer and workflow:verifier directly. You invoke each skill manually."
```

**If Subagent Mode:** Continue to step 3b (auto-dispatch)
**If Manual Mode:** Continue to step 3c (guided manual execution)

### Step 3b: Ask Workflow Start Confirmation (Subagent Mode)

If user chose Subagent Mode:

```
AskUserQuestion:
  question: "Start workflow from step {N} (Mode {MODE}, Subagent auto-dispatch)?"
  options:
    - "Yes, dispatch subagents"
    - "No, stop"
```

**If No:** STOP. **If Yes:** Continue to main loop (subagent dispatch).

### Step 3c: Ask Workflow Start Confirmation (Manual Mode)

If user chose Manual Mode:

```
AskUserQuestion:
  question: "Start workflow from step {N} (Mode {MODE}, manual invocation)?"
  options:
    - "Yes, I'll run skills manually"
    - "No, stop"
```

**If No:** STOP. **If Yes:** Continue to main loop (guided manual execution).

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

### Manual Mode Execution - GUIDED STEP INVOCATION

When user chooses Manual Mode, you guide them through manual invocation instead of dispatching agents.

**When step status is `pending` or `needs-fix`:**

Ask user:
```
AskUserQuestion:
  question: "Ready to implement step {N}: {STEP_NAME}?"
  options:
    - "Yes, I'll run /workflow:implementer"
    - "No, pause here"
```

**If user chooses to implement:**
1. Tell user: "Now run: `/workflow:implementer`"
2. User will:
   - Load skill and follow procedure
   - Invoke the skill themselves
   - Update step file with status = `verification`
3. You wait for user confirmation: "Have you run `/workflow:implementer`?"
4. When confirmed, read step-{N}.md to check status
5. If status == `verification`: Continue to next section
6. If status != `verification`: Ask user for guidance

**When step status is `verification`:**

Ask user:
```
AskUserQuestion:
  question: "Ready to verify step {N}: {STEP_NAME}?"
  options:
    - "Yes, I'll run /workflow:verifier"
    - "No, pause here"
```

**If user chooses to verify:**
1. Tell user: "Now run: `/workflow:verifier`"
2. User will:
   - Load skill and follow procedure
   - Invoke the skill themselves
   - Update step file with status = `complete` or `needs-fix`
3. You wait for user confirmation: "Have you run `/workflow:verifier`?"
4. When confirmed, read step-{N}.md to check status
5. If status == `complete`: Continue to next step (or approval if Mode 1)
6. If status == `needs-fix`: Loop back to implementation
7. If status != `complete` AND != `needs-fix`: Ask user for guidance

**Manual Mode Summary:**
- You guide step-by-step
- User invokes skills directly
- You read files to verify completion
- User has full control and visibility
- Slower than Subagent Mode but more transparent

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

## Execution Rules - Two Modes

### Mode A: Subagent Mode (Recommended)

You dispatch agents automatically. Implementer and Verifier run as isolated subagents.

#### implementer - ALWAYS Subagent (NEVER Direct Skill)

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

### Mode B: Manual Mode

You guide user step-by-step. User invokes skills directly, you coordinate and verify.

#### Manual Implementer Invocation

When step status is `pending` or `needs-fix`:

**You ask user:**
```
"Ready to implement step N? Run: /workflow:implementer"
```

**User does:**
1. Runs `/workflow:implementer`
2. Skill guides them through implementation
3. They update step file with status = `verification`
4. They confirm when done

**You do:**
1. Read step-{N}.md to verify status changed to `verification`
2. If status == `verification`: Continue to verifier step
3. If status != `verification`: Ask user for guidance

#### Manual Verifier Invocation

When step status is `verification`:

**You ask user:**
```
"Ready to verify step N? Run: /workflow:verifier"
```

**User does:**
1. Runs `/workflow:verifier`
2. Skill guides them through verification
3. They update step file with status = `complete` or `needs-fix`
4. They confirm when done

**You do:**
1. Read step-{N}.md to verify status changed
2. If status == `complete`: Continue to next step (or approval if Mode 1)
3. If status == `needs-fix`: Go back to implementer step
4. If status != `complete` AND != `needs-fix`: Ask user for guidance

#### Manual Finalize Invocation

When all steps are complete:

**You ask user:**
```
"All steps done. Ready to finalize? Run: /workflow:finalize"
```

**User does:**
1. Runs `/workflow:finalize`
2. Skill creates final commit
3. Updates PLAN.md to `complete`
4. They confirm when done

**You do:**
1. Verify PLAN.md status is `complete`
2. Confirm workflow is finished

#### Manual Mode Rules

**CORRECT pattern:**
```
You → "Run /workflow:implementer"
User → Runs it
You → Read step file to verify status
```

**Advantages:**
- ✅ User has full visibility and control
- ✅ User invokes skills directly
- ✅ Clear step-by-step guidance
- ✅ More transparent process

**Disadvantages:**
- ❌ More manual work for user
- ❌ Slower execution
- ❌ More back-and-forth

---

## Critical Rules - YOU ARE THE ORCHESTRATOR, NOT THE WORKER

**YOUR ROLE (in BOTH modes):**
- Read and manage state files (PLAN.md, progress.json)
- Coordinate workflow execution (Subagent Mode or Manual Mode)
- Ask users for approval and guidance
- Verify step completion by reading files
- Handle transitions between steps

**SUBAGENT MODE - Agent Dispatch:**
- You dispatch implementer Agent (it does the work)
- You dispatch verifier Agent (it does the work)
- You wait for agents to complete
- You read files to verify status changed

**MANUAL MODE - User Guidance:**
- You guide user to run `/workflow:implementer`
- You guide user to run `/workflow:verifier`
- You wait for user confirmation
- You read files to verify status changed

**NEITHER MODE:**
- ❌ You do NOT write code yourself
- ❌ You do NOT run tests yourself
- ❌ You do NOT call Skill() directly for implementer/verifier (Subagent Mode only)

---

**MUST DO (Both Modes):**
- ✅ Read PLAN.md, progress.json, and .workflow-config.json first (YOU do this)
- ✅ Set workflow_status and timestamps in progress.json (YOU do this)
- ✅ Extract projectType, buildCommand, testCommand, migrateCommand from config (YOU do this)
- ✅ Ask user via AskUserQuestion: "Subagent Mode or Manual Mode?"
- ✅ Handle awaiting-approval status correctly (Mode 1 vs Mode 2)
- ✅ Set paused status when awaiting approval, resume when approval given
- ✅ Read step-N.md to verify completion status changed
- ✅ Handle failures gracefully

**MUST DO (Subagent Mode only):**
- ✅ Use Agent(subagent_type="general-purpose") for implementer and verifier
- ✅ Include `Invoke: Skill(skill: "workflow:implementer")` in implementer Agent prompt
- ✅ Include `Invoke: Skill(skill: "workflow:verifier")` in verifier Agent prompt
- ✅ Wait for Agent() to return completely
- ✅ Read step-N.md to verify subagent updated status field

**MUST DO (Manual Mode only):**
- ✅ Ask user: "Ready to run `/workflow:implementer`?"
- ✅ Ask user: "Ready to run `/workflow:verifier`?"
- ✅ Wait for user confirmation
- ✅ Read step-N.md to verify user updated status field

**NEVER DO (Both Modes):**
- ❌ Skip initial mode selection question
- ❌ Write code or make changes yourself (implementer's job)
- ❌ Run tests or build yourself (verifier's job)
- ❌ Modify step files directly (agents/users update them)
- ❌ Assume success without reading updated step files
- ❌ Dispatch multiple agents in parallel (go sequentially)

**NEVER DO (Subagent Mode only):**
- ❌ Call `Skill(skill: "workflow:implementer")` directly—always dispatch via Agent()
- ❌ Call `Skill(skill: "workflow:verifier")` directly—always dispatch via Agent()
- ❌ Include full skill procedure inline in prompt (use `Invoke: Skill()` instead)

---

## Mode 2: Continuous Execution

If PLAN.mode == 2:
- NO human approval pauses between steps
- Verifier gates advancement (must complete before next step)
- Loop continuously until all steps complete
- Then ask: "Finalize workflow?"
