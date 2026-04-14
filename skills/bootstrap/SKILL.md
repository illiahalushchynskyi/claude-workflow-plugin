---
name: workflow:bootstrap
description: Use when starting a new workflow task - creates PLAN.md and step files from task description and requirements
---

# Workflow Bootstrap Skill

Initialize a new workflow by creating PLAN.md and step definition files.

## When to Use

Use `workflow:bootstrap` when:
- You're starting a new workflow task
- User has provided task description and step requirements
- You need to create `.workflow/TASK_NAME/` directory structure

## Input

User provides:
1. **Task name** - identifier (e.g., `feature-auth-system`)
2. **Task title** - human readable (e.g., `Implement JWT Authentication`)
3. **Task description** - what the task accomplishes
4. **Step requirements** - list of steps with goals and criteria
5. **Mode** - 1 (Step-by-Step with approval) or 2 (End-to-End automated)

## Output

Creates:
- `.workflow/TASK_NAME/PLAN.md`
- `.workflow/TASK_NAME/steps/step-1.md`
- `.workflow/TASK_NAME/steps/step-2.md`
- ... through `step-N.md`

## Your Procedure

### Step 1: Validate Input

Ask user for missing information:
- Task name (lowercase, hyphens only)
- Task title
- Description
- Mode (1 or 2)
- List of steps with:
  - Step name
  - Goal
  - Acceptance criteria (at least 3 per step - these are what verifier will check)

Use AskUserQuestion if any field missing.

**Acceptance Criteria Format:**
Each criterion must be:
- **Specific** - not vague like "it works" or "feature is done"
- **Testable** - can verify with concrete proof (curl output, DB query, test output, etc.)
- **Independent** - can verify each criterion separately
- **Examples:**
  - ✓ GOOD: "API POST /users returns 201 status and user object with id field"
  - ✓ GOOD: "Database table contains inserted user record with correct name"
  - ✗ BAD: "User endpoint works correctly"
  - ✗ BAD: "Everything is done"

When user provides acceptance criteria, validate they meet this standard. Ask for clarification if criteria are too vague.

### Step 2: Create Directory Structure

```bash
mkdir -p .workflow/TASK_NAME/steps
```

### Step 3: Generate PLAN.md

Create `.workflow/TASK_NAME/PLAN.md`:

```yaml
---
name: {TASK_NAME}
title: {Task Title}
description: {Description}
mode: {1|2}
status: planning
created: {TODAY}
started: null
completed: null
---

# {Task Title}

{Description}

## Steps Overview

| Step | Name | Status | Iteration | Acceptance Criteria | Notes |
|------|------|--------|-----------|-------------------|-------|
| 1 | {Step 1 Name} | pending | 1 | See step-1.md | Initial |
| 2 | {Step 2 Name} | pending | 1 | See step-2.md | Initial |
| ... | ... | ... | ... | ... | ... |
| N | {Step N Name} | pending | 1 | See step-N.md | Initial |

## Execution Notes

- Mode: {1=Step-by-Step|2=End-to-End}
- Status: {planning|executing|ready-for-review|approved|complete}

**Note:** Acceptance Criteria column references the detailed criteria in each step file. Verifier uses these criteria to mark each as ✓ passed or ✗ not passed.
```

### Step 4: Generate step-N.md Files

For each step, create `.workflow/TASK_NAME/steps/step-N.md`:

```yaml
---
step_number: {N}
name: {Step Name}
status: pending
iteration: 1
created: {TODAY}
---

# Step {N}: {Step Name}

## Goal

{One sentence description of what this step accomplishes}

## Files to Modify/Create

- [To be determined by implementer]

## Acceptance Criteria

List each criterion the verifier will check:

- **Criterion 1:** {Specific, testable description}
- **Criterion 2:** {Specific, testable description}
- **Criterion 3:** {Specific, testable description}
- **Criterion 4+:** {Add more as needed}

**Format guidance:** Each criterion must be:
- Specific (not vague like "it works")
- Testable (can be verified with concrete evidence)
- Independent (can verify each separately)

Example: "✓ API endpoint POST /users returns 201 status and user object with id field"

## Implementation

### Files to Modify/Create

[To be filled by implementer]

### Implementation Notes

[To be filled by implementer]

## Verification

### Verification Notes

[To be filled by verifier]

### Issues Identified

[If FAIL - issues list]
```

### Step 5: Create .workflow-config.json (Optional)

Create `.workflow/TASK_NAME/.workflow-config.json`:

