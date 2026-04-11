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
  │   ├─ Phase 5: Agent(workflow:verifier) ← ISOLATED
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

Extract:
- PLAN.name, mode (1|2), status
- First step where status ≠ "complete"
- That step's status
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

### Phase 5: Dispatch Verifier

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
- Read step-{N}.md verification criteria
- Set up environment (install deps, build, etc)
- Run all tests (must pass)
- Manually verify each criterion in checklist
- Update step-{N}.md status to "complete" or "needs-fix"
- Document any issues found
- Report detailed findings

**DONE WHEN:**
step-{N}.md shows status: complete OR needs-fix
**AND** implementation notes section updated with verification results

**IF needs-fix:**
- Document exact issues in step file
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
