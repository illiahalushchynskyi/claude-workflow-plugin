---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute Skill

Orchestrate workflow execution. Read plan, ask user, dispatch subagents.

**CRITICAL:** This skill runs in MAIN SESSION. All work delegated to **isolated subagents via Agent tool**.

## Subagent Architecture

```
execute (main session)
  ├─ Phase 1: Load State
  ├─ Phase 2: Ask User
  ├─ Phase 3: Main Loop
  │   ├─ Phase 4: Agent(workflow:implementer) ← ISOLATED
  │   ├─ Phase 5A: Reset Verification Progress (clear previous results)
  │   ├─ Phase 5B: Agent(workflow:verifier) ← ISOLATED (starts fresh)
  │   └─ Phase 6: Ask User (Mode 1 only)
  └─ Phase 7: Agent(workflow:finalize) ← ISOLATED
```

## When to Use

Use `workflow:execute` when:
- Workflow has been bootstrapped (PLAN.md exists)
- User wants to start or resume execution
- PLAN.md status is `planning` or `executing`

---

## YOUR ACTIONS (EXACT PROCEDURE)

### Phase 1: Load State

```
Read: .workflow/TASK_NAME/PLAN.md
Read: .workflow/TASK_NAME/steps/step-1.md through step-N.md
Read: .workflow/TASK_NAME/progress.json

Extract:
- PLAN.name, mode (1|2), status
- First step where status ≠ "complete"
- That step's status
```

#### Step 1: Load Current State from progress.json

Read progress.json to understand where we are in the workflow:

```bash
# Check progress.json exists
if [ ! -f ".workflow/${TASK_NAME}/progress.json" ]; then
  FAIL: "progress.json not found. Run bootstrap first."
fi

# Load current state
CURRENT_STEP=$(jq '.current_step' .workflow/${TASK_NAME}/progress.json)
INCOMPLETE_STEP=$(jq '.steps | to_entries[] | select(.value.status != "complete") | .key' .workflow/${TASK_NAME}/progress.json | head -1)
MODE=$(jq '.mode' .workflow/${TASK_NAME}/progress.json)

echo "Current step: $CURRENT_STEP"
echo "First incomplete step: $INCOMPLETE_STEP"
echo "Mode: $MODE (1=step-by-step, 2=end-to-end)"
```

#### Step 2: Find Next Step to Execute

Determine which step runs next based on progress.json:

```bash
# Find first step that's not complete
NEXT_STEP=$(jq '.steps | to_entries[] | select(.value.status != "complete") | .key' .workflow/${TASK_NAME}/progress.json | head -1)

if [ -z "$NEXT_STEP" ]; then
  echo "✓ All steps complete!"
  # Call finalize (dispatch workflow:finalize skill)
  echo "Ready for finalize step"
else
  echo "Next step to run: $NEXT_STEP"
  STEP_STATUS=$(jq ".steps[\"$NEXT_STEP\"].status" .workflow/${TASK_NAME}/progress.json)
  echo "Current status: $STEP_STATUS"
  
  # Status values: pending → implementation → verification → needs-fix → complete
  case "$STEP_STATUS" in
    "pending")
      echo "Dispatching implementer for step $NEXT_STEP..."
      # Call workflow:implementer skill
      ;;
    "needs-fix")
      echo "Step $NEXT_STEP failed verification. Re-dispatching implementer for fixes..."
      # Call workflow:implementer skill again
      ;;
    "verification")
      echo "Dispatching verifier for step $NEXT_STEP..."
      # Call workflow:verifier skill
      ;;
    *)
      echo "Unknown status: $STEP_STATUS"
      ;;
  esac
fi
```

### Phase 2: Ask User (MANDATORY)

```
AskUserQuestion:
  questions:
    - question: "Start workflow from step {N}?"
      header: "Status"
      options:
        - label: "Yes, proceed"
          description: "{STEP_NAME} - Status: {STATUS}"
        - label: "No, stop"
```

**IF No:** STOP execution.
**IF Yes:** Continue to Phase 3.

#### Progress Update Methods

Helper functions for updating progress.json during execution:

**Update step status:**
```bash
update_step_status() {
  local step_num=$1
  local new_status=$2
  
  jq --arg step "$step_num" --arg status "$new_status" \
    ".steps[\$step].status = \$status | 
     .steps[\$step].updated_at = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
    .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
    mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json
}
```

**Record step completion:**
```bash
mark_step_complete() {
  local step_num=$1
  local iteration=$2
  
  jq --arg step "$step_num" --arg iter "$iteration" \
    ".steps[\$step].status = \"complete\" | 
     .steps[\$step].iteration = (\$iter | tonumber) | 
     .steps[\$step].approval_date = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
    .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
    mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json
}
```

**Update implementation timestamps:**
```bash
set_implementation_start() {
  local step_num=$1
  jq --arg step "$step_num" \
    ".steps[\$step].implementation_start = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
    .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
    mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json
}

set_implementation_end() {
  local step_num=$1
  jq --arg step "$step_num" \
    ".steps[\$step].implementation_end = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
    .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
    mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json
}
```

