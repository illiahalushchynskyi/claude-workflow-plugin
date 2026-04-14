---
name: workflow:implementer
description: SUBAGENT ONLY - Use when a workflow step is in implementation phase, dispatched via workflow:execute skill
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Implementer Skill

⚠️ **CRITICAL: SUBAGENT ONLY - DO NOT CALL DIRECTLY**

**This skill runs in isolated subagent context and MUST be dispatched by workflow:execute.**

**If you are the main orchestrator:**
- ❌ DO NOT call this skill directly
- ✅ Call `workflow:execute` instead
- ✅ `workflow:execute` will dispatch this skill to an isolated subagent

**If you are a subagent:**
- ✅ You were correctly dispatched by execute
- ✅ Follow the procedure below
- ✅ You own the implementation work

---

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

### Step 5: Update Step File (Implementation Section Only)

**Do NOT update step status** - execute skill manages all status transitions.

Only update the Implementation section with your work:

Update {TASK_DIR}/steps/step-{N}.md **Implementation section only**:

```markdown
## Implementation

### Files to Modify/Create
- [List actual files created/modified]

### Implementation Notes

**Implementer**: Claude - {DATE}
- [Summary of changes made]
- [Any architectural decisions]
- [Test results: X tests passing]
- [Anything noteworthy for verifier]
```

Do NOT modify frontmatter - execute skill updates status.

### Step 6: Report Back

Return to execute with brief summary:

```
✓ Step {N} implementation complete
- Implemented [feature description]
- Files: {count} created, {count} modified
- Migrations: {yes/no}
- Tests: {count}/{total} passing

Ready for verification.
```

Status is managed by execute, not reported here.

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
- {TASK_DIR}/steps/step-{N}.md (update Implementation section ONLY)

**DO NOT modify:**
- Step frontmatter status field (execute manages this)
- progress.json (execute manages this)
- PLAN.md (execute manages this)
- Other step files (only current step)

---

## Critical Requirements

**MUST:**
- ✅ Understand goal before coding
- ✅ Make all code changes
- ✅ Run `npm test` - all must pass
- ✅ If ORM models changed: run migrations
- ✅ Commit changes with clear message
- ✅ Update step file: Implementation section only
- ✅ Fill Implementation section with details
- ✅ Return brief summary

**NEVER:**
- ❌ Skip tests
- ❌ Skip migrations if needed
- ❌ Update step status field (execute manages this)
- ❌ Update progress.json (execute manages this)
- ❌ Report done if tests fail
- ❌ Commit broken code

---

## Success = All This Done

Step complete when:
- ✅ Code changes made
- ✅ Tests passing (100%)
- ✅ Migrations ran (if needed)
- ✅ Changes committed
- ✅ Implementation section updated
- ✅ Changes committed to git
- ✅ Ready for verification
