---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute Skill

Orchestrate workflow execution. Read plan, ask user, dispatch subagents via Skill invocation.

**CRITICAL:** This skill runs in MAIN SESSION. All work delegated to isolated subagents.

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

**CRITICAL:** Subagent will use Skill(workflow:implementer)

```
Agent(
  description: "Implement step {N}",
  subagent_type: "general-purpose",
  prompt: """
You are implementer for {TASK_NAME}, step {N}: {STEP_NAME}
Mode: {1|2}
Task directory: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/

PROCEDURE:
1. Invoke Skill(workflow:implementer)
2. Follow the skill exactly
3. Report when done

The skill will guide you through:
- Reading step requirements
- Implementing code changes
- Running migrations
- Running tests
- Updating step file
- Returning summary
"""
)
```

After Agent returns:
- Read step-N.md
- Check new status
- Loop back to Phase 3

### Phase 5: Dispatch Verifier

**CRITICAL:** Subagent will use Skill(workflow:verifier)

```
Agent(
  description: "Verify step {N}",
  subagent_type: "general-purpose",
  prompt: """
You are verifier for {TASK_NAME}, step {N}: {STEP_NAME}
Mode: {1|2}
Task directory: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/

PROCEDURE:
1. Invoke Skill(workflow:verifier)
2. Follow the skill exactly
3. Report when done

The skill will guide you through:
- Reading verification criteria
- Setting up environment
- Checking migrations
- Running tests
- Manual testing each criterion
- Updating step file
- Returning summary
"""
)
```

After Agent returns:
- Read step-N.md
- Check new status (complete or needs-fix)
- Loop back to Phase 3

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
Edit: .workflow/{TASK_NAME}/PLAN.md
Set: status = "ready-for-review"

AskUserQuestion:
  questions:
    - question: "All steps verified. Finalize?"
      options:
        - label: "Yes, finalize"
        - label: "No, review more"
```

**IF Yes:**
```
Agent(
  description: "Finalize workflow",
  subagent_type: "general-purpose",
  prompt: """
You are finalizer for {TASK_NAME}
Task directory: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/

PROCEDURE:
1. Invoke Skill(workflow:finalize)
2. Follow the skill exactly
3. Report when done

The skill will:
- Verify all steps complete
- Create final commit
- Update PLAN.md
- Report completion
"""
)
```

Done!

**IF No:** Loop back to Phase 3 (allow more changes)

---

## Critical Rules

**MUST DO:**
- ✅ Read PLAN.md and step files first
- ✅ Ask user via AskUserQuestion BEFORE any Agent()
- ✅ Use Agent(subagent_type="general-purpose")
- ✅ Pass minimal context to Agent
- ✅ Instruct Agent to use Skill()
- ✅ Wait for Agent() to return

**NEVER DO:**
- ❌ Skip AskUserQuestion
- ❌ Put full procedure in Agent prompt
- ❌ Tell subagent to do work directly
- ❌ Read source files (only workflow files)
- ❌ Run tests yourself
- ❌ Write code yourself

---

## Subagent Responsibility

When you dispatch Agent with "Use Skill(workflow:implementer)":
- Subagent OWNS the work
- Subagent reads Skill
- Subagent follows Skill procedure
- Subagent updates files
- Subagent reports back to you

You simply:
- Read results
- Update PLAN.md status table
- Loop to next action
