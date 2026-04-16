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
  │    └─→ (mode == 1 AND status == "awaiting-approval")? YES → Ask approval
  │    │                                                  NO  → Skip
  │    └─→ User approves? YES → status = "complete", continue
  │                      NO  → status = "needs-fix", loop back (re-enter Step 5)
  │
  ├─→ Step 5: Execute if Work Needed
  │    └─→ status IN [pending, needs-fix, verification]? YES → Execute
  │    │                                                 NO  → Skip (complete or awaiting-approval)
  │    └─→ If status IN [pending, needs-fix]: AUTO-RETRY LOOP (both Mode 1 & 2)
  │        ├─→ Implement → Verify
  │        ├─→ If PASS: Mode 1 → awaiting-approval; Mode 2 → complete
  │        └─→ If FAIL: Re-implement → Verify (repeat until PASS, no limit)
  │    └─→ If status == verification: Verify only (implement already done)
  │        ├─→ If PASS: Mode 1 → awaiting-approval; Mode 2 → complete
  │        └─→ If FAIL: Re-implement → Verify (enter auto-retry loop)
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
| **4** | mode==1 AND status=="awaiting-approval" | Any other condition | User approves OR requests fix (re-enter Step 5 loop) |
| **5** | status IN [pending, needs-fix, verification] | status=="complete" or status=="awaiting-approval" | In Mode 1: auto-retry loop completes with status=awaiting-approval; In Mode 2 or verification: agent returns with new status |

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
| `awaiting-approval` | Ask approval (Mode 1 only) | "Step {N} verified (iteration {I}). Need your approval to continue." |
| `complete` | Move next | "Step {N} complete. Will move to step {N+1}" |
| All steps `complete` | Finalize | "All steps complete. Will finalize workflow." |

**Also explain mode behavior:**
```
Current Mode: {Step Manual Approve (mode=1) or Final Approve (mode=2) or Not set}

If Step Manual Approve (mode=1):
  → Implement → Verify
  → If verification FAILS: Re-implement → Verify again (automatic, no user action)
  → If verification PASSES: You approve → Next step
  → Repeats until verification passes, then pauses for approval

If Final Approve (mode=2):
  → Implement → Verify
  → If verification FAILS: Re-implement → Verify again (automatic, no pauses, no approval)
  → If verification PASSES: Auto-continue to next step
  → All steps run automatically, approval only at the very end for entire workflow
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
| **Step Manual Approve (mode=1)** | Each step: implement → verify → (if fail: re-implement → verify auto-retry until pass) → PAUSE for your approval → next step | After each step passes verification (once per step) |
| **Final Approve (mode=2)** | All steps: implement → verify → (if fail: auto-retry until pass) → auto-continue (no pauses) → next step → ... | Once at the very end after all steps complete |

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
(mode == 1) AND (current_step.status == "awaiting-approval")
```

If either condition false: SKIP this entire step, go to Step 5

| Condition | Value | Action |
|-----------|-------|--------|
| mode == 1? | YES & status=="awaiting-approval" | Execute Step 4 |
| mode == 1? | NO (mode==2) | SKIP → Step 5 |
| status=="awaiting-approval"? | NO | SKIP → Step 5 |

**PRE-CONDITION:** Both conditions must be true:
1. mode == 1 (Step Manual Approve was chosen in Step 3)
2. current_step.status == "awaiting-approval" (Step 5 auto-retry loop completed with all verification criteria PASS)

**PROCEDURE (if executing):**

Present approval question:
```
AskUserQuestion:
  question: "Step {N}: {STEP_NAME} passed verification (iteration {ITERATION}). Approve and continue to next step?"
  header: "Step Approval"
  options:
    - "Approve - Mark complete and move to next step"
    - "Request changes - Back to implementation"
```

**DECISION BASED ON USER RESPONSE:**

**If user selects "Approve":**
1. Record approval in progress.json: approvals.mode_1_manual_approvals += {step: N, approved_at: now, approved_by: "user", iteration: ITERATION}
2. Set step status = "complete"
3. Clear awaiting_approval_since = null
4. Loop back to Step 1 (to handle next step if more exist, or finalize if all done)

**If user selects "Request changes":**
1. Update progress.json: step status = "needs-fix"
2. Increment step.iteration (prepare for re-implementation)
3. Clear awaiting_approval_since = null
4. Loop back to Step 1 (will go to Step 5 for re-implementation)