### Phase 3: Main Loop

**REPEAT** until all steps complete:

1. Read step-N.md frontmatter status
2. Check which phase to run:
   - `pending`, `implementation`, `needs-fix` → Phase 4
   - `verification` → Phase 5
   - `complete` → Phase 6
3. Continue loop after phase

### Phase 4: Dispatch Implementer

**CRITICAL:** Use Agent tool to dispatch implementer as isolated subagent.

```python
Agent(
  description: f"Implement {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are implementing step {N} of {TASK_NAME}.

**INSTRUCTIONS:**
1. Invoke: Skill(skill: "workflow:implementer")
2. Follow the skill procedure exactly
3. The skill will guide you through implementation

**CONTEXT FROM EXECUTE:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: {TASK_DIR_PATH}
- Mode: {MODE} (1=step-by-step, 2=continuous)
- Current working directory: {PROJECT_ROOT}

**YOUR RESPONSIBILITY:**
- Read step-{N}.md requirements
- Implement all code changes
- Run all tests (must pass)
- Commit your work
- Update step-{N}.md status to "verification"
- Report summary when complete

**DONE WHEN:**
step-{N}.md shows status: verification
"""
)
```

**After Agent returns:**
```
1. Read: .workflow/{TASK_NAME}/steps/step-{N}.md
2. Check: frontmatter status field
3. IF status == "verification": ✓ Continue to Phase 5
4. IF status != "verification": ✗ FAIL (implementer did not complete)
   → Ask user: "Implementer failed. Review and retry?"
   → Option to request changes
5. Loop back to Phase 3
```

#### After Implementer Returns

When implementer completes step marking it as ready for verification:

1. Update progress.json:
   ```bash
   update_step_status $NEXT_STEP "verification"
   set_implementation_end $NEXT_STEP
   ```

2. Dispatch verifier skill for this step

### Phase 5A: Reset Verification Progress

**BEFORE dispatching verifier, clear previous verification results:**

```
Read: .workflow/{TASK_NAME}/steps/step-{N}.md

IF verification section already exists:
  Clear it:
  - Remove all previous verification notes
  - Remove all previous criteria results
  - Clear "Issues Identified" section
  
KEEP:
  - Implementation section (implementer's notes)
  - Acceptance criteria definitions
```

This ensures verifier starts fresh from the beginning of each step.

### Phase 5B: Dispatch Verifier

**CRITICAL:** Use Agent tool to dispatch verifier as isolated subagent.

```python
Agent(
  description: f"Verify {TASK_NAME} step {N}: {STEP_NAME}",
  subagent_type: "general-purpose",
  prompt: f"""
You are verifying step {N} of {TASK_NAME}.

**INSTRUCTIONS:**
1. Invoke: Skill(skill: "workflow:verifier")
2. Follow the skill procedure exactly
3. The skill will guide you through verification

**CONTEXT FROM EXECUTE:**
- Task: {TASK_NAME}
- Step: {N} ({STEP_NAME})
- Task directory: {TASK_DIR_PATH}
- Mode: {MODE} (1=step-by-step, 2=continuous)
- Current working directory: {PROJECT_ROOT}
- Previous status: {PREV_STATUS}

**YOUR RESPONSIBILITY:**
- Read step-{N}.md acceptance criteria (start fresh)
- Set up environment (install deps, build, etc)
- Run all tests (must pass)
- For EACH criterion: manually verify and mark ✓ passed or ✗ not passed
- Update step-{N}.md status to "complete" or "needs-fix"
- Document detailed results with evidence
- Report findings

**DONE WHEN:**
step-{N}.md shows status: complete OR needs-fix
**AND** Verification section filled with all criteria results

**IF needs-fix:**
- Document exact issues and reasons
- Implementer will see and re-work
"""
)
```

**After Agent returns:**
```
1. Read: .workflow/{TASK_NAME}/steps/step-{N}.md
2. Check: frontmatter status field
3. IF status == "complete":
   → Phase 6 (Mode 1) OR Phase 3 next step (Mode 2)
4. IF status == "needs-fix":
   → Loop back to Phase 3 (implementer re-works this step)
5. IF status != "complete" AND != "needs-fix":
   → ✗ FAIL (verifier did not complete)
   → Ask user for guidance
```

#### After Verifier Returns

**If verifier marks PASS (all criteria verified):**

```bash
mark_step_complete $NEXT_STEP $ITERATION

if [ "$MODE" == "1" ]; then
  # Mode 1: request human approval before continuing
  echo "✓ Step $NEXT_STEP passed verification"
  echo "Waiting for human approval before next step..."
  # Pause here for human approval (outside this skill)
else
  # Mode 2: continue automatically to next step
  echo "✓ Step $NEXT_STEP complete and approved (Mode 2)"
  # Loop back to find next step
fi
```

**If verifier marks FAIL (criteria not met):**

