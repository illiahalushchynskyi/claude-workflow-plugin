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

Use `workflow:bootstrap` when starting a new workflow task with user-provided task description and step requirements.

## Input

User provides: task name, task title, description, and step list with verification criteria.
**NOTE:** Mode is NOT selected here—it will be chosen by user when running `/workflow:execute`

## Output

Creates `.workflow/TASK_NAME/` directory with PLAN.md, progress.json, step files, and config.

## Your Procedure

### Step 1: Validate Input

Ask user for missing information:
- Task name (lowercase, hyphens only)
- Task title
- Description
- List of steps with:
  - Step name
  - Goal
  - Verification criteria (at least 3 per step - these are what verifier will check)

Use AskUserQuestion if any field missing.

**Verification Criteria Format:**
Each criterion must be:
- **Specific** - not vague like "it works" or "feature is done"
- **Testable** - can verify with concrete proof (curl output, DB query, test output, etc.)
- **Independent** - can verify each criterion separately
- **Examples:**
  - ✓ GOOD: "API POST /users returns 201 status and user object with id field"
  - ✓ GOOD: "Database table contains inserted user record with correct name"
  - ✗ BAD: "User endpoint works correctly"
  - ✗ BAD: "Everything is done"

When user provides verification criteria, validate they meet this standard. Ask for clarification if criteria are too vague.

### Step 2: Create Directory Structure

```bash
mkdir -p .workflow/TASK_NAME/steps
```

### Step 3: Generate PLAN.md

Create `.workflow/TASK_NAME/PLAN.md` as human-facing summary only:

```yaml
---
status: pending
created: 2026-04-14
---

# {Task Title}

{Task description}

## Workflow Execution

**Mode is selected at runtime when you run `/workflow:execute`**
- Step Manual Approve - approve each step after verification
- Final Approve - approve once at the end

See `progress.json` for detailed execution status and mode.
```

**PLAN.md Status Values:**
- `pending` - Not started (corresponds to workflow_status: initialized)
- `in-progress` - Currently executing (corresponds to workflow_status: in-progress or paused)
- `completed` - All steps verified (corresponds to workflow_status: completed)

**CRITICAL:** 
- Do NOT add step lists or progress counters in PLAN.md. All tracking goes in progress.json.
- Do NOT add mode to PLAN.md. Mode is selected when execute starts.
- PLAN.md has only two frontmatter fields: `status` and `created` (execute updates status only).
- PLAN.md is human-readable summary; progress.json is system source of truth for all execution state.

### Step 4: Generate step-N.md Files

For each step, create `.workflow/TASK_NAME/steps/step-N.md`:

