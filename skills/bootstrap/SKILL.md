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

### Step 6: Verify Files Created

Check:
- [ ] PLAN.md exists and is valid YAML
- [ ] All step-N.md files exist
- [ ] All files have correct structure
- [ ] Status fields are correct

### Step 7: Report to User

Show:
- Task structure created
- All files generated
- Ready for execution message
- Next step: `/workflow:execute`

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
