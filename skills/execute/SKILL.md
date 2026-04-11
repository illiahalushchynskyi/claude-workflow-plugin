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

## Inputs

1. **`.workflow/TASK_NAME/PLAN.md`** — mode, step list, overall status
2. **`.workflow/TASK_NAME/steps/step-N.md`** — per-step status and notes
3. **`.workflow/TASK_NAME/.workflow-config.json`** — optional settings

## Execution Loop

At each tick the orchestrator:

1. Reads `PLAN.md` → determines mode and overall status
2. Finds the first step whose status is not `complete`
3. Dispatches the appropriate subagent for that step's current status
4. Reads the updated `step-N.md` when the subagent returns
5. Updates `PLAN.md` step table to reflect new status
6. In Mode 1: pauses for human approval after a step reaches `complete`
7. Repeats until all steps are `complete`, then marks PLAN `ready-for-review` and asks for final approval

### Status → action map

| Step status     | Orchestrator action                                           |
|-----------------|---------------------------------------------------------------|
| `pending`       | Dispatch **implementer** subagent                             |
| `implementation`| Dispatch **implementer** subagent (resuming)                  |
| `needs-fix`     | Dispatch **implementer** subagent with verifier's issues      |
| `verification`  | Dispatch **verifier** subagent                                |
| `complete`      | Mode 1: pause for human approval; Mode 2: advance to next step|

## Dispatching the Implementer (Agent tool)

When a step needs implementation work, call `Agent` with `subagent_type: "general-purpose"` and a **self-contained** prompt. **The subagent will have its own isolated context — it will NOT appear in your active session.** This isolation is intentional and required by the architecture.

The subagent has its own fresh context and sees none of your conversation.

Required prompt contents:

- Role statement: "You are the workflow implementer subagent"
- Absolute path to the step file
- Absolute path to `PLAN.md` (for cross-step context)
- Task name and current mode (1 or 2)
- Instruction to invoke `Skill("workflow:implementer")` for the full procedure and follow it
- **If this is a `needs-fix` cycle (status was `needs-fix`):**
  - Copy the "Issues Identified" section from step-N.md Verification section
  - Include exact instructions like: "The verifier reported these issues — fix each one: [PASTE ISSUES]"
  - Include iteration number if applicable
- Instruction to update the step file status to `verification` on success and return a **brief** report (≤150 words) — no code dumps, no full diffs

Template:

```
You are the workflow implementer subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

1. Invoke Skill("workflow:implementer") and follow its procedure exactly.
2. Read the step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
3. Read PLAN.md for context: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/PLAN.md
4. Perform the implementation described in the step file. Make the code changes, run local checks.
5. Update the step file: frontmatter status → "verification", fill in the Implementation section with notes.
6. {if needs-fix: "The previous verifier reported these issues — address each one: <paste issues>"}
7. Return a brief report (under 150 words): what you changed, any blockers, and which files were touched.
   Do NOT paste file contents or full diffs.
```

Run the Agent call in the **foreground** (synchronous, wait for result) — the orchestrator needs the return signal before it can read the updated step file.

**CRITICAL:** "Foreground" means the Agent() call waits for the subagent to finish, NOT that the subagent runs in your active session. The subagent runs in its own completely isolated context. You will see the Agent tool result (subagent's brief report), but NOT the file reads, code changes, or test output — that stays in the subagent's isolated context.

## Dispatching the Verifier (Agent tool)

**CRITICAL: Verifier MUST run as an isolated subagent, not in the orchestrator's active session.**

When a step reaches `verification` status, dispatch as a fresh subagent using `subagent_type: "general-purpose"`.

Template:

```
You are the workflow verifier subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

**CRITICAL EXECUTION RULE:** You are in a completely isolated subagent context. Do NOT rely on orchestrator state. You must independently:
1. Read all necessary files (step file, source code, test files, etc.)
2. Run ALL verification steps: build, tests, manual testing
3. Actually execute code to verify it works (not just code review)

Procedure:
1. Invoke Skill("workflow:verifier") and follow its procedure exactly.
2. Read the step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
3. Read project files and understand what implementer created
4. **Execute verification (not just review):**
   - Build/compile the project
   - Run all relevant test suites
   - Manually test against verification criteria by executing the code
   - Verify everything actually works (not just "looks correct")
5. On PASS: update frontmatter status → "complete", fill in the Verification section with full PASS notes.
6. On FAIL: update frontmatter status → "needs-fix", bump iteration, list specific issues with file/line refs and proof (test output, error messages).
7. Return a brief report (under 150 words): PASS or FAIL, and the key evidence. Include test output summaries.
```

**KEY POINTS:**
- Both use `subagent_type: "general-purpose"`
- Both run in **completely isolated subagent contexts** (not in orchestrator's active session)
- Verifier's isolation is critical: it independently builds, tests, and verifies without relying on implementer's claims
- You receive only the brief report; test output and code reads stay in subagent context

**Why isolation matters:**
- If verifier ran in your active session, you'd see 50 lines of build output, test output, and code reads
- Isolation keeps your conversation clean while verifier does the heavy lifting
- Implementer and verifier never interfere with each other's work

## Reading Back Subagent Results

After an `Agent` call returns:

1. `Read` the step file (only the file — not the source it modified)
2. Check the frontmatter `status` field
3. Update the `PLAN.md` step table entry
4. Decide the next action from the status → action map

**Do not** open any of the source files the subagent touched. If you need to know what changed, the subagent's returned report is the source of truth; if that report is insufficient, dispatch another subagent to summarize — don't read files yourself.

## Mode 1: Step-by-Step with Human Approval

**Fix-Verify Loop Cycle (CRITICAL):**

```
Loop:
  step = first non-complete step
  if none: break

  if step.status in {pending, implementation}:
    Agent(implementer prompt for step)
    read step file → update PLAN table

  if step.status == needs-fix:
    READ "Issues Identified" section from step-N.md
    Agent(implementer prompt for step, with issues from step file)
    read step file → update PLAN table

  if step.status == verification:
    Agent(verifier prompt for step)
    read step file → update PLAN table
    if step.status changed to needs-fix:
      continue loop (implementer runs again with new issues)

  if step.status == complete:
    AskUserQuestion: "Step {N} complete. Approve and continue?"
      approve → continue loop (move to next step)
      request changes → set step.status = needs-fix, continue loop

After all complete:
  PLAN.status = ready-for-review
  AskUserQuestion: final approval
    approve → Skill("workflow:finalize")
    request changes → identify step, set needs-fix, continue loop
```

**Key Fix-Verify Cycle Behavior:**

1. **Implementer creates code** → status = `verification`
2. **Verifier tests code** →
   - If PASS: status = `complete`
   - If FAIL: status = `needs-fix` + "Issues Identified" section
3. **Orchestrator reads step file, sees `needs-fix`** →
   - Reads "Issues Identified" from step-N.md
   - Extracts issues list
   - Dispatches implementer AGAIN with issues in prompt
4. **Implementer fixes** → status = `verification` again
5. **Loop continues** until verifier says PASS
6. **Only when status = `complete` can step advance to next**

## Mode 2: End-to-End with Verification Gates

Same loop, but **no human pause** between steps — only verifier PASS gates advancement, and the human is asked only once at the end when PLAN reaches `ready-for-review`.

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
