# Claude Workflow Plugin - Detailed Design

## 1. Architecture Overview

### System Components

```
User (main session)
    ↓
execute skill (orchestrator in main session)
    ↓
┌─────────────────────┐
│ 1. Read PLAN.md     │
│ 2. Ask user (📋)    │
│ 3. Dispatch Agent   │
│ 4. Wait for result  │
│ 5. Update files     │
│ 6. Loop             │
└─────────────────────┘
    ↓                     ↓
Agent(implementer)      Agent(verifier)
(isolated context)      (isolated context)
    ↓                     ↓
[Code]              [Test/Verify]
[Migrate]           [Check migrations]
[Test]              [Run tests]
[Commit]            [Manual test]
[Report]            [Report]
    ↓                     ↓
step-N.md updated   step-N.md updated
status→verification status→complete/needs-fix
```

### Key Principle

**Orchestrator NEVER does implementation work.** ALL work is delegated to isolated subagents via Agent() tool.

---

## 2. File Structure

```
.workflow/
  TASK_NAME/
    PLAN.md                    # Overall plan status
    steps/
      step-1.md
      step-2.md
      ...
      step-N.md
    .workflow-config.json      # Optional config
```

### PLAN.md Structure

```yaml
---
name: TASK_NAME
title: Human-readable title
description: What this task accomplishes
mode: 1  # 1=Step-by-Step, 2=End-to-End
status: planning|executing|ready-for-review|approved|complete
created: 2026-04-11
started: null
completed: null
---

# Step List

| Step | Name | Status | Iteration | Notes |
|------|------|--------|-----------|-------|
| 1 | Task 1 | pending | 1 | Initial |
| 2 | Task 2 | pending | 1 | Initial |
```

### step-N.md Structure

```yaml
---
step_number: 1
name: Step name
status: pending|implementation|needs-fix|verification|complete
iteration: 1
created: 2026-04-11
---

# Step 1: Step Name

## Goal
[What this step accomplishes]

## Verification Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Implementation
[Filled by implementer]

### Files Modified/Created
- File 1
- File 2

### Implementation Notes
[What implementer did]

## Verification
[Filled by verifier]

### Verification Notes
[Test results, manual testing notes]

### Issues Identified (if FAIL)
[List of issues]
```

---

## 3. Workflow States

### PLAN.md Status Flow

```
planning
   ↓ (user starts)
executing
   ↓ (all steps complete)
ready-for-review
   ↓ (user approves)
approved
   ↓ (finalize runs)
complete
```

### step-N.md Status Flow

```
pending
   ↓ (implementer dispatched)
implementation
   ↓ (implementer finishes)
verification
   ↓ (verifier dispatched)
complete  (if PASS)
   ↓
[next step]

OR

verification
   ↓ (verifier finds issues)
needs-fix
   ↓ (implementer dispatched again)
implementation
   ↓ (loop back to verification)
```

---

## 4. Execute Skill Procedure (EXACT STEPS)

### Step 1: Load Current State

**Action: Claude executes these steps in main session**

1. `Read(.workflow/TASK_NAME/PLAN.md)`
   - Extract: name, mode (1|2), status, step list

2. `Read(.workflow/TASK_NAME/steps/step-1.md)`
   - Through step-N.md
   - Extract: each step's status

3. Find: **first step where status ≠ complete**

### Step 2: Ask User for Confirmation

**Action: Claude uses AskUserQuestion() BEFORE any subagent dispatch**

```
AskUserQuestion:
  questions:
    - question: "Start workflow from Step {N}? Status: {STATUS}"
      header: "Confirmation"
      options:
        - label: "Yes, proceed"
        - label: "No, stop"
```

**If No:** Stop execution.
**If Yes:** Continue to Step 3.

### Step 3: Determine Action

**Action: Claude reads step status and decides**

