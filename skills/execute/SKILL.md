---
name: workflow:execute
description: Use when a workflow plan is ready and execution should begin - orchestrates step implementation and verification through isolated subagents
---

# Workflow Execute Skill (Orchestrator)

## Overview

`execute` orchestrates workflow execution by reading `PLAN.md`, determining what work to do next, and **dispatching implementer and verifier work as isolated subagents via the `Agent` tool**. The orchestrator itself never edits project code and never runs tests — it only reads state files, launches subagents, reads their updated state, and pauses for human approval.

This is the core architectural rule: **keeping implementation and verification in separate subagent contexts is what prevents the main conversation from being flooded with file reads, diffs, and test output**. If the orchestrator ever does the work itself, the design has failed.

## When to Use

**Use `execute` immediately and automatically when:**
- A workflow has been bootstrapped (`PLAN.md` and step files exist and verified)
- PLAN.md `status` is `executing` or `planning` (not yet started)
- User asks to continue/resume workflow or says "let's start"

**`execute` MUST run in the main session** (not as a subagent). It orchestrates by dispatching implementer and verifier work as separate subagents.

**Where the work happens:**
- **Orchestrator (you)**: Reads files, launches subagents, tracks state → MAIN SESSION
- **Implementer**: Writes code, makes changes → ISOLATED SUBAGENT
- **Verifier**: Tests code, runs builds/tests → ISOLATED SUBAGENT
- **User**: Approves steps (Mode 1) → MAIN SESSION

## The Iron Rule

> The orchestrator MUST NOT read source files, write source files, or run project commands itself.
> Every implementation and verification action is delegated to a subagent through the `Agent` tool.

The only tools the orchestrator uses directly:

- `Read` — on `.workflow/TASK_NAME/PLAN.md` and `.workflow/TASK_NAME/steps/step-N.md` only
- `Edit` — on `PLAN.md` step table and status fields only
- `Agent` — to dispatch implementer and verifier subagents
- `AskUserQuestion` — to request human approval at pause points

Any time you feel the urge to open a project file, run `npm test`, or write code — **stop and dispatch a subagent instead.**

## Subagent Permissions

**Implementer and verifier subagents have automatic permissions** (see CLAUDE.md):

- ✅ Full read/write access to project files
- ✅ Run builds, tests, git commands without asking
- ✅ Update workflow step files
- ✅ Make independent decisions about code quality

Subagents do NOT ask users for permission during execution. They work independently and report results.

## Inputs

1. **`.workflow/TASK_NAME/PLAN.md`** — mode, step list, overall status
2. **`.workflow/TASK_NAME/steps/step-N.md`** — per-step status and notes
3. **`.workflow/TASK_NAME/.workflow-config.json`** — optional settings

## ⚡ EXECUTE THIS NOW (Not Documentation)

**YOU ARE THE ORCHESTRATOR. Follow this execution loop exactly:**

### Step 1: Check Current Status

**ALWAYS START HERE** when executing:

1. **Read PLAN.md** → check current status (planning/executing/ready-for-review)
2. **Read all step files** → find first non-complete step and note its current status
3. **Report to user:**
   ```
   Workflow Status:
   - PLAN status: {executing}
   - Current step: Step 3 (Add API endpoints)
   - Step status: {needs-fix}
   - Last action: Verifier found issues with error handling
   
   I'm about to:
   1. Read Issues Identified from step-3.md
   2. Dispatch implementer subagent to fix issues
   3. Run verifier again
   
   Ready to proceed? (yes/no)
   ```
4. **Wait for user confirmation** before dispatching any subagent

### Execution Loop

At each tick YOU (the orchestrator in main session) must:

1. **Find first non-complete step** → check step-N.md frontmatter `status`
2. **Check status → dispatch subagent via Agent() tool:**
   - If status = `pending`, `implementation`, or `needs-fix` → **Call Agent() for implementer** (see below)
   - If status = `verification` → **Call Agent() for verifier** (see below)
   - If status = `complete` → Ask user for approval (Mode 1) or continue to next step (Mode 2)
3. **Wait for Agent() to return** (foreground call)
4. **Read updated step-N.md** → check new status
5. **Update PLAN.md step table** to reflect new status
6. **Repeat loop** until all steps `complete`

### When You (Orchestrator) Must Act

