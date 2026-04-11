---
name: workflow:finalize
description: Use when a workflow is approved and ready to finalize - creates completion commit and marks workflow as complete
---

# Workflow Finalize Skill

Complete a workflow by creating final commit and marking it done. This skill runs in an isolated subagent.

## When to Use

Use this skill when:
- All steps are `complete` in step-N.md
- PLAN.md status is `ready-for-review`
- User has approved finalization
- execute dispatched this skill

## Input

From execute's Agent() prompt:
- Task name
- PLAN.md path
- Step files path

## Output

- Final commit created with comprehensive message
- PLAN.md updated: status = "complete", completed = today
- Workflow marked complete in git history

---

## YOUR PROCEDURE

### Step 1: Verify All Steps Complete

```
Read PLAN.md
Read each step-N.md

Verify: All steps have status = "complete"
If any step ≠ complete:
  → FAIL, report which step not ready
```

### Step 2: Gather Summary Info

```
From each step-N.md extract:
- Step name
- Files created/modified
- Implementation notes summary
- Verification results
- Iterations (if any fix cycles)

Build summary of:
- Total steps: {N}
- Total iterations: {count of fix cycles}
- Files changed: {count}
```

### Step 3: Create Commit Message

Generate comprehensive commit:

```
workflow: complete {TASK_NAME}

Summary:
- Task: {TASK_TITLE}
- Steps: {N} total, all verified ✓
- Mode: {1|2}
- Duration: {created} to {today}

## Changes by Step

### Step 1: {Name}
- Goal: {goal}
- Files: {list}
- Verified: ✓ PASS

### Step 2: {Name}
- Goal: {goal}
- Files: {list}
- Verified: ✓ PASS

[... for each step ...]

## Summary

- Total steps: {N}
- Total iterations: {count}
- Files created: {count}
- Files modified: {count}
- All verification criteria passing

Workflow execution complete.
Started: {date}
Completed: {date}
```

### Step 4: Create Final Commit

```
git add .workflow/  (state files)
git commit -m "[message from Step 3]"

git status  (verify clean)
```

### Step 5: Update PLAN.md

Edit PLAN.md:

```yaml
---
status: complete
completed: {TODAY}
---

## Completion

Workflow completed successfully on {DATE}.

All steps verified:
- Step 1: ✓ complete
- Step 2: ✓ complete
- ...
- Step N: ✓ complete

Total iterations: {count}
Workflow time: {created} to {completed}
```

### Step 6: Final Commit

```
git add PLAN.md
git commit -m "Mark workflow complete"
```

### Step 7: Report Back

Return summary to execute:

```
✓ Workflow {TASK_NAME} complete
  - {N} steps verified
  - {X} files changed
  - Final commit created
  - PLAN.md updated

Ready for review: git log --oneline -5
```

---

## Commit Message Format

Use this exact format:

```
workflow: complete {task-name}

Summary:
- Task: {human readable title}
- Steps: {N} total, all verified ✓
- Mode: {1|2} (Step-by-Step|End-to-End)
- Duration: {date} to {date}

[Details of each step]

## Summary Statistics

- Total steps completed: {N}
- Total iterations: {count of fix cycles}
- Files created: {count}
- Files modified: {count}
- Test pass rate: 100%

Workflow successfully completed.
```

---

## Critical Rules

**MUST DO:**
- ✅ Verify all steps = "complete"
- ✅ Gather info from all step files
- ✅ Create comprehensive commit message
- ✅ Commit workflow state files
- ✅ Update PLAN.md with completion date
- ✅ Create final commit
- ✅ Report back to execute

**NEVER DO:**
- ❌ Finalize if any step not complete
- ❌ Make generic commit message
- ❌ Forget to update PLAN.md
- ❌ Leave PLAN.md status as "ready-for-review"

---

## Success Criteria

Workflow finalized when:
- ✅ All steps verified as `complete`
- ✅ Comprehensive commit created
- ✅ PLAN.md marked complete with date
- ✅ Git log shows finalize commits
- ✅ status = "complete"
- ✅ Report provided to execute

---

## Example Output

```
✓ Workflow feature-auth-system complete

Commit summary:
- workflow: complete feature-auth-system
- 3 steps verified
- 8 files created, 12 modified
- 1 fix cycle (step 2)

PLAN.md updated:
- status: complete
- completed: 2026-04-11

Ready for next steps: review, merge, or deployment
```

---

## Post-Finalize

After this skill completes:
1. User can review workflow in git log
2. Workflow state saved in PLAN.md and step files
3. Ready for:
   - Code review
   - Merging to main
   - Release/deployment
   - Documentation generation
