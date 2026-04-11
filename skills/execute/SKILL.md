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

### Execution Loop

At each tick YOU (the orchestrator in main session) must:

1. **Read PLAN.md** → determine mode and overall status
2. **Find first non-complete step** → check step-N.md frontmatter `status`
3. **Check status → dispatch subagent via Agent() tool:**
   - If status = `pending`, `implementation`, or `needs-fix` → **Call Agent() for implementer** (see below)
   - If status = `verification` → **Call Agent() for verifier** (see below)
   - If status = `complete` → Ask user for approval (Mode 1) or continue to next step (Mode 2)
4. **Wait for Agent() to return** (foreground call)
5. **Read updated step-N.md** → check new status
6. **Update PLAN.md step table** to reflect new status
7. **Repeat loop** until all steps `complete`

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

### CALL THIS IMMEDIATELY:

```python
Agent(
  description: "Workflow implementer for task {TASK_NAME}, step {N}",
  subagent_type: "general-purpose",
  prompt: """You are the workflow implementer subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

CRITICAL: You are in an ISOLATED subagent context. You have full permissions for:
- Reading/writing project files
- Running builds, tests, git commands
- Making implementation decisions
- Updating workflow step files

Do NOT ask the user for permission on anything.

PROCEDURE:
1. Invoke Skill("workflow:implementer") and follow its procedure exactly.
2. Read the step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
3. Read PLAN.md for context: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/PLAN.md
4. Perform the implementation. Make code changes, run local checks.
5. Update step-N.md: frontmatter status → "verification", fill Implementation section.
{IF NEEDS-FIX: "6. The previous verifier reported these issues — fix each one:\n{ISSUES_FROM_FILE}"}
6. Return a brief report (under 150 words): what changed, blockers, files touched.
   Do NOT paste file contents or full diffs.

AFTER COMPLETION:
- Step file status MUST be "verification"
- Report progress back to orchestrator"""
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

### CALL THIS IMMEDIATELY:

```python
Agent(
  description: "Workflow verifier for task {TASK_NAME}, step {N}",
  subagent_type: "general-purpose",
  prompt: """You are the workflow verifier subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

CRITICAL: You are in an ISOLATED subagent context. You have full permissions for:
- Reading all project files
- Running builds, tests, Docker commands
- Making verification decisions
- Updating workflow step files

Do NOT ask the user for permission on anything.

EXECUTION RULE: You MUST execute code and tests. Do not just review code.

PROCEDURE:
1. Invoke Skill("workflow:verifier") and follow its procedure exactly.
2. Read the step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
3. Read project files and understand what was implemented
4. EXECUTE verification (mandatory steps):
   - Set up test environment (Docker, databases, services)
   - Build/compile the project (if build fails → FAIL)
   - Run ALL test suites (if any test fails → FAIL)
   - Manually test each verification criterion by RUNNING the code
   - Prove everything works with concrete evidence (test output, curl responses, etc.)
5. On PASS: 
   - Update frontmatter status → "complete"
   - Fill Verification section with PASS notes and proof
6. On FAIL:
   - Update frontmatter status → "needs-fix"
   - Increment iteration number
   - Document "Issues Identified" section with exact errors and file locations
7. Return a brief report (under 150 words): PASS or FAIL with evidence.

AFTER COMPLETION:
- Step file status MUST be "complete" or "needs-fix"
- Report progress back to orchestrator"""
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

### Critical Differences from Main Session

- **Verifier runs in isolated subagent context** (invisible to you)
- Build output, test runs, manual testing — all invisible
- You only see brief summary + step file updates
- This keeps your conversation clean while verifier works independently

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
