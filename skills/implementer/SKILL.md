---
name: workflow:implementer
description: Use when a workflow step is in implementation phase - executes code changes and updates step tracking
---

# Workflow Implementer Skill

Implement a workflow step. You are a subagent dispatched by execute.

**This skill runs in isolated subagent context.** Execute passed you task directory and mode. YOU own the work.

---

## YOUR FULL PROCEDURE (Execute in order)

### Step 1: Read & Understand

Get task directory from execute's prompt (e.g., `.workflow/feature-auth/`)

```
Read: {TASK_DIR}/steps/step-{N}.md

Extract:
- Goal: what to accomplish
- Verification Criteria: success conditions
- Files to Modify/Create: what to change
- Issues Identified: if this is needs-fix cycle

Read: {TASK_DIR}/PLAN.md
- Mode (1|2)
- Current step number
```

### Step 2: Implement Code Changes

```
Make code changes:
- Use Edit tool for existing files
- Use Write tool for new files
- Follow project conventions
- Keep changes focused to this step

If step modifies ORM models:
  → Create migration files if needed
  → Run MANDATORY: npm run migrate
  → Verify schema: psql -c "\dt"
```

### Step 3: Test Locally

```
MANDATORY: npm test

If tests fail:
  → Fix code or tests
  → Re-run until all pass
  → Document what was wrong

Example passing:
  45 tests passing
  0 failures
  Coverage: 92%
```

### Step 4: Commit Changes

```
git commit -m "Implement step {N}: {Goal}"

Include in message:
- What was implemented
- Files changed
- If migrations were run
```

### Step 5: Update Step File

Edit {TASK_DIR}/steps/step-{N}.md:

```yaml
---
status: verification
iteration: {SAME or INCREMENTED}
---

## Implementation

### Files Modified/Created
- file1.ts
- file2.ts
- migrations/001_add_users.sql

### Implementation Notes

**Implementer**: Claude - {DATE}
- Created [feature] using [approach]
- Ran migrations: [yes/no]
- Tests passing: [count/total]
- [Any notes for verifier]
```

### Step 6: Report Back

Return to execute with brief summary (under 100 words):

```
✓ Step {N} complete
- Implemented {goal}
- Files: {count} created, {count} modified
- Migrations: {yes/no}
- Tests: {count} passing
- Status: verification

Ready for verification.
```

---

## Decision Points During Work

**Tests fail?**
- Fix the code or test
- Re-run `npm test`
- When all pass, continue

**Migration fails?**
- Check migration file syntax
- Fix the migration
- Run `npm run migrate` again
- Verify with `psql -c "\dt"`

**Build fails?**
- Fix the error
- `npm run build` again
- No progress until success

**Uncertain about approach?**
- Complete what you can
- Document uncertainty in step file
- Report: "Step complete but uncertain about [X]"
- Let verifier review

**Blocked/Can't proceed?**
- Document exact blocker
- Report: "BLOCKED: [reason]"
- Let execute handle escalation

---

## Files You Work With

**Read:**
- {TASK_DIR}/steps/step-{N}.md (task definition)
- {TASK_DIR}/PLAN.md (context)
- Project source files (whatever you need)

**Write/Edit:**
- Project source files
- Migration files
- {TASK_DIR}/steps/step-{N}.md (update Implementation section + status)

**DO NOT modify:**
- PLAN.md (except updater)
- Other step files (only current step)

---

## Critical Requirements

**MUST:**
- ✅ Understand goal before coding
- ✅ Make all code changes
- ✅ Run `npm test` - all must pass
- ✅ If ORM models changed: run migrations
- ✅ Commit changes with clear message
- ✅ Update step file: status = "verification"
- ✅ Fill Implementation section
- ✅ Return brief summary

**NEVER:**
- ❌ Skip tests
- ❌ Skip migrations if needed
- ❌ Leave step file unchanged
- ❌ Report done if tests fail
- ❌ Commit broken code

---

## Success = All This Done

Step complete when:
- ✅ Code changes made
- ✅ Tests passing (100%)
- ✅ Migrations ran (if needed)
- ✅ Changes committed
- ✅ step-N.md status = "verification"
- ✅ Implementation section filled
- ✅ Summary reported