**DO THIS NOW:**
```
Loop:
  Read PLAN.md status
  Find first step where status ≠ complete
  
  if step.status in {pending, implementation, needs-fix}:
    → CALL Agent(implementer subagent) - see "Dispatching Implementer" below
    → Wait for result
    → Read step-N.md again
    → Loop
  
  if step.status == verification:
    → CALL Agent(verifier subagent) - see "Dispatching Verifier" below
    → Wait for result
    → Read step-N.md again
    → Loop
  
  if step.status == complete:
    → Ask user: "Step N complete. Approve?" (Mode 1 only)
    → Loop to next step
```

### Status → action map

| Step status     | Orchestrator action                                           |
|-----------------|---------------------------------------------------------------|
| `pending`       | Dispatch **implementer** subagent                             |
| `implementation`| Dispatch **implementer** subagent (resuming)                  |
| `needs-fix`     | Dispatch **implementer** subagent with verifier's issues      |
| `verification`  | Dispatch **verifier** subagent                                |
| `complete`      | Mode 1: pause for human approval; Mode 2: advance to next step|

## ⚡ DISPATCHING THE IMPLEMENTER (DO THIS NOW)

**When step.status is `pending`, `implementation`, or `needs-fix`:**

### CALL THIS IMMEDIATELY (Do NOT invoke Skill() in the prompt):

```python
Agent(
  description: "Workflow implementer for task {TASK_NAME}, step {N}",
  subagent_type: "general-purpose",
  prompt: """You are the workflow implementer subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

CRITICAL: You are in an ISOLATED subagent context with full permissions:
- Read/write any project files
- Run builds, tests, migrations, git commands
- Update workflow step files
- Make implementation decisions independently

Do NOT ask user for permission. Work independently.

## Your Task

Implement step {N} ({STEP_NAME}) as described:

### Step 1: Read & Understand
- Read step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
- Read PLAN.md for context: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/PLAN.md
- Understand the goal and verification criteria
{IF NEEDS-FIX: "- Read 'Issues Identified' section — these are the issues to fix"}

### Step 2: Implement
- Make code changes following project patterns
- Create migration files if ORM models changed
- **Run migrations immediately** if schema changed: `npm run migrate`
- Commit changes with clear messages
- Run local tests: `npm test`

### Step 3: Verify Locally
- Ensure project builds: `npm run build`
- All tests passing: `npm test`
- Code follows project conventions
- No console errors or warnings

### Step 4: Update Step File
Edit {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md:
- Set frontmatter `status: verification`
- Set `iteration: {ITERATION}` (increment if fix cycle)
- Fill Implementation section with:
  - Files created/modified (with specific changes)
  - Key decisions made
  - Any blockers encountered
  - Test results (pass/fail counts)
  - Notes for verifier

### Step 5: Report Back
Return brief summary (under 150 words):
- What you implemented
- Any blockers or concerns
- Test results
- Files touched
Do NOT paste file contents or diffs.

## If You Get Stuck
- Clear blocker? Report status: BLOCKED, describe issue
- Need more context? Report status: NEEDS_CONTEXT
- Completed but uncertain? Report status: DONE_WITH_CONCERNS
- All good? Complete normally with status DONE"""
)
```

### Step-by-Step for This Agent() Call:

1. **Read the step file first** (you, in main session) to get exact paths and details
2. **Build the prompt** above with:
   - `{TASK_NAME}` = name from PLAN.md frontmatter
   - `{STEP_NAME}` = step name from step-N.md
   - `{ABSOLUTE_PATH}` = `/home/illia/work/claude-workflow-plugin` (or actual path)
   - `{N}` = step number
   - `{1|2}` = mode from PLAN.md
   - If status = `needs-fix`: `{ISSUES_FROM_FILE}` = copy "Issues Identified" section from step-N.md
3. **Call Agent()** with the prompt above
4. **WAIT for result** (foreground, synchronous)
5. **Check returned report** — should indicate success and status change to "verification"
6. **Read step-N.md again** to confirm status changed

### What Happens in Subagent (NOT in your session)
- Implementer runs in completely isolated context
- File reads, code writes, test output — all invisible to you
- You only see the brief summary returned
- This keeps your conversation clean

## ⚡ DISPATCHING THE VERIFIER (DO THIS NOW)

**When step.status is `verification`:**

### CALL THIS IMMEDIATELY (Do NOT invoke Skill() in the prompt):

