---
name: workflow:bootstrap
description: Use when starting a new workflow task - creates PLAN.md and step files from task description and requirements
---

# Workflow Bootstrap Skill

Initialize a new workflow by creating:
- **PLAN.md** - workflow metadata and status
- **progress.json** - detailed step state tracking
- **steps/step-N.md** - individual step definitions
- **.workflow-config.json** - project metadata

## When to Use

Use `workflow:bootstrap` when starting a new workflow task with user-provided task description, step requirements, and mode selection.

## Input

User provides: task name, task title, description, mode (1 or 2), and step list with acceptance criteria.

## Output

Creates `.workflow/TASK_NAME/` directory with PLAN.md, progress.json, step files, and config.

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

Create `.workflow/TASK_NAME/PLAN.md` as human-facing summary only:

```yaml
---
status: bootstrap
mode: {MODE}
created: 2026-04-14
started: null
completed: null
---

# {Task Title}

{Task description}
```

**Status transitions:** `bootstrap` → `in-progress` → `ready-for-review` → `approved` → `complete`

**CRITICAL:** Do NOT add step lists or progress counters. All tracking goes in progress.json.

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

### Step 5: Detect Project Type

Detect the project's programming language/build system by checking for these files (in order):

1. `package.json` → `node`
2. `pyproject.toml` or `requirements.txt` → `python`
3. `Cargo.toml` → `rust`
4. `go.mod` → `go`
5. `pom.xml` → `java-maven`
6. `build.gradle` or `build.gradle.kts` → `java-gradle`
7. `Gemfile` → `ruby`
8. `CMakeLists.txt` → `cpp-cmake`
9. `Makefile` → `generic-make`
10. If none found → `other` (ask user for build/test commands)

Load the detected project type's commands from `skills/templates/commands.yaml`. If detection fails or commands are missing, ask user to provide:
- Build command (e.g., `cargo build`)
- Test command (e.g., `cargo test`)
- Optional: Migrate command (e.g., `sqlx migrate run`)

### Step 5.5: Create .workflow-config.json

Create `.workflow/TASK_NAME/.workflow-config.json`:

```json
{
  "taskName": "{TASK_NAME}",
  "projectType": "{DETECTED_TYPE}",
  "buildCommand": "{BUILD_COMMAND}",
  "testCommand": "{TEST_COMMAND}",
  "migrateCommand": null,
  "commitStyle": "conventional",
  "push": false
}
```

**Field Definitions:**
- `projectType` (string) - Detected or user-specified language/framework (node, python, rust, go, java-maven, java-gradle, ruby, cpp-cmake, generic-make, other)
- `buildCommand` (string) - Command to build the project (e.g., `npm run build`, `cargo build --release`)
- `testCommand` (string) - Command to run tests (e.g., `npm test`, `pytest`)
- `migrateCommand` (string|null) - Command to run migrations, if applicable (e.g., `npm run migrate`, `sqlx migrate run`)

These commands are used by implementer and verifier subagents instead of assuming npm/node.

### Step 6: Initialize progress.json

Create `.workflow/TASK_NAME/progress.json` for detailed step execution tracking (timestamps, iterations, approvals). Build steps from the step files you just created.

**Structure:**

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
- `approvals.mode_1_manual_approvals` (array) - History of user approvals for Mode 1 workflows

**Key Points:**
- All steps initialized to `pending` status, iteration 1
- All timestamps are null at initialization
- `current_step` points to the step being worked on
- PLAN.md is human-facing summary; progress.json is system source of truth
- This file is updated by workflow:execute, workflow:implementer, and workflow:verifier

### Step 7: Verify Files Created

Verify:
- [ ] PLAN.md exists with correct YAML frontmatter (status, mode, created, started: null, completed: null)
- [ ] All step-N.md files exist with correct frontmatter
- [ ] progress.json is valid JSON with all steps at "pending" status
- [ ] .workflow-config.json exists with projectType, buildCommand, testCommand, migrateCommand fields
- [ ] buildCommand and testCommand are valid for detected projectType

### Step 8: Report to User

Report files created (PLAN.md, progress.json, steps/step-*.md, .workflow-config.json), status as "Ready to execute", and next step: `/workflow:execute`

## Example

Input: `feature-auth` with 3 steps and Mode 1

Output: `.workflow/feature-auth/` with PLAN.md, progress.json, steps/step-1.md through step-3.md, and .workflow-config.json

Ready to execute: `/workflow:execute`

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