```yaml
---
step_number: {N}
name: {Step Name}
created: {TODAY}
---

# Step {N}: {Step Name}

## Goal

{One sentence description of what this step accomplishes}

## Files to Modify/Create

- [To be determined by implementer]

## Verification Criteria

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

### Step 5: Detect Project Type and Migration Framework

**Detect project language/build system** by checking for these files (in order):

1. `package.json` → `node`
2. `composer.json` → `php`
3. `pyproject.toml` or `requirements.txt` → `python`
4. `Cargo.toml` → `rust`
5. `go.mod` → `go`
6. `pom.xml` → `java-maven`
7. `build.gradle` or `build.gradle.kts` → `java-gradle`
8. `Gemfile` → `ruby`
9. `CMakeLists.txt` → `cpp-cmake`
10. `Makefile` → `generic-make`
11. If none found → `other` (ask user for build/test commands)

**Detect migration framework** (if applicable) by checking for:
- Node.js: `knex.js`, `sequelize`, `typeorm` migrations directories
- Python: `alembic/`, `migrations/` (SQLAlchemy)
- Go: `migrations/`, `sqlc/` directories or sql-migrate config
- Rust: `sqlx/migrations/`, `diesel/migrations/`
- Ruby: `db/migrate/`
- Java: `db/changelog/` (Liquibase) or `src/main/resources/db/migration/` (Flyway)

Load the detected project type's commands from `skills/templates/commands.yaml`. If detection fails or commands are missing, ask user to provide:
- Build command (e.g., `cargo build`)
- Test command (e.g., `cargo test`)
- Optional: Migrate command (e.g., `sqlx migrate run`, `npm run migrate`, `alembic upgrade head`)

### Step 5.5: Create .workflow-config.json

Create `.workflow/TASK_NAME/.workflow-config.json`:

```json
{
  "taskName": "{TASK_NAME}",
  "projectType": "{DETECTED_TYPE}",
  "buildCommand": "{BUILD_COMMAND}",
  "testCommand": "{TEST_COMMAND}",
  "migrateCommand": null
}
```

**Field Definitions (Project Configuration Only):**
- `taskName` (string) - Task identifier, matches directory name
- `projectType` (string) - Language/framework: node, php, python, rust, go, java-maven, java-gradle, ruby, cpp-cmake, generic-make, other
- `buildCommand` (string) - Command to build project (e.g., `npm run build`, `cargo build --release`, `go build ./...`)
- `testCommand` (string) - Command to run tests (e.g., `npm test`, `pytest`, `cargo test`, `go test ./...`)
- `migrateCommand` (string|null) - Command to run migrations if applicable (e.g., `npm run migrate`, `sqlx migrate run`, `alembic upgrade head`)

**CRITICAL:**
- .workflow-config.json is for **project configuration ONLY** (how to build/test this codebase)
- Do NOT add workflow execution settings here (mode, approval strategy, etc.)
- Do NOT add metadata that duplicates PLAN.md or progress.json
- These commands are used by implementer and verifier instead of assuming npm/node

### Step 6: Initialize progress.json

Create `.workflow/TASK_NAME/progress.json` for detailed step execution tracking. This is the **SOURCE OF TRUTH** for all workflow state.

**Example - 2-step workflow initialization:**

```json
{
  "task_name": "{TASK_NAME}",
  "mode": null,
  "workflow_status": "initialized",
  "created": "{TODAY}",
  "started": null,
  "completed": null,
  "current_step": 1,
  "steps": {
    "1": {
      "name": "{Step 1 Name}",
      "status": "pending",
      "iteration": 1,
      "awaiting_approval_since": null
    },
    "2": {
      "name": "{Step 2 Name}",
      "status": "pending",
      "iteration": 1,
      "awaiting_approval_since": null
    }
  },
  "approvals": {
    "mode_1_manual_approvals": []
  }
}
```

**Field Definitions:**

**Top-level workflow fields:**
- `task_name` (string) - Task identifier, matches directory name
- `mode` (number|null) - Workflow approval mode (null at init, set by execute at runtime):
  - `null` - Not yet set (waiting for execute to ask user)
  - `2` - Final Approve (approve only at end of all steps)
  - `1` - Step Manual Approve (approve each step before next)
- `workflow_status` (string) - Overall workflow status: `initialized`, `in-progress`, `paused`, `completed`
  - `initialized` - Created but not started (mode is null)
  - `in-progress` - Currently executing steps (mode is set)
  - `paused` - Waiting for approval (Step Manual Approve mode only)
  - `completed` - All steps verified and complete
- `created` (string) - Task creation date in YYYY-MM-DD format
- `started` (string|null) - ISO 8601 timestamp when first step began
- `completed` (string|null) - ISO 8601 timestamp when last step verified as complete
- `current_step` (number) - Currently active step number

**Step status values:**
- `pending` - Not started
- `implementation` - Implementer is working on this step
- `verification` - Verifier is testing this step
- `awaiting-approval` - Verification passed, waiting for human approval (Mode 1 only)
- `needs-fix` - Verifier found issues, needs re-implementation
- `complete` - Verified and approved (or auto-approved in Mode 2)

**Step-level fields:**
- `name` (string) - Step name from step-N.md frontmatter
- `status` (string) - One of values above: pending, implementation, verification, awaiting-approval, needs-fix, complete
- `iteration` (number) - Current iteration number (increments when step is retried after needs-fix)
- `awaiting_approval_since` (string|null) - ISO 8601 timestamp when step entered awaiting-approval status (Mode 1 only, used to track pause state)

**Approvals tracking:**
- `approvals.mode_1_manual_approvals` (array) - History of user approvals in Mode 1
  - Each entry: `{"step": 1, "approved_at": "2026-04-15T10:30:00Z", "approved_by": "user"}`

**Key Points:**
- All steps initialized to `pending` status, iteration 1
- Workflow-level `started` and `completed` are null at initialization (set by execute/finalize)
- `workflow_status` starts as `initialized`, becomes `in-progress` when first step starts
- `current_step` points to the step being worked on (updated by execute on resume)
- Workflow can be paused at `awaiting-approval` status (Mode 1) and resumed from same state
- PLAN.md is human-facing summary (only status and created fields); progress.json is system source of truth
- progress.json is updated by workflow:execute (status, workflow_status, current_step, started/completed)
- Mode 1: awaiting-approval triggers human decision; Mode 2: auto-completes after verification

### Step 7: Verify Files Created

Verify:
- [ ] PLAN.md exists with correct YAML frontmatter (status: pending, created date, title, description only)
- [ ] All step-N.md files exist with correct frontmatter (step_number, name, created only - NO status field)
- [ ] progress.json is valid JSON:
  - workflow_status: "initialized"
  - mode: null (will be set by execute)
  - All steps at "pending" status with iteration: 1
  - workflow-level started and completed are null
  - approvals.mode_1_manual_approvals is empty array
- [ ] .workflow-config.json exists with taskName, projectType, buildCommand, testCommand, migrateCommand fields
- [ ] buildCommand and testCommand are valid for detected projectType
- [ ] No execution settings in .workflow-config.json (only project config)

### Step 8: Report to User

Report files created (PLAN.md, progress.json, steps/step-*.md, .workflow-config.json), status as "Ready to execute", and next step: `/workflow:execute`

## Example

Input: `feature-auth` with 3 steps

Output: `.workflow/feature-auth/` with PLAN.md, progress.json, steps/step-1.md through step-3.md, and .workflow-config.json

Ready to execute: `/workflow:execute` (you will choose mode when execute starts)

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