```python
Agent(
  description: "Workflow verifier for task {TASK_NAME}, step {N}",
  subagent_type: "general-purpose",
  prompt: """You are the workflow verifier subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

CRITICAL: You are in an ISOLATED subagent context with full permissions:
- Read any project files
- Run builds, tests, Docker, git commands
- Update workflow step files
- Make verification decisions independently

Do NOT ask user for permission. Test independently.

## Your Task

Verify that step {N} implementation works correctly.

### Step 1: Setup & Prepare
- Read step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
- Read verification criteria section
- Understand what should work

### Step 2: Environment Setup (MANDATORY)
```bash
npm install
npm run build
docker-compose up -d  # if using Docker
docker-compose ps    # verify running
```

### Step 3: Verify Migrations (DO NOT RUN - ONLY CHECK)
```bash
# Check if migrations were run (not your job to run them)
psql -c "SELECT * FROM schema_migrations;"
# If incomplete → FAIL (implementer's responsibility)

psql -c "\\dt"  # verify expected tables exist
# If missing tables → FAIL
```

### Step 4: Run All Tests (MANDATORY)
```bash
npm test  # all tests MUST pass
# If any fail → FAIL immediately
```

### Step 5: Manual Testing Against Criteria (MANDATORY)
For each criterion in step-N.md:
- If API endpoint: call it with `curl`, verify response code and data
- If database: insert data, query DB with `psql`, verify persistence
- If page: load page, verify content with `curl` or browser
- If Docker service: verify container running, service accessible

Examples:
```bash
# Test API
curl -X POST http://localhost:3000/api/users \\
  -H "Content-Type: application/json" \\
  -d '{"name":"Test","email":"test@example.com"}'
# Check: 201 response with user ID

# Test DB persistence
psql -c "SELECT * FROM users WHERE email='test@example.com'"
# Check: Row exists with correct data

# Test error handling
curl -X POST http://localhost:3000/api/users -d '{}' 
# Check: 400 Bad Request (validation works)
```

### Step 6: Document Results

For each criterion tested, document:
- ✓ Criterion: [name]
  - Tested: [what command/action]
  - Expected: [expected result]
  - Got: [actual result]
  - Status: PASS or FAIL

### Step 7: Decision

If ALL tests pass AND ALL criteria verified:
- Set status: "complete"
- Fill "Verification Notes" with PASS proof
- Report: PASS with evidence

If ANY test fails OR ANY criterion fails:
- Set status: "needs-fix"
- Increment iteration counter
- Create "Issues Identified" section:
  ```
  ### Issues Identified
  1. [Issue name]
     - File: path/to/file.ts:line
     - Error: exact error message
     - Proof: how you discovered it
     - Fix needed: specific action
  ```
- Report: FAIL with specific issues

### Step 8: Report Back
Brief summary (under 150 words):
- PASS or FAIL
- Key evidence (test counts, curl responses, DB state)
- Do NOT paste full output"""
)
```

### Step-by-Step for This Agent() Call:

1. **Read the step file first** (you, in main session) to get paths and iteration count
2. **Build the prompt** above with:
   - `{TASK_NAME}` = name from PLAN.md
   - `{STEP_NAME}` = step name from step-N.md
   - `{ABSOLUTE_PATH}` = `/home/illia/work/claude-workflow-plugin`
   - `{N}` = step number
   - `{1|2}` = mode from PLAN.md
3. **Call Agent()** with the prompt above
4. **WAIT for result** (foreground, synchronous)
5. **Check returned report** — PASS or FAIL with evidence
6. **Read step-N.md again** to confirm status changed to `complete` or `needs-fix`
7. **If FAIL:** Extract "Issues Identified" for next implementer cycle

### How to Build & Call Agent() for Verifier:

**Before calling Agent():**

1. **Read step-N.md** (you, in main session)
2. **Extract values:**
   - All variables from step-N.md (name, number, iteration, etc.)
   - Extract "Issues Identified" if status was `needs-fix`
3. **Replace placeholders** in verifier prompt template above
4. **Call Agent()** with completed prompt
5. **WAIT** for Agent to return

**After Agent returns:**
- Read step-N.md to see updated status
- Check if status = `complete` or `needs-fix`
- Update PLAN.md
- Continue loop

### Critical: Subagent Isolation

