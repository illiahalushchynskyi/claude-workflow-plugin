---
name: workflow:finalize
description: Finalize workflow - create final commit, mark complete (runs in main session)
---

# Workflow Finalize

**This skill runs in MAIN SESSION, invoked directly by execute**

Execute calls this skill after all steps verified complete and human approves finalization.

## Entry Conditions - GUARD

| Condition | Must Be True |
|---|---|
| All steps status == "complete" | ✅ YES (required) |
| Mode selected (1 or 2) | ✅ YES (required) |
| If Mode 1: User approved finalization | ✅ YES (required) |
| If Mode 2: All steps verified complete | ✅ YES (required) |
| progress.json exists | ✅ YES (required) |
| PLAN.md exists | ✅ YES (required) |

If ANY condition false: DO NOT PROCEED—report error to user

## When to Use (By Mode)

**Mode 1 (Step Manual Approve):**
- All steps have status = "complete"
- User approved finalization after last step

**Mode 2 (Final Approve):**
- All steps have status = "complete"
- Execute is ready to finalize

## Procedure - Step-by-Step

### Step 1: Verify All Steps Complete

**ACTION:** Check that all steps in progress.json have status = "complete"

**PROCEDURE:**
```bash
# Load progress.json
if [ ! -f ".workflow/${TASK_NAME}/progress.json" ]; then
  ERROR: "progress.json not found at .workflow/${TASK_NAME}/"
  exit 1
fi

# Count complete steps
COMPLETE=$(jq '.steps | map(select(.status == "complete")) | length' \
  .workflow/${TASK_NAME}/progress.json)
TOTAL=$(jq '.steps | length' .workflow/${TASK_NAME}/progress.json)

# Verify all complete
if [ "$COMPLETE" -ne "$TOTAL" ]; then
  ERROR: "$COMPLETE of $TOTAL steps complete. Cannot finalize yet."
  Show: which steps not complete: $(jq '.steps | to_entries[] | select(.value.status != "complete")')
  exit 1
fi

# Get task name
TASK=$(jq -r '.task_name' .workflow/${TASK_NAME}/progress.json)
echo "✓ Verified: All $TOTAL steps status = complete"
```

**SUCCESS SIGNAL:** All steps verified complete

### Step 2: Update progress.json - Set Completion Timestamp

**ACTION:** Add completion timestamp to progress.json

**PROCEDURE:**
```bash
COMPLETION_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

jq ".completed = \"${COMPLETION_TIME}\"" \
  .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
  mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json

echo "✓ Set completed timestamp: ${COMPLETION_TIME}"
```

**VERIFY:** progress.json now has `"completed": "{ISO8601}"`

### Step 3: Update PLAN.md Status

**ACTION:** Update PLAN.md frontmatter status field only

**CURRENT STATE:**
```yaml
---
status: in-progress
created: {ORIGINAL_DATE}
---
```

**CHANGE TO:**
```yaml
---
status: completed
created: {ORIGINAL_DATE}
---
```

**IMPORTANT:**
- ✅ Change ONLY the `status` field
- ✅ Keep `created` date unchanged
- ✅ Keep description and content below frontmatter
- ❌ Do NOT add any new fields
- ❌ Do NOT change any other frontmatter fields

**ADD TO BODY:** Single line after frontmatter:
```
Workflow completed on ${COMPLETION_TIME}. See progress.json for detailed results.
```

### Step 4: Create Final Commit

**ACTION:** Commit the completed workflow state

**PROCEDURE:**
```bash
git add .workflow/${TASK_NAME}/PLAN.md .workflow/${TASK_NAME}/progress.json

git commit -m "workflow: finalize ${TASK}

All ${TOTAL} steps verified and completed.

Final timestamp: ${COMPLETION_TIME}
See progress.json for detailed step tracking and results."
```

**VERIFY:**
```bash
git log --oneline -1  # Shows: "workflow: finalize {task}"
git show HEAD --stat  # Shows modified: PLAN.md, progress.json
```

### Step 5: Report Completion

**ACTION:** Inform user workflow is finalized

**OUTPUT:**
```
✓ Workflow ${TASK} finalized successfully

Summary:
- Total steps: ${TOTAL}
- Status: All verified and complete
- Completion time: ${COMPLETION_TIME}
- Final commit: $(git rev-parse --short HEAD)
- PLAN.md status: ✓ marked as completed
- progress.json: ✓ timestamped

Next: Ready for review, deployment, or archive
```

---

## Completion Checklist

Before reporting success, verify:

```
✅ All steps verified complete in progress.json
✅ progress.json.completed field has ISO8601 timestamp
✅ PLAN.md status field changed to "completed"
✅ PLAN.md created field unchanged
✅ PLAN.md body has completion message
✅ git commit created with message containing "workflow: finalize"
✅ git log shows both PLAN.md and progress.json committed
✅ No uncommitted changes in .workflow/{TASK_NAME}/
```

If ANY item unchecked: RETURN to that step, fix it, re-verify

## Success Criteria

Finalize is successful when:

| Criteria | Check |
|---|---|
| **progress.json** | `completed` field has ISO8601 timestamp (not null) |
| **progress.json** | All steps have `status: "complete"` |
| **progress.json** | No syntax errors (valid JSON) |
| **PLAN.md** | Frontmatter `status: completed` |
| **PLAN.md** | Frontmatter `created` unchanged |
| **PLAN.md** | No extra frontmatter fields added |
| **Git** | Final commit with message containing "workflow: finalize" |
| **Git** | Both .workflow/{TASK}/PLAN.md and progress.json in final commit |
| **Git** | `git status` shows clean working tree |

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| "Not all steps complete" | Some step still pending/needs-fix | Return to execute, complete remaining steps |
| progress.json syntax error | Invalid JSON after edit | Fix JSON syntax, retry finalize |
| PLAN.md not found | Wrong task name | Verify .workflow/{TASK_NAME}/PLAN.md exists |
| Git commit fails | Git not configured or hook fails | Check git status, fix hooks if needed |

## Final State

After finalize completes:
- ✅ Workflow status = completed (in progress.json)
- ✅ PLAN.md status = completed
- ✅ All timestamps recorded
- ✅ Final commit in git history
- ✅ Ready for review or deployment