```
if step.status == "pending" or "implementation" or "needs-fix":
  → DISPATCH IMPLEMENTER

if step.status == "verification":
  → DISPATCH VERIFIER

if step.status == "complete":
  → ASK USER FOR APPROVAL (Mode 1 only)
  → MOVE TO NEXT STEP
```

### Step 4: Dispatch Implementer

**Action: Claude calls Agent() tool with full prompt**

```python
Agent(
  description: "Implement step {N}",
  subagent_type: "general-purpose",
  prompt: """Full implementation procedure inline (see step 5)"""
)
```

**CRITICAL: Do NOT call Skill(). Prompt must include ALL instructions inline.**

After Agent returns:
- `Read(step-N.md)` to check new status
- If status = "verification": loop back to Step 3
- If status = "needs-fix": already has issues, loop back to Step 3

### Step 5: Implementer Task (in isolated subagent)

**Requirements:**
1. Read step-N.md to understand goal
2. Read PLAN.md for context
3. Make code changes
4. **If ORM models changed: run migrations** (`npm run migrate`)
5. Run tests locally (`npm test`)
6. Commit changes
7. Update step-N.md:
   - Set frontmatter: `status: verification`
   - Fill Implementation section with notes
8. Return brief report (100 words max)

**Success criteria:** All tests pass, code builds, step-N.md status = verification

### Step 6: Dispatch Verifier

**Action: Claude calls Agent() tool with full prompt**

```python
Agent(
  description: "Verify step {N}",
  subagent_type: "general-purpose",
  prompt: """Full verification procedure inline (see step 7)"""
)
```

**CRITICAL: Do NOT call Skill(). Prompt must include ALL instructions inline.**

After Agent returns:
- `Read(step-N.md)` to check new status
- If status = "complete": loop back to Step 3 (next step)
- If status = "needs-fix": issues documented, loop back to Step 3 (implementer again)

### Step 7: Verifier Task (in isolated subagent)

