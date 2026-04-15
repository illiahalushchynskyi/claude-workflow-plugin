---
name: workflow:execute
description: Orchestrate workflow execution - read state, confirm subagent execution, ask about workflow mode
---

# Workflow Execute

**⚠️ ORCHESTRATOR: Runs in MAIN SESSION**

Execute manages workflow state and coordinates step execution in strict order.

---

## Execution Decision Tree

```
START: /workflow:execute
  │
  ├─→ Step 1: Read State
  │    └─→ File exists? NO  → STOP (tell user to run /workflow:bootstrap)
  │    └─→ File exists? YES → Continue
  │
  ├─→ Step 1.5: Confirm Subagent
  │    └─→ workflow_status == "initialized"? YES → Ask confirmation
  │    │                                     NO  → Skip (resume mode)
  │    └─→ User agrees? YES → Continue to Step 2
  │                     NO  → EXIT (user can resume later)
  │
  ├─→ Step 2: Show What's Next
  │    └─→ Inform user based on current_step.status
  │
  ├─→ Step 3: Ask Mode (ALWAYS - every execution)
  │    └─→ User chooses: mode = 1 (Manual) or 2 (Final)
  │    └─→ If workflow_status == "initialized": set to "in-progress"
  │
  ├─→ Step 4: Check Approval Needed
  │    └─→ (mode == 1 AND status == "complete")? YES → Ask approval
  │    │                                          NO  → Skip
  │    └─→ User approves? YES → status = "complete", continue
  │                      NO  → status = "needs-fix", loop back
  │
  ├─→ Step 5: Execute if Work Needed
  │    └─→ status IN ["pending", "needs-fix", "verification"]? YES → Dispatch agent
  │    │                                                        NO  → status already complete
  │    └─→ After agent returns:
  │        ├─→ If was implementing: status = "verification"
  │        └─→ If was verifying: status = "complete" or "needs-fix"
  │
  └─→ LOOP: Back to Step 1 until all steps "complete"
       └─→ All complete? YES → Finalize → END
                          NO  → Continue loop
```

---

## Step Execution Conditions

| Step | When to Run | When to Skip | Signal to Continue |
|------|-----------|--------------|-------------------|
| **1.5** | workflow_status == "initialized" | Any other status | User confirms OR exits |
| **4** | mode==1 AND status=="complete" | Any other condition | User approves OR requests fix |
| **5** | status IN [pending, needs-fix, verification] | status=="complete" | Agent returns with new status |

---

## Procedure - Exact Order (Non-Negotiable)

### Step 1: Read Current State

**GUARD:** progress.json must exist (if not: STOP and tell user to run `/workflow:bootstrap`)

**PRE-CONDITION:** None (entry point)

**PROCEDURE:**
1. Verify `.workflow/{TASK_NAME}/progress.json` exists
   - ✅ If exists: Continue to file reads
   - ❌ If missing: STOP—user must run `/workflow:bootstrap`
2. Read `.workflow/{TASK_NAME}/PLAN.md` → Extract: title, description, created date
3. Read `.workflow/{TASK_NAME}/progress.json` (SOURCE OF TRUTH)
   - Extract: task_name, mode, workflow_status, current_step, all step statuses, iterations
4. Read `.workflow/{TASK_NAME}/.workflow-config.json` → Extract: projectType, buildCommand, testCommand, migrateCommand
5. Locate current step from progress.json.current_step
6. Locate current step's status from progress.json.steps[current_step].status

**POST-CONDITION:** All 4 files loaded into memory, current_step and its status identified

**OUTPUT TO USER:**
```
Current state:
- Workflow: {TASK_NAME}
- Mode: {Step Manual Approve (mode=1) / Final Approve (mode=2) / Not set}
- Current step: {N} ({STEP_NAME})
- Current status: {pending|implementation|verification|awaiting-approval|needs-fix|complete}
- Workflow status: {initialized|in-progress|paused|completed}
- Iteration: {current iteration count}
```

