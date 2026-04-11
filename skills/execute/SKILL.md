---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute Skill

Orchestrate workflow execution. Read plan, ask user, dispatch subagents, track progress.

## When to Use

Use `workflow:execute` when:
- Workflow has been bootstrapped (PLAN.md exists)
- User wants to start or resume execution
- PLAN.md status is `planning` or `executing`

## Input

- `.workflow/TASK_NAME/PLAN.md`
- `.workflow/TASK_NAME/steps/step-*.md`

## Output

- Updated PLAN.md and step files
- Workflow advances toward completion

---

## YOUR PROCEDURE (Execute These Steps)

### Phase 1: Load State

```
Read: .workflow/TASK_NAME/PLAN.md
  Extract: name, mode (1|2), status, step list

Read: .workflow/TASK_NAME/steps/step-1.md through step-N.md
  Extract: each step status in frontmatter

Find: FIRST step where status ≠ "complete"
```

### Phase 2: Ask User (MANDATORY)

Use AskUserQuestion BEFORE any work:

```
AskUserQuestion:
  questions:
    - question: "Start workflow execution from step {N}?"
      header: "Status"
      options:
        - label: "Yes, proceed"
          description: "Status: {STEP_STATUS}, Name: {STEP_NAME}"
        - label: "No, stop"
          description: "Cancel execution"
```

**IF user says "No":** STOP. Don't proceed.
**IF user says "Yes":** Continue to Phase 3.

### Phase 3: Main Loop

Repeat until all steps complete:

1. **Read** step-N.md frontmatter to check current status
2. **Check status:**
   - `pending`, `implementation`, `needs-fix` → **Phase 4 (Implementer)**
   - `verification` → **Phase 5 (Verifier)**
   - `complete` → **Phase 6 (Approval)**
3. **Loop back** after phase completes

### Phase 4: Dispatch Implementer

```
Agent(
  description: "Implement step {N}",
  subagent_type: "general-purpose",
  prompt: """
You are implementer for step {N}: {STEP_NAME}
Mode: {1|2}
Task: .workflow/{TASK_NAME}/

READ:
- Step file: .workflow/{TASK_NAME}/steps/step-{N}.md
- PLAN.md: .workflow/{TASK_NAME}/PLAN.md

YOUR TASK:
1. Understand goal and success criteria
2. Make code changes following project patterns
3. If ORM models changed: run `npm run migrate`
4. Run `npm test` - all must pass
5. Update step-{N}.md:
   - Frontmatter: status = "verification"
   - Increment iteration number
   - Fill Implementation section
6. Return brief summary (under 100 words)

SUCCESS: status in step file = "verification"
BLOCKED/FAILED: Report status with details
"""
)
```

After Agent returns:
- Read step-N.md
- Check new status
- Loop back to Phase 3

### Phase 5: Dispatch Verifier

```
Agent(
  description: "Verify step {N}",
  subagent_type: "general-purpose",
  prompt: """
You are verifier for step {N}: {STEP_NAME}
Mode: {1|2}
Task: .workflow/{TASK_NAME}/

READ:
- Step file: .workflow/{TASK_NAME}/steps/step-{N}.md
- PLAN.md: .workflow/{TASK_NAME}/PLAN.md

YOUR TASK:
1. Understand verification criteria
2. Setup: `npm install && npm run build`
3. Check migrations: `psql -c "SELECT * FROM schema_migrations"`
   - Do NOT run migrations
   - If incomplete: FAIL (implementer must fix)
4. Run tests: `npm test` (all must pass)
5. Manually test EACH criterion:
   - API: call with curl, check response
   - Database: insert, query, verify persistence
   - Pages: load, check content
   - Docker: verify containers running
6. Document results with proof
7. Update step-{N}.md:
   - If all pass: status = "complete"
   - If any fail: status = "needs-fix", list Issues
8. Return brief summary (under 100 words)

SUCCESS: status in step file = "complete"
FAILED: status = "needs-fix" with issues list
"""
)
```

After Agent returns:
- Read step-N.md
- Check new status
- Loop back to Phase 3

### Phase 6: Step Complete - Ask Approval (Mode 1 only)

**IF PLAN mode = 2:** Skip to Phase 3 (next step)

**IF PLAN mode = 1:**

```
AskUserQuestion:
  questions:
    - question: "Step {N} complete. Continue to step {N+1}?"
      header: "Approval"
      options:
        - label: "Approve and continue"
        - label: "Request changes"
```

**IF user says "Approve":**
- Loop back to Phase 3 (next step)

**IF user says "Request changes":**
- Edit step-N.md: set status = "needs-fix"
- Loop back to Phase 3 (implementer fixes)

### Phase 7: All Steps Complete

When no more non-complete steps:

```
Edit: .workflow/{TASK_NAME}/PLAN.md
- Set: status = "ready-for-review"

AskUserQuestion:
  questions:
    - question: "All steps verified. Finalize workflow?"
      options:
        - label: "Yes, finalize"
        - label: "No, review more"
```

**IF user says "Yes":**
- Dispatch finalize (Agent with finalize prompt)
- All done

**IF user says "No":**
- Loop back to Phase 3 (allow more changes)

---

## Critical Rules

**MUST DO:**
- ✅ Read PLAN.md and all step files FIRST
- ✅ Ask user via AskUserQuestion() BEFORE any Agent()
- ✅ Use Agent(subagent_type="general-purpose") to dispatch work
- ✅ NEVER use Skill() in Agent prompt
- ✅ Wait for Agent() to return before reading files

**NEVER DO:**
- ❌ Skip the AskUserQuestion() - ALWAYS ask user first
- ❌ Use Skill(workflow:implementer) or Skill(workflow:verifier)
- ❌ Read source code files - only workflow files
- ❌ Run tests yourself - dispatch to Agent
- ❌ Write code yourself - dispatch to Agent

## State Management

Keep track in PLAN.md:
- Update step table status after each phase
- Update created/started/completed timestamps
- Set status: planning → executing → ready-for-review → approved → complete

All work tracked in step-N.md:
- Implementer sets status = "verification"
- Verifier sets status = "complete" or "needs-fix"
- Never manually edit step status fields
