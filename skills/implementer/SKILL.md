---
name: workflow:implementer
description: SUBAGENT ONLY - Use when a workflow step is in implementation phase, dispatched via workflow:execute skill
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Implementer

⚠️ **EXECUTION GUARD: SUBAGENT ONLY**

| Execution Context | Action |
|---|---|
| Running as **subagent** (dispatched by execute) | ✅ Proceed with full skill |
| Running in **main session** directly | ❌ STOP—use `workflow:execute` instead |
| Called without step definition | ❌ STOP—require step-N.md file |

**Pre-entry checks:**
- Am I running as a subagent via Agent() dispatch? **YES** → Continue
- Do I have step-N.md with goal and criteria? **YES** → Continue
- Do I have .workflow-config.json with build/test commands? **YES** → Continue
- **NO to any?** → Report error and exit

This skill is invoked by execute ONLY after user confirms subagent execution in Step 1.5.

## Procedure - Step-by-Step

### 1. Load Project Config

**ACTION:** Read `.workflow-config.json` from task directory

**EXTRACT:**
- `projectType` - Language/framework (node, python, rust, go, etc.)
- `buildCommand` - How to build (e.g., `npm run build`, `cargo build`)
- `testCommand` - How to test (e.g., `npm test`, `pytest`)
- `migrateCommand` - How to migrate (if not null) (e.g., `npm run migrate`)

**MUST USE:** These commands from config, NOT hardcoded npm/python

### 2. Read Step Definition

**ACTION:** Read `.workflow/{TASK_NAME}/steps/step-{N}.md`

**EXTRACT:**
- Goal (what this step accomplishes)
- Files to Modify/Create section
- Verification Criteria (what verifier will check)

### 3. Implement Code Changes

**ACTION:** Write code to accomplish goal

**HOW:**
- Use Edit tool for existing files
- Use Write tool for new files
- Make changes to files listed in step

**SUCCESS SIGNAL:** Changes match goal and meet all criteria listed

### 4. Test - MANDATORY

**ACTION:** Run `${testCommand}` from config (not npm test!)

**MUST PASS:** All tests pass with exit code 0

**IF FAIL:**
- Fix code or test issues
- Rerun testCommand
- Repeat until ALL tests pass

**DOCUMENT:** Test output and which testCommand was used

### 5. Run Migrations (if applicable)

**CONDITION:** Check if `migrateCommand` is set in config.json
- ✅ If set (not null): run migrations (Step 5a)
- ✅ If null: skip (Step 5b)

**5a. If migrateCommand set:**
```bash
run ${migrateCommand}
# Examples: npm run migrate, sqlx migrate run, alembic upgrade head
```

**MUST SUCCEED:** Migration runs without error
- If failed: fix migration files/code, retry
- Document migration output

**5b. If migrateCommand null:**
- Skip migrations entirely
- Note in Implementation: "No migrations needed"

### 6. Create Commit

**ACTION:** 
```bash
git commit -m "Implement step {N}: {goal}"
```

**REQUIREMENT:** Commit must exist in git log

### 7. Update Implementation Section

**ACTION:** Update `.workflow/{TASK_NAME}/steps/step-{N}.md` Implementation section:

```markdown
## Implementation

### Files Modified/Created
- File 1: description of changes
- File 2: description of changes
- (new) File 3: description of what was added

### Implementation Notes
- Summary of changes made
- Key decisions made

### Test Results
- Test command used: ${testCommand}
- Result: ✓ All {N} tests passed (exit code 0)
- Evidence: [test output summary]

### Migration Results
- Migration command used: ${migrateCommand}
- Result: ✓ Migrations applied successfully
- Evidence: [migration output or schema verification]
(OR: "No migrations required for this step")
```

**CRITICAL:** 
- ✅ DO fill Implementation section completely
- ❌ DO NOT update status field (execute manages it)
- ❌ DO NOT update progress.json

### 8. Signal Completion to Execute

**COMPLETION SIGNAL:** step-{N}.md Implementation section is FULLY FILLED

Execute will:
1. Read Implementation section
2. Verify it's filled (not empty)
3. Set progress.json status = `verification`
4. Dispatch verifier for next phase

**HOW IT WORKS:**
- In subagent mode: Agent naturally completes when this step finishes
- Execute sees implementation section filled → knows we're done → moves to verification

## Files

**Read:** step-{N}.md, PLAN.md, project source
**Write:** source files, step-{N}.md Implementation section only

**Do NOT modify:** Status field, progress.json, PLAN.md

## Rules - Critical for Success

### MUST DO ✅

| Rule | Why | How to Verify |
|------|-----|---|
| Load .workflow-config.json | Different projects use different commands | Check buildCommand/testCommand from config, not hardcoded |
| Understand step goal BEFORE coding | Prevents wrong implementation | Read goal section, ask if unclear |
| Run testCommand from config | Tests validate implementation | Run exact command from config.json, not npm test |
| All tests MUST pass | Can't proceed if tests fail | Exit code 0 + all tests pass |
| Run migrations if set | Data structure changes must be applied | Check if migrateCommand != null, run if set |
| Commit changes to git | Track work in version control | `git log` shows commit with step number |
| Update Implementation section fully | Execute needs to know you're done | Fill: files, notes, test results, migration results |
| Implementation section = completion signal | Execute reads this to proceed | Leave NO section empty |

### NEVER DO ❌

| Rule | Why | Consequence |
|---|---|---|
| Update step status field | Execute manages all state | Causes sync problems |
| Update progress.json | Only execute modifies this | Causes state corruption |
| Claim done if tests fail | Tests are verification gate | Verifier will fail step |
| Use hardcoded npm/make/etc | Projects use different build systems | Breaks in non-Node projects |
| Assume what to build | Must read step goal | Implements wrong feature |

## Completion Checklist

Before finishing, verify:

```
✅ Step goal understood (re-read goal section)
✅ All code changes implemented
✅ ${testCommand} run and passed (all tests green)
✅ Migrations run IF migrateCommand set (else skipped)
✅ Git commit created with step {N} in message
✅ Implementation section FILLED with:
   - Files modified/created listed
   - Changes summarized
   - Test command and results documented
   - Migration results documented (or "not needed")
✅ Status field NOT touched
✅ progress.json NOT modified
```

If ANY item unchecked: RETURN to that step, fix it, re-verify
