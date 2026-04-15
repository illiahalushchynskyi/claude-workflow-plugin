---
name: workflow:implementer
description: SUBAGENT ONLY - Use when a workflow step is in implementation phase, dispatched via workflow:execute skill
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Implementer

⚠️ **EXECUTION GUARD: SUBAGENT ONLY**

This skill ONLY runs in subagent context via workflow:execute.
DO NOT call this skill directly from main session.

- If in main session → Use `workflow:execute` instead
- If you are a subagent → You were correctly dispatched, proceed

## Procedure

1. **Load Project Config**
   - Read `.workflow-config.json` from task directory
   - Extract: projectType, buildCommand, testCommand, migrateCommand
   - Use these commands instead of hardcoded npm commands

2. **Read** step definition from {TASK_DIR}/steps/step-{N}.md
   - Extract goal, criteria, files to modify

3. **Implement** code changes
   - Use Edit for existing files, Write for new files

4. **Test** - MANDATORY: Use `${testCommand}` from config
   - Must pass on all test frameworks (npm test, pytest, cargo test, go test, etc.)
   - Fix code or tests until all pass
   - Document results

5. **Run Migrations** (if applicable)
   - If `migrateCommand` is set in config (not null): run `${migrateCommand}`
   - Examples: `npm run migrate`, `sqlx migrate run`, `alembic upgrade head`, `python manage.py migrate`
   - Migrations must succeed—if failed, fix migration files or code and retry
   - Document migration results

6. **Commit** - `git commit -m "Implement step {N}: {goal}"`

7. **Update Implementation section** in step-{N}.md
   - Files modified/created
   - Summary of changes
   - Test results (including which testCommand was used)
   - Migration results (if migrations ran)
   - Do NOT update status field (execute manages it)

8. **Report** back to execute with brief summary

## Files

**Read:** step-{N}.md, PLAN.md, project source
**Write:** source files, step-{N}.md Implementation section only

**Do NOT modify:** Status field, progress.json, PLAN.md

## Critical

**MUST:**
- ✅ Load and use .workflow-config.json commands (not hardcoded npm)
- ✅ Understand goal before coding
- ✅ Run tests using testCommand from config - all must pass
- ✅ Commit changes
- ✅ Update Implementation section with command info
- ✅ Report summary to execute

**NEVER:**
- ❌ Update step status
- ❌ Update progress.json
- ❌ Claim done if tests fail