- **Verifier runs in completely isolated context** (invisible to you)
- Build output, test runs, manual testing — all invisible
- You only see brief summary + updated step-N.md file
- Isolation keeps your context clean
- Each subagent starts fresh with no conversation history

## After Agent() Returns (You Still in Main Session)

1. **Read the step file** (only this file, NOT the source code it touched)
   ```bash
   cat .workflow/TASK_NAME/steps/step-N.md
   ```
2. **Check the frontmatter** — what is new `status`?
   - `verification` = implementer just finished
   - `complete` = verifier says it passes
   - `needs-fix` = verifier found issues
3. **Update PLAN.md step table** with new status
   ```bash
   # Edit PLAN.md: find the step row, change status column
   # Example: | Step 1 | Create API | verification | → | Step 1 | Create API | complete |
   ```
4. **Decide next action** from status:
   - If `verification`: loop back, call Agent() for verifier
   - If `needs-fix`: loop back, call Agent() for implementer (with issues)
   - If `complete`: ask user (Mode 1) or continue to next step (Mode 2)

**IMPORTANT: Do NOT read source files the subagent touched.** 
- Subagent's returned report is the truth
- You only need to check step-N.md and PLAN.md
- Keep your context clean

## Mode-Specific Behavior

### Mode 1: Step-by-Step with Human Approval

After each step reaches `complete`:
- Ask user: "Step {N} complete. Approve and continue to step {N+1}?"
- If approve: continue loop to next step
- If "request changes": set step status back to `needs-fix`, continue loop

At end (all steps `complete`):
- Set PLAN.status = `ready-for-review`
- Ask user final approval
- If approve: **Call Skill("workflow:finalize")** to complete workflow

### Mode 2: End-to-End with Verification Gates

**Same loop, but:**
- NO human pause between steps
- Verifier PASS (status = `complete`) is the only gate
- Only ask human once at the very end when PLAN.status = `ready-for-review`
- If all steps pass: can finalize immediately after final approval

## State Transitions

```
PLAN.md:
  planning → executing → ready-for-review → approved → complete

step-N.md (set by subagents, never by orchestrator):
  pending → implementation → verification → complete
                                         ↘ needs-fix → implementation → ...
```

The orchestrator only edits the `PLAN.md` step table and PLAN-level `status`. Per-step frontmatter is owned by the subagents.

## Resume & Checkpointing

On a fresh `execute` invocation:

1. Read `PLAN.md`
2. If `status == complete`: nothing to do
3. If `status == approved`: dispatch `workflow:finalize`
4. If `status == ready-for-review`: ask for final approval
5. If `status == executing`: find first non-complete step and re-enter the loop
6. If `status == planning`: set to `executing`, start at step 1

Because all state lives in files, a resume is just "re-read PLAN.md and step files, then continue the loop". No in-memory state is carried between sessions.

## Progress Display

Between subagent dispatches, print a short progress summary to the user:

```
Workflow: cms-predictor-platform (Mode 1)
  [x] Step 1: Database & Schema Setup
  [~] Step 2: Authentication System  ← dispatching verifier
  [ ] Step 3: Admin API Foundations
  ...
```

One line per step. No file contents, no diffs.

## Error Handling

1. **Subagent returned without updating step status** — re-dispatch once with an explicit reminder to update frontmatter. If still wrong, surface to user.
2. **Subagent report indicates a blocker** — stop the loop, surface the blocker via `AskUserQuestion`.
3. **Mode 2: previous step not complete** — should be impossible if the loop is correct; if seen, stop and report.
4. **`PLAN.md` missing or malformed** — stop and ask the user to restore or re-bootstrap.

## Anti-patterns (do not do these)

- ❌ Reading project source files in the orchestrator context to "check progress"
- ❌ Running `npm test` / `tsc` / build commands from the orchestrator
- ❌ Editing project files directly because "it's a small fix"
- ❌ Invoking `Skill("workflow:implementer")` in the main conversation (that loads the skill text here and tempts the orchestrator to do the work itself — always go through `Agent`)
- ❌ Asking the subagent to return full diffs or file contents
- ❌ Dispatching a subagent without the step file's absolute path

## Related Skills

- `workflow:bootstrap` — Creates the `PLAN.md` and step files this skill reads
- `workflow:implementer` — The procedure a dispatched implementer subagent follows
- `workflow:verifier` — The procedure a dispatched verifier subagent follows
- `workflow:finalize` — Called after final human approval
