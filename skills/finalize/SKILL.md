---
name: workflow:finalize
description: SUBAGENT ONLY - Use when a workflow is approved and ready to finalize, dispatched via workflow:execute skill
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Finalize Skill

⚠️ **CRITICAL: SUBAGENT ONLY - DO NOT CALL DIRECTLY**

**This skill runs in isolated subagent context and MUST be dispatched by workflow:execute.**

**If you are the main orchestrator:**
- ❌ DO NOT call this skill directly
- ✅ Call `workflow:execute` instead
- ✅ `workflow:execute` will dispatch this skill to an isolated subagent

**If you are a subagent:**
- ✅ You were correctly dispatched by execute
- ✅ Follow the procedure below
- ✅ You own the finalization work

---

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
- progress.json path
- Step files path

## Output

- Final commit created with comprehensive message
- PLAN.md updated: status = "complete", completed = today
- Workflow marked complete in git history

---

## YOUR PROCEDURE

### Step 1: Load Progress State

Read progress.json to verify all steps are complete and gather summary:

```bash
# Check progress.json exists
if [ ! -f ".workflow/${TASK_NAME}/progress.json" ]; then
  FAIL: "progress.json not found"
fi

# Verify all steps are complete
COMPLETE_STEPS=$(jq '.steps | map(select(.status == "complete")) | length' .workflow/${TASK_NAME}/progress.json)
TOTAL_STEPS=$(jq '.steps | length' .workflow/${TASK_NAME}/progress.json)

if [ "$COMPLETE_STEPS" -ne "$TOTAL_STEPS" ]; then
  echo "Not all steps complete:"
  jq '.steps[] | select(.status != "complete") | {name, status}' .workflow/${TASK_NAME}/progress.json
  FAIL: "Cannot finalize until all steps are complete"
fi

echo "✓ All $TOTAL_STEPS steps are complete"

# Load metadata
TASK_NAME=$(jq -r '.task_name' .workflow/${TASK_NAME}/progress.json)
MODE=$(jq -r '.mode' .workflow/${TASK_NAME}/progress.json)
CREATED=$(jq -r '.created' .workflow/${TASK_NAME}/progress.json)

echo "Workflow: $TASK_NAME"
echo "Mode: $MODE"
echo "Created: $CREATED"
```

Then gather step information from progress.json:

```bash
# Extract step information from progress.json
jq '.steps | to_entries[] | 
  {
    step: .key,
    name: .value.name,
    status: .value.status,
    iteration: .value.iteration,
    approval_date: .value.approval_date
  }' .workflow/${TASK_NAME}/progress.json > steps-summary.json

echo "Step Summary:"
jq '.' steps-summary.json
```

### Step 2: Generate Commit Summary

Use progress.json to generate the comprehensive commit message:
- Each completed step with its iteration count
- Approval dates for Mode 1 workflows
- Total iterations (sum of all step iterations)

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

### Step 4B: Update progress.json Completion Date

Set completion timestamp in progress.json:

```bash
jq ".completed = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
  .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
  mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json

echo "✓ Completion date set in progress.json"
```

Then continue with updating PLAN.md as before.

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

Report workflow completion to user:

```
✓ Workflow {TASK_NAME} complete

Summary:
  - All {N} steps verified and completed
  - Total iterations: {count} (fix cycles if any)
  - Completed: {date}

Files updated:
  - PLAN.md: status = complete
  - progress.json: completion date set
  - .workflow/{TASK_NAME}/: all state files finalized

Next steps: review, merge, or deployment
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
- ✅ Load progress.json and verify all steps = "complete"
- ✅ Gather info from progress.json (not by scanning step files)
- ✅ Create comprehensive commit message
- ✅ Commit workflow state files
- ✅ Set completion date in progress.json
- ✅ Update PLAN.md with completion date
- ✅ Create final commits
- ✅ Report back to execute

**NEVER DO:**
- ❌ Finalize if any step not complete
- ❌ Scan step files manually to gather information
- ❌ Make generic commit message
- ❌ Forget to update progress.json completion date
- ❌ Forget to update PLAN.md
- ❌ Leave PLAN.md status as "ready-for-review"

---

## Success Criteria

Workflow finalized when:
- ✅ progress.json exists and verified all steps = "complete"
- ✅ All steps verified as `complete` in progress.json
- ✅ Comprehensive commit created with progress.json data
- ✅ progress.json updated with completion date
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

## Error Handling

| Error | Resolution |
|-------|-----------|
| progress.json not found | Run bootstrap first, then execute to create workflows |
| Not all steps complete | Check progress.json, verify all steps have status = "complete" |
| Invalid progress.json | Check JSON syntax, may have been corrupted |
| PLAN.md missing | Ensure PLAN.md exists in workflow directory |
| Git commit fails | Check git status, unstaged changes, or permission issues |
| jq command fails | Verify jq is installed and progress.json is valid JSON |

Note: Step information is now loaded from progress.json instead of scanning individual step files. This ensures consistency and prevents missing or incomplete data.

---

## Post-Finalize

After this skill completes:
1. User can review workflow in git log
2. Workflow state saved in PLAN.md, progress.json, and step files
3. Ready for:
   - Code review
   - Merging to main
   - Release/deployment
   - Documentation generation
