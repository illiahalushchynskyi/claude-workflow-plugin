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

1. **Read** step definition from {TASK_DIR}/steps/step-{N}.md
   - Extract goal, criteria, files to modify

2. **Implement** code changes
   - Use Edit for existing files, Write for new files
   - Run migrations if needed: `npm run migrate`

3. **Test** - MANDATORY: `npm test` must pass
   - Fix code or tests until all pass
   - Document results

4. **Commit** - `git commit -m "Implement step {N}: {goal}"`

5. **Update Implementation section** in step-{N}.md
   - Files modified/created
   - Summary of changes
   - Test results
   - Do NOT update status field

6. **Report** back to execute with brief summary

## Files

**Read:** step-{N}.md, PLAN.md, project source
**Write:** source files, step-{N}.md Implementation section only

**Do NOT modify:** Status field, progress.json, PLAN.md

## Critical

**MUST:**
- ✅ Understand goal before coding
- ✅ Run tests - all must pass
- ✅ Commit changes
- ✅ Update Implementation section
- ✅ Report summary to execute

**NEVER:**
- ❌ Update step status
- ❌ Update progress.json
- ❌ Claim done if tests fail