**POST-CONDITION:** 
- If approved: step marked complete, approval recorded, continue to next
- If rejected: step marked needs-fix, iteration incremented, ready for re-implementation loop
- Either way: progress.json updated, loop restarts

---

### Step 5: Execute with Subagent (If Work Needed)

**GUARD - ONLY EXECUTE IF:**
```
current_step.status IN ["pending", "needs-fix", "verification"]
```

If status == "complete" or status == "awaiting-approval": SKIP this step, loop back to Step 1

| Status | Action |
|--------|--------|
| `pending` | Execute auto-retry loop (implement → verify until pass) |
| `needs-fix` | Execute auto-retry loop (re-implement → verify until pass) |
| `verification` | Execute verify only (implement already done) |
| `complete` | SKIP (already done) |
| `awaiting-approval` | SKIP (waiting for user approval in Step 4) |

**PRE-CONDITION:** 
- progress.json loaded and mode set
- current step identified
- status is NOT "complete" and NOT "awaiting-approval"

**PROCEDURE:**

#### Auto-Retry Loop (Both Mode 1 and Mode 2)

**IF** status IN [pending, needs-fix]:
```
iteration_count = 0

WHILE status != "awaiting-approval" AND status != "complete":
  1. Dispatch implementer
     → Set status = "verification"
  
  2. Dispatch verifier
     → Read verification results from step-{N}.md
  
  3. Check verification results:
     IF all criteria PASSED:
       IF mode == 1:
         → Set status = "awaiting-approval"
         → Set awaiting_approval_since = now (ISO8601)
         → Break loop
       ELSE (mode == 2):
         → Set status = "complete"
         → Break loop
     ELSE (any criterion FAILED):
       → Set status = "needs-fix"
       → Increment iteration counter (for tracking)
       → Read updated step-{N}.md with failures
       → Continue loop (re-run implementer)

END WHILE
```

**This means:** 
- Mode 1: implement → verify → (if fail) re-implement → verify → ... until success → awaiting-approval (pause for user)
- Mode 2: implement → verify → (if fail) re-implement → verify → ... until success → complete (auto-continue to next step)
- No iteration limits — continues until verification passes

**Agent Dispatch Examples:**

```python
# For implementer
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

# For verifier
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

**AFTER LOOP COMPLETES:**

1. **Status is now:**
   - Mode 1: `awaiting-approval` (paused, waiting for user approval)
   - Mode 2: `complete` (auto-continued to next step)

2. **If status == "verification" (never entered loop):**
   - Dispatch verifier only (step already implemented)
   - After verify: check results and set status as above

3. **Loop back to Step 1** to process new status
   - Step 1: Read updated state
   - Step 1.5: Skip (already in-progress)
   - Step 2: Show what's next based on new status
   - Step 3: Ask mode again
   - Step 4: If mode==1 AND status==awaiting-approval, ask approval
   - Step 5: If not complete, continue or finalize

**POST-CONDITION:** 
- step-{N}.md has Implementation or Verification section filled (if Mode 1 with auto-retry: filled from last iteration)
- progress.json updated with new status (awaiting-approval if Mode 1 success, or per-step status if Mode 2)
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
- ❌ Ask approval (Step 4) unless Step Manual Approve mode AND status == "awaiting-approval"
- ❌ In Mode 1 or Mode 2 with status IN [pending, needs-fix]: ask user between implement/verify cycles - let auto-retry loop run until verification passes
- ❌ Break auto-retry loop before verification passes (no iteration limits - continues until success or max retries handled by external timeout)

**⚠️ WARNING:** 
- Auto-retry loops have no iteration limit. If verification criteria are impossible to satisfy, the loop will continue indefinitely.
- Verifier must be robust: criteria should be testable and achievable by implementer.
- For unsatisfiable criteria: verifier will keep failing, implementer will keep re-trying, loop continues forever.
- Recommendation: If a step is stuck in infinite retry after several executions, review verification criteria and either fix them or reconsider the step goal.

---

## Example Flow: 3 Steps, Step Manual Approve (Mode 1)

### Case 1: Step Passes on First Try

```
Start: /workflow:execute

Step 1: Read state
  → Step 1 pending, Mode not set, Workflow initialized

Step 1.5: Confirm subagent execution
  → "This workflow will use subagents. Continue?"
  → User: Agree
  → Continue to Step 2

Step 2: Show next
  → "Will implement step 1, then verify automatically (no user approval until pass)"

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve
  → Set mode = 1, workflow_status = in-progress

Step 4: Check approval
  → Step 1 status = pending (not awaiting-approval) → Skip

