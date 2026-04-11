---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute Skill - STRICT ACTION PROCEDURE

When user invokes `/workflow:execute`, YOU MUST follow this exact procedure. These are NOT descriptions—these are YOUR ACTIONS.

## Mandatory Startup Checklist

- [ ] Read PLAN.md with Read() tool
- [ ] Read all step-N.md files with Read() tool
- [ ] Ask user with AskUserQuestion() tool
- [ ] Only then dispatch Agent()

---

## YOUR ACTIONS (Follow Exactly)

### Action 1: Check Current State

```
Read: .workflow/TASK_NAME/PLAN.md
- Find: current status (planning/executing/ready-for-review)
- Note: mode (1 or 2)
```

```
Read each: .workflow/TASK_NAME/steps/step-1.md through step-N.md
- Find: FIRST step where status ≠ "complete"
- Note: that step's status (pending/implementation/needs-fix/verification)
- Note: step name and number
```

### Action 2: Ask User for Confirmation (MANDATORY - DO NOT SKIP)

Use AskUserQuestion tool. MUST ask before proceeding:

```
AskUserQuestion:
  questions:
    - question: "Resume workflow execution from this point?"
      header: "Execution Status"
      options:
        - label: "Yes, continue"
          description: "Current step: {STEP_NUMBER} - {STEP_NAME}, Status: {STATUS}"
        - label: "No, stop"
          description: "Cancel execution"
```

**IF user says "No":** STOP. Do not proceed.

**IF user says "Yes":** Continue to Action 3.

### Action 3: Main Loop - Find Next Action

Read frontmatter `status` from first non-complete step:

**IF status = "pending" or "implementation":**
→ Go to Action 4a (Dispatch Implementer)

**IF status = "needs-fix":**
→ Go to Action 4a (Dispatch Implementer with issues)

**IF status = "verification":**
→ Go to Action 4b (Dispatch Verifier)

**IF status = "complete":**
→ Go to Action 5 (Ask for Approval)

### Action 4a: Dispatch Implementer Subagent

**CRITICAL: Use Agent() tool. NEVER use Skill().**

Get values from step-N.md:
- TASK_NAME = from PLAN.md
- N = step number
- STEP_NAME = step heading
- ABSOLUTE_PATH = /home/illia/work/claude-workflow-plugin
- ITERATION = from step-N.md frontmatter

Call Agent():

```
Agent(
  description: "Implementer for step {N}",
  subagent_type: "general-purpose",
  prompt: """You are implementer for {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1 or 2 from PLAN}.

You have full permissions: read/write files, run commands, update step files.
Do NOT ask permission. Work independently.

READ FIRST:
- Step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}.md
- PLAN.md: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/PLAN.md

YOUR TASK:
1. Understand the step goal and success criteria
2. Make all required code changes
3. If ORM models changed: run migrations with `npm run migrate`
4. Run tests: `npm test` - all must pass
5. Update step-{N}.md frontmatter: status = "verification"
6. Increment iteration number
7. Fill Implementation section with what you did
8. Return brief summary (under 100 words)

If blocked: report status BLOCKED with details
If need context: report status NEEDS_CONTEXT with details
If uncertain: report status DONE_WITH_CONCERNS
Otherwise: complete normally"""
)
```

After Agent returns:
- [ ] Read step-N.md again
- [ ] Check new status in frontmatter
- [ ] Go back to Action 3

### Action 4b: Dispatch Verifier Subagent

**CRITICAL: Use Agent() tool. NEVER use Skill().**

Get values from step-N.md:
- TASK_NAME = from PLAN.md
- N = step number
- STEP_NAME = step heading
- ABSOLUTE_PATH = /home/illia/work/claude-workflow-plugin

Call Agent():

```
Agent(
  description: "Verifier for step {N}",
  subagent_type: "general-purpose",
  prompt: """You are verifier for {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1 or 2 from PLAN}.

You have full permissions: read files, run builds/tests, update step files.
Do NOT ask permission. Test independently.

READ FIRST:
- Step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}.md
- PLAN.md: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/PLAN.md

YOUR TASK:
1. Understand verification criteria from step file
2. Build project: `npm run build`
3. Check migrations ran: `psql -c "SELECT * FROM schema_migrations"`
   - If incomplete: FAIL (implementer's job to fix)
4. Run all tests: `npm test`
   - If any fail: FAIL (implementer's job to fix)
5. Manually test each criterion by RUNNING code:
   - API endpoints: call with curl, check response
   - Database: insert data, verify in DB
   - Pages: load page, verify content
6. Document results with proof

DECISION:
- If ALL pass: Update step-{N}.md frontmatter status = "complete"
- If ANY fail: Update status = "needs-fix", document Issues Identified
7. Return brief summary: PASS or FAIL with evidence

Never run migrations yourself—only verify they ran."""
)
```

After Agent returns:
- [ ] Read step-N.md again
- [ ] Check new status (complete or needs-fix)
- [ ] Go back to Action 3

### Action 5: Step is Complete - Ask for Approval (Mode 1 only)

Check PLAN.md mode:

**IF Mode 2:** Skip to Action 3 (move to next step auto)

**IF Mode 1:** Use AskUserQuestion:

```
AskUserQuestion:
  questions:
    - question: "Step {N} complete. Continue to step {N+1}?"
      header: "Approval Required"
      options:
        - label: "Approve and continue"
          description: "Move to next step"
        - label: "Request changes"
          description: "Return to implementation"
```

**IF user says "Approve":**
→ Go to Action 3 (next step)

**IF user says "Request changes":**
- Edit step-N.md: set status = "needs-fix"
→ Go to Action 3 (loop back to implementer)

### Action 6: All Steps Complete

When no more non-complete steps exist:

```
Edit PLAN.md:
- Set status = "ready-for-review"

AskUserQuestion:
  questions:
    - question: "All steps verified. Approve final completion?"
      header: "Final Approval"
      options:
        - label: "Yes, finalize"
          description: "Create final commit and complete workflow"
        - label: "No, review more"
          description: "Return to earlier steps"
```

**IF "Yes":**
→ Call Agent(finalize) [future: workflow:finalize skill]

**IF "No":**
→ Go back to Action 3 (loop through steps)

---

## Critical Rules

**DO:**
- ✅ Use Read() for PLAN.md and step files
- ✅ Use AskUserQuestion() before dispatching ANY subagent
- ✅ Use Agent() to dispatch implementer/verifier
- ✅ Use Edit() to update PLAN.md status
- ✅ Wait for Agent() to return before reading step files

**DON'T:**
- ❌ Skip AskUserQuestion() - always ask user first
- ❌ Use Skill() for implementer/verifier - that runs in main session
- ❌ Read source code files - only read workflow files
- ❌ Run tests yourself - dispatch to subagent
- ❌ Make implementation decisions - that's subagent's job

---

## Implementation Checklist

When implementing execute, ensure:

- [ ] Step 1: Always read PLAN.md and step files first
- [ ] Step 2: Always ask user AskUserQuestion() - this is mandatory
- [ ] Step 3: Find first non-complete step (read frontmatter status)
- [ ] Step 4a: If pending/implementation/needs-fix → Agent(implementer)
- [ ] Step 4b: If verification → Agent(verifier)
- [ ] Step 5: If complete → AskUserQuestion for approval (Mode 1)
- [ ] Step 6: Loop back after any Agent() returns
- [ ] All steps complete → Ask for final approval

This is NOT documentation. These are YOUR ACTIONS to perform.
