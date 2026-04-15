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

---

## Project Type Reference Table

Use this table to detect project type and set build/test commands:

| Detection | Type | Build Command | Test Command | Migrate Command |
|-----------|------|---|---|---|
| `package.json` exists | `node` | `npm run build` | `npm test` | `npm run migrate` |
| `composer.json` exists | `php` | (none) | `composer test` or `phpunit` | (varies) |
| `pyproject.toml` or `requirements.txt` | `python` | `python setup.py build` | `pytest` | `alembic upgrade head` |
| `Cargo.toml` exists | `rust` | `cargo build --release` | `cargo test` | (none typically) |
| `go.mod` exists | `go` | `go build ./...` | `go test ./...` | (varies) |
| `pom.xml` exists | `java-maven` | `mvn clean compile` | `mvn test` | (varies) |
| `build.gradle` or `build.gradle.kts` | `java-gradle` | `gradle build` | `gradle test` | (varies) |
| `Gemfile` exists | `ruby` | (varies) | `rspec` or `rake test` | (varies) |
| `CMakeLists.txt` exists | `cpp-cmake` | `cmake --build .` | `ctest` | (none) |
| `Makefile` exists | `generic-make` | `make` | `make test` | (varies) |
| None of above | `other` | Ask user | Ask user | Ask user |

**Detection Order:** Check in the order above. Use first match.

---

## Your Procedure

### Step 1: Validate Input

**GUARD:** STOP if user cannot provide all required information

**PRE-CONDITION:** None (entry point)

**REQUIRED FIELDS** (ask for all):
1. **Task name** - format: lowercase, hyphens only (e.g., `feature-auth`)
2. **Task title** - human-readable title
3. **Description** - what this workflow accomplishes
4. **Steps** - list of steps with:
   - Step name
   - Step goal (one sentence: what this step accomplishes)
   - Verification criteria (minimum 3 per step—these are what verifier will check)

**PROCEDURE:**

Use AskUserQuestion for any missing field. Present fields one at a time:

```
Question 1: Task Name
  - Format: lowercase, hyphens only
  - Example: "feature-auth", "fix-pagination"
  - If user provides invalid format: suggest correction

Question 2: Task Title
  - Human-readable description
  - Example: "Add user authentication system"

Question 3: Description
  - What does this workflow accomplish?
  - Example: "Implement JWT-based authentication..."

Question 4: Steps List
  - How many steps? (user provides count)
  - For each step:
    - Step name
    - Goal (one sentence)
    - Verification criteria (at least 3)
```

**VERIFICATION CRITERIA VALIDATION:**

Each criterion MUST be:
- **Specific** - Not vague ("it works", "feature is done")
- **Testable** - Can verify with concrete proof (curl response, database query result, test output, etc.)
- **Independent** - Can verify each criterion separately

**GOOD Examples:**
- ✓ "API endpoint POST /users returns 201 status and user object with id field"
- ✓ "Database table 'users' contains inserted record with correct name"
- ✓ "All unit tests pass with exit code 0"

**BAD Examples:**
- ✗ "User endpoint works correctly"
- ✗ "Everything is done"
- ✗ "Feature implemented"

**VALIDATION LOGIC:**

For each criterion user provides:
```
Is it specific? (NOT vague wording)
  NO → Ask for clarification
Is it testable? (Has concrete proof method)
  NO → Ask what concrete evidence would prove this
Is it independent? (Can verify separately)
  NO → Ask user to split into separate criteria
```

If all criteria valid: Continue to Step 2

**POST-CONDITION:** All 5 fields validated, criteria meet standards

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

**GUARD:** User must provide task description before reaching this step

**PRE-CONDITION:** Task name and description received from user

**PROCEDURE:**

**1. Detect Project Type:**

Check for files **in this order**. Use first match:

```
1. package.json       → type = "node"
2. composer.json      → type = "php"
3. pyproject.toml OR requirements.txt → type = "python"
4. Cargo.toml         → type = "rust"
5. go.mod             → type = "go"
6. pom.xml            → type = "java-maven"
7. build.gradle or build.gradle.kts → type = "java-gradle"
8. Gemfile            → type = "ruby"
9. CMakeLists.txt     → type = "cpp-cmake"
10. Makefile          → type = "generic-make"
11. None found        → type = "other" (ask user)
```

**2. Detect Migration Framework (if applicable):**

Check for migration config files by project type:

| Type | Check For | If Found |
|------|-----------|----------|
| `node` | `knex.js`, `sequelize`, `typeorm` directories | Extract migrate command |
| `python` | `alembic/`, `migrations/` | Extract migrate command |
| `go` | `migrations/`, `sqlc/` directories | Extract migrate command |
| `rust` | `sqlx/migrations/`, `diesel/migrations/` | Extract migrate command |
| `ruby` | `db/migrate/` | Use `rake db:migrate` |
| `java-*` | `db/changelog/`, `src/main/resources/db/migration/` | Extract migrate command |

If migration framework found: set migrateCommand in config.json
If not found: set migrateCommand = null in config.json

**3. Get Build/Test Commands:**

Look up detected type in **Project Type Reference Table** (above) and extract:
- buildCommand (standard for that type)
- testCommand (standard for that type)
- migrateCommand (if framework detected)

**4. If Type = "other":**