Step 5: Execute with subagent (Mode 1 Auto-Retry Loop)
  → MAX_ITERATIONS = 5, iteration = 0
  → Dispatch Agent(implementer step 1)
  → Agent returns, status = verification
  → Dispatch Agent(verifier step 1)
  → Agent returns, all criteria PASS
  → Set status = awaiting-approval
  → Set awaiting_approval_since = now
  → Break loop
  → Loop back to Step 1

---

/workflow:execute (Second Run - User Approval)

Step 1: Read state
  → Step 1 awaiting-approval, Mode = 1

Step 1.5: Skip (workflow_status is in-progress)

Step 2: Show next
  → "Step 1 passed verification. Ready for your approval."

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve (can change if wanted)

Step 4: Check approval (Mode = 1 AND status = awaiting-approval)
  → "Step 1: {STEP_NAME} passed verification. Approve and continue to next step?"
  → User: Approve
  → Status = complete, loop back to Step 1 for Step 2

[Repeat for steps 2 & 3...]

All steps complete → Finalize
```

### Case 2: Step Needs Multiple Tries (Verification Fails, Re-implement, Verify Again Passes)

```
Start: /workflow:execute

Step 1: Read state
  → Step 1 pending, Mode = 1 (user already chose this in prior attempt)

Step 1.5: Skip (workflow_status is in-progress)

Step 2: Show next
  → "Will implement step 1, verify, and re-try if needed (no user approval until pass)"

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve (confirm or keep same)

Step 4: Check approval
  → Step 1 status = pending (not awaiting-approval) → Skip

Step 5: Execute with subagent (Mode 1 Auto-Retry Loop)
  → MAX_ITERATIONS = 5, iteration = 0
  
  **Iteration 1:**
  → Dispatch Agent(implementer step 1)
  → Agent returns, status = verification
  → Dispatch Agent(verifier step 1)
  → Agent returns, 1 criterion PASSED, 2 criteria FAILED
  → Set status = needs-fix
  → iteration = 1, continue loop
  
  **Iteration 2:**
  → Dispatch Agent(implementer step 1) [re-implement based on failures]
  → Agent returns, status = verification
  → Dispatch Agent(verifier step 1)
  → Agent returns, all criteria PASS
  → Set status = awaiting-approval
  → Set awaiting_approval_since = now
  → Break loop
  → Loop back to Step 1

---

/workflow:execute (Second Run - User Approval)

Step 1: Read state
  → Step 1 awaiting-approval, Mode = 1, iteration = 2

Step 1.5: Skip (workflow_status is in-progress)

Step 2: Show next
  → "Step 1 passed verification on iteration 2. Ready for your approval."

Step 3: Ask workflow mode (EVERY execution)
  → "How should approval work?"
  → User: Step Manual Approve

Step 4: Check approval (Mode = 1 AND status = awaiting-approval)
  → "Step 1: {STEP_NAME} passed verification (after 2 attempts). Approve?"
  → User: Approve
  → Status = complete, proceed to Step 2

[Repeat for steps 2 & 3...]

All steps complete → Finalize
```

### Mode 2 (Final Approve) - Auto-Retry Without Pauses

```
Start: /workflow:execute

Step 1: Read state
  → Step 1 pending, Mode not set

Step 1.5: Confirm subagent execution
  → User: Agree

Step 2: Show next
  → "Will implement all steps, verify each, and auto-continue (approve at end only)"

Step 3: Ask workflow mode
  → User: Final Approve (mode = 2)

Step 4: Check approval
  → Step 1 status = pending (not awaiting-approval) → Skip

Step 5: Execute with subagent (Mode 2 Auto-Retry Loop)
  → MAX_ITERATIONS = unlimited
  
  **Iteration 1:**
  → Dispatch Agent(implementer step 1)
  → status = verification
  → Dispatch Agent(verifier step 1)
  → Result: 1 criterion FAILED
  → status = needs-fix, iteration = 1, continue loop (NO pause, NO user interaction)
  
  **Iteration 2:**
  → Dispatch Agent(implementer step 1) [re-implement based on failures]
  → status = verification
  → Dispatch Agent(verifier step 1)
  → Result: All criteria PASS
  → Set status = complete (no awaiting-approval)
  → Break loop, continue to Step 1 for Step 2

[Repeat without pauses or user input for steps 2 & 3...]

All steps complete → Finalize with single user approval at the very end
```

---

See [EXECUTION_MODES.md](../docs/EXECUTION_MODES.md) for detailed execution method comparison.
