---
name: workflow:finalize
description: Finalize workflow - create final commit, mark complete (runs in main session)
---

# Workflow Finalize

**This skill runs in MAIN SESSION, invoked directly by execute**

Execute calls this skill after all steps verified complete and human approves finalization.

## When to Use

- All steps complete in progress.json
- PLAN.md status = "ready-for-review"
- User has approved finalization

## Your Procedure

### Step 1: Verify All Steps Complete

```bash
# Load progress.json and verify all steps are complete
if [ ! -f ".workflow/${TASK_NAME}/progress.json" ]; then
  echo "ERROR: progress.json not found"; exit 1
fi

COMPLETE=$(jq '.steps | map(select(.status == "complete")) | length' .workflow/${TASK_NAME}/progress.json)
TOTAL=$(jq '.steps | length' .workflow/${TASK_NAME}/progress.json)

if [ "$COMPLETE" -ne "$TOTAL" ]; then
  echo "ERROR: Not all steps complete"; exit 1
fi

# Load task info
TASK=$(jq -r '.task_name' .workflow/${TASK_NAME}/progress.json)
echo "✓ All $TOTAL steps verified complete"
```

### Step 2: Update progress.json with Completion Timestamp

```bash
jq ".completed = \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"" \
  .workflow/${TASK_NAME}/progress.json > .workflow/${TASK_NAME}/progress.json.tmp && \
  mv .workflow/${TASK_NAME}/progress.json.tmp .workflow/${TASK_NAME}/progress.json
```

### Step 3: Update PLAN.md Status

Edit `.workflow/${TASK_NAME}/PLAN.md` frontmatter:

```yaml
---
status: complete
completed: {TODAY}
---
```

Add single line to body: `Workflow completed {DATE}. See progress.json for details.`

### Step 4: Create Final Commit

```bash
git add .workflow/${TASK_NAME}/PLAN.md .workflow/${TASK_NAME}/progress.json

git commit -m "workflow: complete ${TASK}

All ${TOTAL} steps verified and completed.

See progress.json for detailed step tracking and timestamps."
```

### Step 5: Report Completion

Output to user:

```
✓ Workflow ${TASK} finalized

- All ${TOTAL} steps complete
- Completion date set
- Final commit created
- PLAN.md marked complete

Ready for review/merge/deployment
```

---

## Success Criteria

Finalize complete when:
- ✅ progress.json has completion timestamp
- ✅ PLAN.md status = "complete"
- ✅ Final commit in git log
- ✅ All workflow files clean in git