Ask user via AskUserQuestion:
```
question: "Please provide your project's build and test commands"
options:
  - Custom build and test commands (requires manual entry)
```

Then ask:
- "What is your build command?" (e.g., `make build`)
- "What is your test command?" (e.g., `make test`)
- "Do you have a migration command?" (optional, e.g., `make migrate`)

**POST-CONDITION:** projectType determined, build/test commands identified

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

**GUARD:** All previous steps completed before creating progress.json

**PRE-CONDITION:** All fields from Step 1 validated, project type detected from Step 5

**SOURCE OF TRUTH:** progress.json is THE ONLY place execute reads/writes workflow state

**STRUCTURE:** Create `.workflow/TASK_NAME/progress.json` with this schema:

```json
{
  "task_name": "string",
  "mode": null,
  "workflow_status": "initialized",
  "created": "YYYY-MM-DD",
  "started": null,
  "completed": null,
  "current_step": 1,
  "steps": {
    "1": {"name": "string", "status": "pending", "iteration": 1, "awaiting_approval_since": null},
    "2": {"name": "string", "status": "pending", "iteration": 1, "awaiting_approval_since": null}
  },
  "approvals": {
    "mode_1_manual_approvals": []
  }
}
```

**Field Definitions:**

**Workflow-level:**
| Field | Type | Initial Value | Purpose |
|-------|------|---|---|
| `task_name` | string | User input | Task identifier |
| `mode` | number or null | `null` | Approval mode: `1` = Step Manual, `2` = Final, `null` = not set yet |
| `workflow_status` | string | `"initialized"` | Overall state: initialized→in-progress→paused→completed |
| `created` | string (YYYY-MM-DD) | Today | Immutable creation date |
| `started` | string (ISO8601) or null | `null` | Set by execute when first step begins |
| `completed` | string (ISO8601) or null | `null` | Set by finalize when workflow ends |
| `current_step` | number | `1` | Currently active step |

**Per-Step (steps.{N}):**
| Field | Type | Initial Value | Purpose |
|-------|------|---|---|
| `name` | string | Step name | Reference to step-N.md |
| `status` | string | `"pending"` | pending→implementation→verification→awaiting-approval→needs-fix→complete |
| `iteration` | number | `1` | Increments when step retried after needs-fix |
| `awaiting_approval_since` | ISO8601 or null | `null` | Set by execute when status = awaiting-approval (Mode 1 pause tracking) |

**Approvals:**
| Field | Type | Initial Value | Purpose |
|-------|------|---|---|
| `approvals.mode_1_manual_approvals` | array | `[]` | History: `[{"step": N, "approved_at": "ISO8601", "approved_by": "user"}, ...]` |

**EXAMPLE - 3-step workflow:**
```json
{
  "task_name": "feature-auth",
  "mode": null,
  "workflow_status": "initialized",
  "created": "2026-04-15",
  "started": null,
  "completed": null,
  "current_step": 1,
  "steps": {
    "1": {"name": "Setup auth database", "status": "pending", "iteration": 1, "awaiting_approval_since": null},
    "2": {"name": "Implement JWT middleware", "status": "pending", "iteration": 1, "awaiting_approval_since": null},
    "3": {"name": "Add login endpoint", "status": "pending", "iteration": 1, "awaiting_approval_since": null}
  },
  "approvals": {
    "mode_1_manual_approvals": []
  }
}
```

**POST-CONDITION:** progress.json created with all steps initialized to pending
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

**GUARD:** Cannot continue if verification fails

**VERIFICATION CHECKLIST:**

| File | Check | Must Pass |
|------|-------|-----------|
| **PLAN.md** | Frontmatter has ONLY: `status: pending`, `created: YYYY-MM-DD` | ✅ YES |
| **PLAN.md** | Body has title and description (no step table) | ✅ YES |
| **step-N.md** | Frontmatter has ONLY: `step_number`, `name`, `created` | ✅ YES |
| **step-N.md** | NO status field in frontmatter | ✅ YES |
| **step-N.md** | Goal section filled | ✅ YES |
| **step-N.md** | Verification Criteria section lists all criteria | ✅ YES |
| **progress.json** | Valid JSON (no syntax errors) | ✅ YES |
| **progress.json** | `workflow_status: "initialized"` | ✅ YES |
| **progress.json** | `mode: null` (will be set by execute) | ✅ YES |
| **progress.json** | All steps have `status: "pending"` | ✅ YES |
| **progress.json** | All steps have `iteration: 1` | ✅ YES |
| **progress.json** | `started: null`, `completed: null` | ✅ YES |
| **progress.json** | `approvals.mode_1_manual_approvals: []` | ✅ YES |
| **.workflow-config.json** | Has: taskName, projectType, buildCommand, testCommand, migrateCommand | ✅ YES |
| **.workflow-config.json** | No mode, no approval settings, no execution config | ✅ YES |
| **.workflow-config.json** | buildCommand is valid for projectType | ✅ YES |
| **.workflow-config.json** | testCommand is valid for projectType | ✅ YES |

**VALIDATION PROCEDURE:**

```bash
# For each check:
if [ NOT_TRUE ]; then
  echo "ERROR: [Check Name] failed"
  exit 1
fi
```

If ANY check fails: STOP and report which check failed and why

**POST-CONDITION:** All files exist and pass validation

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