**Requirements:**
1. Read step-N.md to understand criteria
2. Build project: `npm run build` (fail if errors)
3. **Check migrations:** `psql -c "SELECT * FROM schema_migrations"`
   - Do NOT run migrations yourself
   - If incomplete: FAIL (implementer's responsibility)
4. Run tests: `npm test` (all must pass)
5. Manually test each criterion:
   - API: call with `curl`, check response
   - Database: insert, query, verify
   - Pages: load, verify content
   - Docker: verify containers running
6. Document results with evidence
7. Update step-N.md:
   - If all pass: status = "complete", add PASS notes
   - If any fail: status = "needs-fix", document Issues Identified
8. Return brief report (100 words max)

**Success criteria:** All tests pass, all criteria verified, step-N.md status = complete

### Step 8: Ask for Approval (Mode 1 only)

**Action: Claude checks Mode and asks user**

```
if PLAN.mode == 1:
  AskUserQuestion:
    - question: "Approve step {N}?"
      options:
        - label: "Yes, continue"
        - label: "No, request changes"
```

**If Yes:** Update PLAN.md step table, loop back to Step 3 (next step)
**If No:** Set step status = "needs-fix", loop back to Step 3 (implementer again)

### Step 9: Loop & Repeat

Back to Step 3 until all steps complete.

### Step 10: Final Approval

When all steps = "complete":

```
Edit PLAN.md:
- status = "ready-for-review"

AskUserQuestion:
  - question: "All steps done. Finalize workflow?"
    options:
      - label: "Yes, finalize"
      - label: "No, review more"
```

**If Yes:** 
- Dispatch finalize subagent
- Create final commit
- Set PLAN.status = "complete"

**If No:**
- Loop back (allow user to request changes on any step)

---

## 5. Bootstrap Skill

**Creates:** PLAN.md and all step-N.md files

**Action:** User provides task description and step requirements
**Output:** 
- `.workflow/TASK_NAME/PLAN.md`
- `.workflow/TASK_NAME/steps/step-1.md`
- ... through step-N.md

---

## 6. Implementer Skill

**RUNS:** In isolated subagent (via Agent() from execute)

**Input:** Full procedure in Agent() prompt
**Output:** Updated step-N.md with status = "verification", code changes committed

**Process:**
1. Read step definition
2. Implement code changes
3. Run migrations if needed
4. Run tests
5. Commit
6. Update step file
7. Report back

---

## 7. Verifier Skill

**RUNS:** In isolated subagent (via Agent() from execute)

**Input:** Full procedure in Agent() prompt
**Output:** Updated step-N.md with status = "complete" or "needs-fix"

**Process:**
1. Read step criteria
2. Setup environment
3. Verify migrations ran (don't run them)
4. Run tests
5. Manually test criteria
6. Update step file with results
7. Report back

---

## 8. Finalize Skill

**RUNS:** In isolated subagent (via Agent() from execute)

**Input:** Full procedure in Agent() prompt
**Output:** Final commit, updated PLAN.md

**Process:**
1. Verify all steps complete
2. Gather all changes
3. Create comprehensive commit message
4. Commit changes
5. Update PLAN.md: status = "complete"
6. Generate documentation (optional)

---

## 9. Execute vs Implementer vs Verifier

| Component | Runs In | Job | Tools |
|-----------|---------|-----|-------|
| execute | Main session | Orchestrate, ask user, dispatch subagents | Read, Edit, AskUserQuestion, Agent |
| implementer | Subagent (via Agent) | Write code | Edit, Write, Bash (build/test/git) |
| verifier | Subagent (via Agent) | Test code | Read, Edit (for step file), Bash (test/curl) |

---

## 10. Critical Constraints

1. **Execute Never Does Work**
   - Never reads source files
   - Never runs tests
   - Never writes code
   - Only delegates via Agent()

2. **No Skill() in Subagent Prompts**
   - Implementer prompt includes full procedure inline
   - Verifier prompt includes full procedure inline
   - Never invoke Skill() in subagent context

3. **Agent() is the Gateway**
   - All work happens via Agent(subagent_type="general-purpose")
   - Agent() prompt is self-contained
   - Subagent never needs to read files or call skills

4. **Ask User Before Dispatch**
   - Execute asks AskUserQuestion() before ANY Agent() call
   - User confirms before work starts

5. **Migrations are Implementer's Job**
   - Implementer runs migrations
   - Verifier only checks they ran
   - Verifier fails if incomplete (doesn't fix)

6. **Verifier Tests, Doesn't Review**
   - Build/test execution, not code review
   - Actual curl calls, not "looks correct"
   - Manual testing of criteria, with proof

---

## 11. Success Criteria for This System

- [ ] User runs `/workflow:execute`
- [ ] Execute asks: "Start from step {N}?"
- [ ] User confirms
- [ ] Agent(implementer) dispatched and runs in isolated context
- [ ] Implementer runs tests, commits, updates step file
- [ ] Agent(verifier) dispatched and runs in isolated context
- [ ] Verifier runs tests, checks migrations, manually tests
- [ ] If PASS: step status = "complete"
- [ ] If FAIL: step status = "needs-fix", issues documented
- [ ] Execute loops back and dispatches next action
- [ ] No Skill() calls in subagent context
- [ ] No test output or build logs in main conversation
- [ ] All work tracked in step files
- [ ] User can see progress in PLAN.md and step-N.md

---

## 12. Error Cases

| Scenario | Action |
|----------|--------|
| Implementer: migration fails | FAIL step, return with error message |
| Implementer: tests fail | FAIL step, return which tests failed |
| Implementer: build fails | FAIL step, return build error |
| Verifier: migrations incomplete | FAIL step, don't run them (implementer must fix) |
| Verifier: test fails | FAIL step, document which test |
| Verifier: API returns 500 | FAIL step, show error response |
| Verifier: manual test fails | FAIL step, show what was tested and result |

All failures update step-N.md with Issues Identified, return to implementer.