```bash
update_step_status $NEXT_STEP "needs-fix"
CURRENT_ITERATION=$(jq ".steps[\"$NEXT_STEP\"].iteration" .workflow/${TASK_NAME}/progress.json)
NEXT_ITERATION=$((CURRENT_ITERATION + 1))

jq --arg step "$NEXT_STEP" --arg iter "$NEXT_ITERATION" \
  ".steps[\$step].iteration = (\$iter | tonumber)" \
  .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
  mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json

echo "✗ Step $NEXT_STEP needs fixes"
echo "Iteration incremented to $NEXT_ITERATION"
# Loop back to dispatch implementer again for same step
```

### Phase 6: Step Complete - Ask Approval (Mode 1 only)

**IF PLAN mode = 2:** Skip, go back to Phase 3 (next step)

**IF PLAN mode = 1:**

```
AskUserQuestion:
  questions:
    - question: "Step {N} complete. Continue?"
      options:
        - label: "Approve and continue"
        - label: "Request changes"
```

**IF Approve:** Loop back to Phase 3 (next step)
**IF Request changes:** Set step status = "needs-fix", loop to Phase 3

### Phase 7: All Steps Complete

When no more non-complete steps:

```
1. Edit: .workflow/{TASK_NAME}/PLAN.md
   Set: status = "ready-for-review"

2. AskUserQuestion:
   - question: "All steps verified. Finalize workflow?"
   - options:
     - label: "Yes, finalize and commit"
       description: "Create final commit and mark complete"
     - label: "No, review more"
       description: "Review changes or re-run verification"
```

**IF "Yes, finalize":**
```python
Agent(
  description: f"Finalize {TASK_NAME} workflow",
  subagent_type: "general-purpose",
  prompt: f"""
You are finalizing {TASK_NAME}.

**INSTRUCTIONS:**
1. Invoke: Skill(skill: "workflow:finalize")
2. Follow the skill procedure exactly
3. The skill will guide you through finalization

**CONTEXT FROM EXECUTE:**
- Task: {TASK_NAME}
- Task directory: {TASK_DIR_PATH}
- Current working directory: {PROJECT_ROOT}

**YOUR RESPONSIBILITY:**
- Verify all steps show status: complete
- Create comprehensive final commit
- Update PLAN.md status to "complete"
- Generate summary of all changes
- Report completion

**DONE WHEN:**
PLAN.md shows status: complete
AND final commit exists in git log
"""
)
```

After finalize completes, progress.json will have:
- All steps: status = "complete"
- PLAN.md updated with completed steps
- Final commit created with comprehensive summary

Then: **WORKFLOW COMPLETE** ✓

**IF "No, review more":**
- Loop back to Phase 3 (allow re-work of any step)

---

## Critical Rules

**MUST DO:**
- ✅ Read PLAN.md and step files first
- ✅ Ask user via AskUserQuestion BEFORE first Agent()
- ✅ Use Agent(subagent_type="general-purpose") for implementer/verifier/finalize
- ✅ Pass context and task directory to Agent prompt
- ✅ Instruct Agent to Invoke Skill() for their task
- ✅ Before Phase 5B: Reset verification progress (clear previous results)
- ✅ Wait for Agent() to return completely
- ✅ Read step-N.md to verify status changed
- ✅ Handle failures and ask user

**NEVER DO:**
- ❌ Skip initial AskUserQuestion
- ❌ Write code or make changes yourself
- ❌ Run tests or migrations yourself
- ❌ Read/edit source files (only workflow files)
- ❌ Modify step files directly (agents do this)
- ❌ Assume Agent succeeded without reading files
- ❌ Dispatch multiple agents in parallel (go sequentially)

---

## Mode 2 Special Rules (Continuous with Verification Gates)

If PLAN.mode == 2:

```
Between steps:
- NO human approval pause
- Verifier gates advancement (must complete before next step)
- Implementer can start step N+1 as soon as step N verifies complete

Check before dispatching implementer for step N:
- IF N > 1: MUST check step N-1 status == "complete"
- IF step N-1 != "complete": WAIT (verifier still working on N-1)
- Only dispatch implementer for N when N-1 is done
```

---

## Error Handling

**If Agent times out or fails:**
```
1. Check Agent result output
2. If partial completion: ask user "Proceed or abort?"
3. If clear failure: document and ask user "Retry or abandon?"
4. Update PLAN.md notes with error
5. Proceed based on user decision
```

**If step file status didn't change:**
```
1. Read step-{N}.md frontmatter again
2. If still old status: Agent didn't complete
3. Ask user: "Subagent didn't finish. Retry?"
4. Can retry same Agent or abort
```

---

## Subagent Responsibility

When you dispatch Agent(workflow:implementer):
- Subagent OWNS the implementation
- Subagent reads Skill(workflow:implementer)
- Subagent modifies code and tests
- Subagent updates step-{N}.md status
- Subagent reports summary

When you dispatch Agent(workflow:verifier):
- Subagent OWNS the verification
- Subagent reads Skill(workflow:verifier)
- Subagent runs tests and checks criteria
- Subagent updates step-{N}.md status
- Subagent documents findings

**You (execute) simply:**
- Read results (PLAN.md, step-N.md)
- Verify status changed as expected
- Loop to next action or ask user
- Never second-guess subagent work (they own it)
