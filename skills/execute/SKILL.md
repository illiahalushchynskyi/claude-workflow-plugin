---
name: workflow:execute
description: Orchestrate workflow execution - manages mode selection and step sequencing between implementer and verifier agents
---

# Workflow Execute Skill (Orchestrator)

## Overview

`execute` orchestrates workflow execution by reading `PLAN.md`, determining what work to do next, and **dispatching implementer and verifier work as isolated subagents via the `Agent` tool**. The orchestrator itself never edits project code and never runs tests — it only reads state files, launches subagents, reads their updated state, and pauses for human approval.

This is the core architectural rule: **keeping implementation and verification in separate subagent contexts is what prevents the main conversation from being flooded with file reads, diffs, and test output**. If the orchestrator ever does the work itself, the design has failed.

## When to Use

Use `execute` when:

- A workflow has been bootstrapped (`PLAN.md` and step files exist)
- You're ready to begin or resume implementation
- You want to continue from where the previous session left off

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

When a step needs implementation work, call `Agent` with `subagent_type: "general-purpose"` and a **self-contained** prompt. The subagent has its own fresh context and sees none of your conversation.

Required prompt contents:

- Role statement: "You are the workflow implementer subagent"
- Absolute path to the step file
- Absolute path to `PLAN.md` (for cross-step context)
- Task name and current mode (1 or 2)
- Instruction to invoke `Skill("workflow:implementer")` for the full procedure and follow it
- If this is a `needs-fix` cycle: the specific issues from the verifier's notes
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

Run the Agent call in the **foreground** — the orchestrator needs the return signal before it can read the updated step file.

## Dispatching the Verifier (Agent tool)

Same pattern, with `subagent_type: "general-purpose"` (or the `superpowers:code-reviewer` agent if available and appropriate).

Template:

```
You are the workflow verifier subagent for task {TASK_NAME}, step {N} ({STEP_NAME}).
Mode: {1|2}.

1. Invoke Skill("workflow:verifier") and follow its procedure exactly.
2. Read the step file: {ABSOLUTE_PATH}/.workflow/{TASK_NAME}/steps/step-{N}-{slug}.md
3. Check every verification criterion. Run builds, tests, linters as required.
4. On PASS: update frontmatter status → "complete", fill in the Verification section with PASS notes.
5. On FAIL: update frontmatter status → "needs-fix", bump iteration, list specific issues with file/line refs.
6. Return a brief report (under 150 words): PASS or FAIL, and the key evidence. No full test output dumps.
```

## Reading Back Subagent Results

After an `Agent` call returns:

1. `Read` the step file (only the file — not the source it modified)
2. Check the frontmatter `status` field
3. Update the `PLAN.md` step table entry
4. Decide the next action from the status → action map

**Do not** open any of the source files the subagent touched. If you need to know what changed, the subagent's returned report is the source of truth; if that report is insufficient, dispatch another subagent to summarize — don't read files yourself.

## Mode 1: Step-by-Step with Human Approval

```
Loop:
  step = first non-complete step
  if none: break

  if step.status in {pending, implementation, needs-fix}:
    Agent(implementer prompt for step)
    read step file → update PLAN table

  if step.status == verification:
    Agent(verifier prompt for step)
    read step file → update PLAN table

  if step.status == complete:
    AskUserQuestion: "Step {N} complete. Approve and continue?"
      approve → continue loop
      request changes → set step.status = needs-fix, loop

After all complete:
  PLAN.status = ready-for-review
  AskUserQuestion: final approval
    approve → Skill("workflow:finalize")
    request changes → identify step, set needs-fix, loop
```

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