---

### Step 1.5: Confirm Subagent Execution (First Run Only)

**GUARD:** ONLY execute this step if workflow_status == "initialized"
- ✅ If "initialized": Execute Step 1.5
- ✅ If "in-progress", "paused", or "completed": SKIP to Step 2

**PRE-CONDITION:** workflow_status must be exactly "initialized" (from Step 1 read)

**PROCEDURE:**

If workflow_status == "initialized":
```
AskUserQuestion:
  question: "This workflow will use subagents for implementation and verification. Continue?"
  header: "Subagent Confirmation"
  options:
    - "Agree - Continue with subagent execution"
    - "Stop - Cancel workflow execution"
```

**DECISION:**
- **User selects "Agree"**: Continue to Step 2
- **User selects "Stop"**: Exit /workflow:execute (user can resume later with same command)

If workflow_status != "initialized" (resume mode):
- Skip this step entirely, continue to Step 2

**POST-CONDITION:** 
- If user agreed: proceed to Step 2
- If user stopped: workflow paused, no state changes made
- If resuming: skipped, no state changes made

---

### Step 2: Show What Will Happen Next

**GUARD:** Must have completed Step 1 (all files loaded)

**PRE-CONDITION:** current_step and its status known from Step 1

**PROCEDURE:** Based on current_step.status, determine and tell user what's next:

| Current Status | Next Action | Message to User |
|---|---|---|
| `pending` | Implement | "Will implement step {N}: {STEP_NAME}" |
| `implementation` | Implement | "Continuing implementation of step {N}" |
| `needs-fix` | Re-implement | "Will re-implement step {N} (iteration {I+1})" |
| `verification` | Verify | "Will verify step {N}" |
| `awaiting-approval` | Ask approval | "Step {N} verified. Need your approval to continue." |
| `complete` | Move next | "Step {N} complete. Will move to step {N+1}" |
| All steps `complete` | Finalize | "All steps complete. Will finalize workflow." |

**Also explain mode behavior:**
```
Current Mode: {Step Manual Approve (mode=1) or Final Approve (mode=2) or Not set}

If Step Manual Approve (mode=1):
  → After implementation: Verifier tests → You approve → Next step

If Final Approve (mode=2):
  → After implementation: Verifier tests → Auto-continue (no approval)
```

**POST-CONDITION:** User understands what will happen next

---

### Step 3: Ask About Workflow Mode (Every Execution)

**GUARD:** ALWAYS execute this step—no conditions, every execution
- First run: Set mode for the first time
- Resume: User can change mode if desired

**PRE-CONDITION:** Step 1 and 1.5 completed

**PROCEDURE:** Present mode choice to user:

```
AskUserQuestion:
  question: "How should approval work for this execution?"
  header: "Workflow Approval Mode"
  options:
    - "Step Manual Approve (mode=1) - I approve each step after verification"
    - "Final Approve (mode=2) - Approve only at the end after all steps verified"
```

**Mode Explanation Table:**

| Mode | Behavior | When You're Asked to Approve |
|------|----------|-----|
| **Step Manual Approve (mode=1)** | After each step: implement → verify → PAUSE for your approval → next step | After each step completes verification |
| **Final Approve (mode=2)** | All steps: implement → verify → auto-continue (no pauses) | Once at the very end after all complete |

