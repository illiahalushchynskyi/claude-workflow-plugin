---
name: workflow:bootstrap
description: Initialize a new workflow task - creates PLAN.md and initial step files from task description and requirements
---

# Workflow Bootstrap Skill

## Overview

The `bootstrap` skill initializes a new workflow task by creating the complete directory structure and template files needed for execution. It prompts the user for task details, mode selection, and step planning, then generates all necessary markdown files with proper frontmatter.

## When to Use

Use `bootstrap` when:
- Starting a new development task that requires multi-step implementation
- You want to establish clear goals and step definitions before work begins
- Planning a feature, bugfix, refactoring, or architectural change
- Setting up a task for Mode 1 (step-by-step) or Mode 2 (end-to-end) execution

## Input Parameters

The skill prompts for:

1. **Task Name** — identifier for the workflow (e.g., `feature-auth-system`, `bugfix-timeout`)
   - Used in directory structure: `.workflow/TASK_NAME/`
   - Must be URL-safe (alphanumerics, hyphens, underscores)

2. **Task Title** — human-readable title (e.g., "Build JWT Authentication System")

3. **Task Description** — 2-3 sentences explaining what this task accomplishes

4. **Mode Selection** — choose Mode 1 or Mode 2:
   - **Mode 1:** Step-by-step verification with human approval after each step
   - **Mode 2:** End-to-end execution with single human approval at completion

5. **Step Definitions** — for each step, provide:
   - Step name (e.g., "Add JWT token generation")
   - Goal (what this step accomplishes)
   - Verification criteria (how verifier knows it's correct)

6. **Architecture & Approach** — brief explanation of overall strategy

## Output

Creates the following file structure:

```
project-root/
└── .workflow/
    └── TASK_NAME/
        ├── PLAN.md                    # Main plan file
        ├── .workflow-config.json      # Configuration for this task
        └── steps/
            ├── step-1-<name>.md       # Initial step files
            ├── step-2-<name>.md
            └── ... (one per step)
```

### PLAN.md Template

```markdown
---
task_name: TASK_NAME
title: Task Title
mode: 1 or 2
status: planning
created: YYYY-MM-DD
started: null
completed: null
---

# Task Title

## Goal
[One sentence description]

## Architecture & Approach
[2-3 sentences explaining strategy]

## Mode Selected
- **Mode 1 (Step-by-Step)**: [Selected if chosen]
- **Mode 2 (End-to-End)**: [Selected if chosen]

## Steps Overview
| Step | Name | Status | Issues |
|------|------|--------|--------|
| 1 | [step-1-name] | pending | — |
| 2 | [step-2-name] | pending | — |
...

## Overall Progress
- [ ] Step 1 Complete
- [ ] Step 2 Complete
...
- [ ] Human Approval
- [ ] Commit & Merge

## Step Details Location
All step detail files are in: `.workflow/TASK_NAME/steps/`

## Notes & Escalations
[None yet]
```

### .workflow-config.json Template

```json
{
  "task_name": "TASK_NAME",
  "mode": 1 or 2,
  "auto_advance": false,
  "verification_strategy": "comprehensive",
  "notification_method": "none",
  "branch_prefix": "workflow-",
  "commit_on_complete": true,
  "step_approval_required": true
}
```

### step-N-<name>.md Template

```markdown
---
step_number: N
name: Step Name
status: pending
iteration: 1
created: YYYY-MM-DD HH:MM
---

# Step N: Step Name

## Goal
[What this step accomplishes]

## Verification Criteria
[How verifier knows it's correct]
- Criterion 1
- Criterion 2
- Criterion 3

## Implementation

### Files to Modify/Create
- Create: `path/to/new-file.ts`
- Modify: `path/to/existing.ts` (lines X-Y)

### Implementation Notes
[Implementer updates this section as they work]

**Implementer**: [Name/timestamp]
- [Updates as work progresses]

## Verification

### Verification Notes
[Verifier updates this section after testing]

**Verifier**: [Name/timestamp]
- [Test results]

**Result**: [PASS|FAIL]

### If Issues Found
[List issues and fixes if re-verification needed]

---

**History**: 
- Iteration 1: [date] - [Status]
```

## Process Flow

1. **Collect Input**
   - Prompt user for task details (name, title, description)
   - Get mode selection (1 or 2)
   - Get step definitions (count, names, goals, verification criteria)

2. **Validate Input**
   - Ensure task name is URL-safe
   - Verify at least 1 step defined
   - Check mode is 1 or 2

3. **Create Directory Structure**
   - Create `.workflow/TASK_NAME/` directory
   - Create `.workflow/TASK_NAME/steps/` subdirectory

4. **Generate Files**
   - Create `.workflow/TASK_NAME/PLAN.md` with frontmatter
   - Create `.workflow/TASK_NAME/.workflow-config.json`
   - Create `.workflow/TASK_NAME/steps/step-1-*.md` through step-N-*.md

5. **Initialize Git**
   - Create feature branch: `workflow-TASK_NAME`
   - Commit initial structure: "workflow: initialize TASK_NAME workflow"

6. **Signal Ready**
   - Print confirmation with directory structure
   - Display next steps (use `workflow:execute` to start)
   - Print PLAN.md location for review

## Validation

The skill validates:

- Task name format (alphanumerics, hyphens, underscores only)
- Mode is integer 1 or 2
- At least 1 step defined
- Each step has name, goal, and verification criteria
- PLAN.md frontmatter matches schema

## Next Steps

After bootstrap completes:

1. **Review PLAN.md** — ensure goals and steps are clear
2. **Refine Step Details** — add more detail to verification criteria if needed
3. **Start Execution** — use `workflow:execute` to begin work

## Error Handling

If bootstrap fails, check:

- **"task name already exists"** — `.workflow/TASK_NAME/` already present, choose different name
- **"invalid mode"** — mode must be 1 or 2
- **"no steps defined"** — add at least 1 step
- **"step name conflicts"** — step names must be unique per task

## Related Skills

- `workflow:execute` — Start workflow execution after bootstrap
- `workflow:implementer` — Implement individual steps
- `workflow:verifier` — Verify completed steps
