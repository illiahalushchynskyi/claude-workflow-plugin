---
name: workflow:implementer
description: Use when a workflow step is in implementation phase - executes code changes and updates step tracking
---

# Workflow Implementer Skill

Implement a workflow step. This skill runs in an isolated subagent dispatched by execute.

## When to Use

Use this skill when:
- You are a subagent dispatched by workflow:execute
- Step status is `pending`, `implementation`, or `needs-fix`
- execute provided full task context in the prompt

## Input

From execute's Agent() prompt:
- Step file path
- PLAN.md path
- Task name and mode
- Current iteration number
- Issues (if needs-fix cycle)

## Output

- Code changes committed to project
- step-N.md updated with:
  - status = "verification"
  - Implementation section filled
  - Iteration incremented (if fix cycle)
- Brief report returned to execute

---

## YOUR PROCEDURE

### Step 1: Read & Understand

```
Read step file to understand:
- Goal: what this step accomplishes
- Verification Criteria: success conditions
- Previous Implementation Notes: what was tried
- Issues Identified: what needs fixing (if needs-fix)
```

### Step 2: Implement Changes

```
Make code changes:
- Create new files if needed
- Modify existing files
- Follow project conventions
- Use Edit/Write tools

If ORM models changed:
  - Create migration files
  - Run: npm run migrate (MANDATORY)
  - Verify schema with: psql -c "\dt"
```

### Step 3: Test Locally

```
Run tests: npm test
- All tests MUST pass
- If any fail: FIX them before continuing
- Document test results
```

### Step 4: Commit Changes

```
git commit -m "Implement step {N}: {Goal}"
- Include files changed
- Clear commit message
```

### Step 5: Update Step File

Edit step-N.md:

```yaml
---
status: verification
iteration: {SAME or INCREMENTED if fix cycle}
---

## Implementation

### Files Modified/Created
- File 1
- File 2

### Implementation Notes
**Implementer**: Claude - {DATE}
- {Decision made}
- {Blocker if any}
- {Test results}
- {Notes for verifier}
```

### Step 6: Report Back

Return brief summary (under 100 words):
- What you implemented
- Files changed
- Test results
- Any concerns

Example:
```
Implemented JWT token service:
- Created TokenService with sign/verify methods
- Added type definitions
- 8 unit tests passing
- All imports working
- Runs migrations successfully

Status: verification
Iteration: 1
```

---

## Decision Points

**Tests fail?**
- Fix the test or code
- Re-run until all pass
- Document what was wrong

**Migration fails?**
- Fix the migration file
- Run again
- Verify schema applied

**Build fails?**
- Fix the error
- Rebuild
- Ensure success before moving on

**Uncertain about approach?**
- Report status: DONE_WITH_CONCERNS
- List what you're uncertain about
- Let verifier review

**Totally blocked?**
- Report status: BLOCKED
- Describe exact blocker
- Let execute handle escalation

---

## Critical Requirements

**MUST DO:**
- ✅ Run migrations if ORM models changed
- ✅ Run all tests - they must pass
- ✅ Build project - it must compile
- ✅ Commit changes with clear message
- ✅ Update step file: status = "verification"
- ✅ Fill Implementation section with details

**NEVER DO:**
- ❌ Skip migrations
- ❌ Say tests pass if they don't
- ❌ Commit broken code
- ❌ Leave step file unchanged
- ❌ Report done if tests fail

---

## File Structure

Files you'll need:
- `.workflow/{TASK_NAME}/steps/step-{N}.md` - Update this
- `.workflow/{TASK_NAME}/PLAN.md` - Reference only
- Project files - Edit/Write as needed
- Migration files - Create if needed

---

## Success Criteria

Step complete when:
- ✅ Code changes made
- ✅ All tests passing
- ✅ Migrations ran (if needed)
- ✅ Changes committed
- ✅ step-N.md updated with status="verification"
- ✅ Implementation section filled
- ✅ Report provided to execute