```json
{
  "taskName": "{TASK_NAME}",
  "projectType": "api|library|app|other",
  "commitStyle": "conventional",
  "push": false
}
```

### Step 5.5: Initialize progress.json

Create `.workflow/TASK_NAME/progress.json` to track detailed step execution state.

**Purpose:** progress.json is the source of truth for detailed step execution, timing, and approvals. While PLAN.md shows high-level status, progress.json tracks timestamps and iteration history.

**Before:**
```
No progress tracking file - workflow cannot track timing or iterative attempts
```

**After:**
```
.workflow/TASK_NAME/progress.json created with all steps in pending status
- Task initialized with creation date
- All steps ready for execution tracking
- Mode-specific approval structure initialized
```

**Implementation:**

Use this bash command to create progress.json. Build the steps object from the step files you just created:

```bash
# Build steps object from step files
STEPS_JSON="{"
for i in .workflow/TASK_NAME/steps/step-*.md; do
  if [ -f "$i" ]; then
    # Extract step number from filename (step-1.md -> 1)
    STEP_NUM=$(basename "$i" | sed 's/step-\([0-9]*\)\.md/\1/')
    # Extract step name from frontmatter
    STEP_NAME=$(grep "^name:" "$i" | head -1 | sed 's/^name: //' | sed "s/'//g" | sed 's/"//g')
    
    # Add to steps object
    if [ "$STEP_NUM" != "1" ]; then
      STEPS_JSON="$STEPS_JSON,"
    fi
    STEPS_JSON="$STEPS_JSON
    \"$STEP_NUM\": {
      \"name\": \"$STEP_NAME\",
      \"status\": \"pending\",
      \"iteration\": 1,
      \"implementation_start\": null,
      \"implementation_end\": null,
      \"verification_start\": null,
      \"verification_end\": null,
      \"approval_date\": null
    }"
  fi
done
STEPS_JSON="$STEPS_JSON
  }"

# Create progress.json
cat > .workflow/TASK_NAME/progress.json << 'PROGRESS_EOF'
{
  "task_name": "{TASK_NAME}",
  "mode": {MODE},
  "created": "$(date -u +%Y-%m-%d)",
  "started": null,
  "current_step": 1,
  "steps": STEPS_JSON_PLACEHOLDER,
  "approvals": {
    "mode_1_manual_approvals": []
  }
}
PROGRESS_EOF
```

**Alternative: Using jq for cleaner JSON generation:**

```bash
# Collect step names into array
STEP_NAMES=()
for i in .workflow/TASK_NAME/steps/step-*.md; do
  if [ -f "$i" ]; then
    STEP_NAME=$(grep "^name:" "$i" | head -1 | sed 's/^name: //' | sed "s/'//g" | sed 's/"//g')
    STEP_NAMES+=("$STEP_NAME")
  fi
done

# Create steps object with jq
STEPS_OBJ=$(
  jq -n \
    --argjson step_names "$(printf '%s\n' "${STEP_NAMES[@]}" | jq -R . | jq -s .)" \
    '
    $step_names | to_entries | map(
      {
        key: (.key + 1 | tostring),
        value: {
          name: .value,
          status: "pending",
          iteration: 1,
          implementation_start: null,
          implementation_end: null,
          verification_start: null,
          verification_end: null,
          approval_date: null
        }
      }
    ) | from_entries
    '
)

# Create complete progress.json
jq -n \
  --arg task_name "{TASK_NAME}" \
  --arg mode "{MODE}" \
  --arg created "$(date -u +%Y-%m-%d)" \
  --argjson steps "$STEPS_OBJ" \
  '{
    task_name: $task_name,
    mode: ($mode | tonumber),
    created: $created,
    started: null,
    current_step: 1,
    steps: $steps,
    approvals: {
      mode_1_manual_approvals: []
    }
  }' > .workflow/TASK_NAME/progress.json
```

**File Structure Reference:**

```json
{
  "task_name": "{TASK_NAME}",
  "mode": {1|2},
  "created": "{TODAY}",
  "started": null,
  "current_step": 1,
  "steps": {
    "1": {
      "name": "{Step 1 Name}",
      "status": "pending",
      "iteration": 1,
      "implementation_start": null,
      "implementation_end": null,
      "verification_start": null,
      "verification_end": null,
      "approval_date": null
    },
    "2": {
      "name": "{Step 2 Name}",
      "status": "pending",
      "iteration": 1,
      "implementation_start": null,
      "implementation_end": null,
      "verification_start": null,
      "verification_end": null,
      "approval_date": null
    }
  },
  "approvals": {
    "mode_1_manual_approvals": []
  }
}
```