**ACTIONS AFTER USER CHOOSES:**
1. Update progress.json: `mode` field = 1 or 2 (user's choice)
2. If workflow_status == "initialized": set workflow_status = "in-progress" and started = now (ISO8601)
3. If workflow_status != "initialized": keep as is (resume mode doesn't change status)

**POST-CONDITION:** mode is set in progress.json, user knows what will happen

**Then continue to Step 4.**

---

### Step 4: Check If Step Approval Needed (Step Manual Approve Mode Only)

**GUARD - ONLY EXECUTE IF ALL CONDITIONS TRUE:**
```
(mode == 1) AND (current_step.status == "complete")
```

If either condition false: SKIP this entire step, go to Step 5

| Condition | Value | Action |
|-----------|-------|--------|
| mode == 1? | YES & status=="complete" | Execute Step 4 |
| mode == 1? | NO (mode==2) | SKIP → Step 5 |
| status=="complete"? | NO | SKIP → Step 5 |

**PRE-CONDITION:** Both conditions must be true:
1. mode == 1 (Step Manual Approve was chosen in Step 3)
2. current_step.status == "complete" (verifier just finished and marked complete)

**PROCEDURE (if executing):**

Present approval question:
```
AskUserQuestion:
  question: "Step {N}: {STEP_NAME} passed verification. Approve and continue to next step?"
  header: "Step Approval"
  options:
    - "Approve - Mark complete and move to next step"
    - "Request changes - Back to implementation"
```

**DECISION BASED ON USER RESPONSE:**

**If user selects "Approve":**
1. Record approval in progress.json: approvals.mode_1_manual_approvals += {step: N, approved_at: now, approved_by: "user"}
2. Ensure step status = "complete" (should already be)
3. Clear awaiting_approval_since = null
4. Loop back to Step 1 (to handle next step if more exist, or finalize if all done)

**If user selects "Request changes":**
1. Update progress.json: step status = "needs-fix"
2. Increment step.iteration (prepare for re-implementation)
3. Loop back to Step 1 (will go to Step 5 for re-implementation)

**POST-CONDITION:** 
- If approved: step marked complete, approval recorded, continue to next
- If rejected: step marked needs-fix, ready for re-implementation
- Either way: progress.json updated, loop restarts

---

### Step 5: Execute with Subagent (If Work Needed)

**GUARD - ONLY EXECUTE IF:**
```
current_step.status IN ["pending", "needs-fix", "verification"]
```

If status == "complete": SKIP this step, loop back to Step 1

| Status | Action |
|--------|--------|
| `pending` | Execute (implement) |
| `needs-fix` | Execute (re-implement) |
| `verification` | Execute (verify) |
| `complete` | SKIP (already done) |

**PRE-CONDITION:** 
- progress.json loaded and mode set
- current step identified
- status is NOT "complete"

**PROCEDURE:**

**IF implementing (status = pending or needs-fix):**
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

Completion signal: step-{N}.md Implementation section is filled with:
- List of files modified/created
- Summary of changes
- Test results and migration results (if any)
- Git commit created
"""
)
```

**IF verifying (status = verification):**
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

Completion signal: step-{N}.md Verification section is filled with:
- Build and test results
- Each criterion tested with PASS/FAIL status
- Evidence for each criterion
"""
)
```

**AFTER AGENT RETURNS:**

1. **Check Completion Signal:**
   - If implementing: Read step-{N}.md → Implementation section must be filled
   - If verifying: Read step-{N}.md → Verification section must be filled with criterion results

2. **Update progress.json Step Status:**
   ```
   If was implementing (old status pending/needs-fix):
     → new status = "verification"
   
   If was verifying (old status verification):
     → If all criteria PASSED: new status = "complete"
     → If any criterion FAILED: new status = "needs-fix"
   ```

3. **If Step Manual Approve Mode AND now complete:**
   - Set awaiting_approval_since = now (ISO8601)
   - This marks workflow as "paused" waiting for user approval

4. **Loop back to Step 1** to process new status
   - Step 1: Read updated state
   - Step 1.5: Skip (already in-progress)
   - Step 2: Show what's next based on new status
   - Step 3: Ask mode again
   - Step 4: If mode==1 AND status==complete, ask approval
   - Step 5: If not complete, continue or finalize

**POST-CONDITION:** 
- step-{N}.md has Implementation or Verification section filled
- progress.json updated with new status
- Ready to loop back to Step 1

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