**Field Definitions:**
- `task_name` (string) - Task identifier, matches directory name
- `mode` (number) - Execution mode: 1 for Step-by-Step (manual approval), 2 for End-to-End (automated)
- `created` (string) - Task creation date in YYYY-MM-DD format
- `started` (string|null) - Workflow execution start date, null until first step begins
- `current_step` (number) - Currently active step number
- `steps` (object) - Tracks each step's execution state:
  - `name` (string) - Step name from step-N.md frontmatter
  - `status` (string) - One of: `pending`, `implementation`, `verification`, `needs-fix`, `complete`
  - `iteration` (number) - Current iteration number (increments when step is retried)
  - `implementation_start` (string|null) - ISO 8601 timestamp when implementer started
  - `implementation_end` (string|null) - ISO 8601 timestamp when implementer finished
  - `verification_start` (string|null) - ISO 8601 timestamp when verifier started
  - `verification_end` (string|null) - ISO 8601 timestamp when verification completed
  - `approval_date` (string|null) - ISO 8601 timestamp when manually approved (Mode 1 only)
- `approvals.mode_1_manual_approvals` (array) - History of user approvals for Mode 1 workflows, each entry includes step number and approval timestamp

**Key Points:**
- All steps initialized to `pending` status with iteration 1
- All timestamps are null at initialization (populated during execution)
- `current_step` always points to the step being worked on
- For Mode 1 workflows, manual approvals are recorded in the approvals array
- This file is updated by workflow:execute, workflow:implementer, and workflow:verifier agents
- PLAN.md shows only the completed/current state; progress.json shows full history

### Step 6: Verify Files Created

Check:
- [ ] PLAN.md exists and is valid YAML with all required fields
- [ ] All step-N.md files exist (step-1.md through step-N.md)
- [ ] All step files have correct YAML frontmatter and content structure
- [ ] PLAN.md status fields are correct (should show "pending" for all steps)
- [ ] progress.json exists and is valid JSON
- [ ] progress.json has all steps initialized to "pending" status
- [ ] progress.json started field is null, created field has today's date
- [ ] progress.json mode matches the specified mode (1 or 2)
- [ ] .workflow-config.json exists with correct project metadata

### Step 7: Report to User

Show:
- Task structure created with workflow directory
- Files generated:
  - ✓ PLAN.md created
  - ✓ progress.json initialized
  - ✓ step-*.md files created
  - ✓ .workflow-config.json created
- All steps initialized to pending status
- Ready for execution message
- Next step: `/workflow:execute`

**Sample Output:**
```
✅ Workflow initialized: {TASK_NAME}

Files created:
  ✓ .workflow/{TASK_NAME}/PLAN.md
  ✓ .workflow/{TASK_NAME}/progress.json (all steps pending)
  ✓ .workflow/{TASK_NAME}/steps/step-1.md
  ✓ .workflow/{TASK_NAME}/steps/step-2.md
  ...
  ✓ .workflow/{TASK_NAME}/steps/step-N.md
  ✓ .workflow/{TASK_NAME}/.workflow-config.json

Status: Ready to execute
Mode: {1=Step-by-Step with approval|2=End-to-End automated}

Next: Use /workflow:execute to begin implementation
```

## Example

**Input:**
```
Task: feature-auth
Title: Implement JWT Authentication
Mode: 1
Steps:
1. Create JWT token service
   - Criteria: Token generated, verified, has expiry
2. Add authentication middleware
   - Criteria: Protected routes require token, 401 on missing
3. Add login endpoint
   - Criteria: Endpoint exists, validates credentials, returns token
```

**Output:**
```
✅ Workflow created: feature-auth
Files:
  .workflow/feature-auth/PLAN.md
  .workflow/feature-auth/steps/step-1.md
  .workflow/feature-auth/steps/step-2.md
  .workflow/feature-auth/steps/step-3.md

Ready to execute: /workflow:execute
```

## Error Handling

- **Missing input** → Ask for clarification via AskUserQuestion
- **Invalid task name** → Suggest lowercase/hyphens format
- **File creation fails** → Report error with details
- **Invalid YAML** → Regenerate with proper formatting

## Notes

- All dates use YYYY-MM-DD format
- Step names should be clear and descriptive
- Criteria should be testable and measurable
- Mode 1 requires user approval after each step
- Mode 2 runs steps automatically (verifier only gates)
