---
name: workflow:implementer
description: Use when a workflow step is in implementation phase - executes code changes and updates step tracking
---

# Workflow Implementer Skill

## Overview

The `implementer` skill executes the implementation work for a specific workflow step. It reads the step definition from `step-N.md`, performs the required code changes, and updates the step file with detailed implementation notes and progress tracking.

**This skill is designed to run inside a subagent dispatched by `workflow:execute` via the `Agent` tool.** The orchestrator invokes an `Agent` call whose prompt tells the subagent to load this skill. The subagent has its own isolated context — that is the whole point, and it is what keeps the orchestrator's context clean. Do NOT invoke this skill directly in the main orchestrator conversation; if you do, all the file reads and edits will land in the main context instead of a subagent's.

## When to Use

Use `implementer` when:
- You are a subagent dispatched by the orchestrator with an instruction to load `workflow:implementer`
- A workflow step is in `pending`, `implementation`, or `needs-fix` status
- The orchestrator's prompt has given you the absolute path to the step file

## Your Powers (See CLAUDE.md)

**You have automatic permissions to:**
- ✅ Read any project files
- ✅ Edit and create source files
- ✅ Run build, test, and setup commands
- ✅ Commit code changes
- ✅ Update workflow step files
- ✅ Make implementation decisions independently

**You do NOT need to ask the user for permission** on any of these. Work independently and report your progress.

## Input

The implementer skill:

1. **Reads** `.workflow/TASK_NAME/steps/step-N.md` to understand:
   - Step goal
   - Verification criteria (used to guide implementation)
   - File modification requirements
   - Any previous iteration notes

2. **Checks Mode & Prerequisites** (for Mode 2):
   - If Mode 2: verifies previous step (N-1) has status `complete`
   - If Mode 1: may start immediately after bootstrap or after human approval

3. **Executes Implementation**
   - Makes code changes (create new files, modify existing files)
   - Runs local tests to validate approach
   - Documents decisions and approach

## Implementation Process

### Step 1: Preparation

- Read `step-N.md` file details section
- Identify files to create/modify from "Files to Modify/Create"
- Review verification criteria to understand success conditions
- Check previous iteration notes if this is a fix cycle

### Step 2: Development

- Make code changes using appropriate tools (Edit, Write, Bash)
- Follow project conventions and existing patterns
- Commit frequently to feature branch for history
- Run local tests/checks as you go

### Step 3: Documentation & Status Update

Update `step-N.md` implementation section with:

```markdown
## Implementation

### Files to Modify/Create
[Actual files created/modified with specific line numbers]

### Implementation Notes

**Implementer**: [Your name/Claude - timestamp]
- [Decision about architecture choice]
- [Any blockers encountered and how you resolved them]
- [Code changes summary]
- [Local test results]
- [Anything noteworthy for the verifier]

[Continue adding notes as work progresses]
```

### Step 4: Prepare for Verification

- Update step file frontmatter `status` to `verification`
- Update `iteration` count if fixing previous issues
- Mark timestamp
- Leave clear notes for verifier about what to test

### Step 5: Return a Brief Report

Return a short summary (under ~150 words) to the orchestrator: what you changed, files touched, any blockers, and anything the verifier should focus on. **Do not** paste file contents, full diffs, or long test output — the orchestrator does not need them and they would defeat the purpose of running in an isolated subagent.

## Output

Updates `.workflow/TASK_NAME/steps/step-N.md`:

- **Status**: Changes from `pending` → `implementation` → `verification`
- **Implementation section**: Filled with detailed notes
- **Iteration counter**: Incremented if this is a fix cycle
- **Code changes**: Reflected in project files

Example updated file:

```markdown
---
step_number: 1
name: Add JWT token generation endpoint
status: verification
iteration: 1
created: 2026-04-08 10:30
---

# Step 1: Add JWT token generation endpoint

## Goal
Create `/api/auth/token` POST endpoint that issues JWT tokens to authenticated users

## Verification Criteria
- [ ] POST /api/auth/token responds with 200 on valid credentials
- [ ] Response includes `access_token` field with valid JWT
- [ ] Token contains user ID and role in payload
- [ ] Invalid credentials return 401
- [ ] No TypeScript errors
- [ ] Endpoint is documented in API docs

## Implementation

### Files to Modify/Create
- Create: `backend/src/controllers/AuthController.ts`
- Create: `backend/src/services/TokenService.ts`
- Modify: `backend/src/routes/auth.ts` (lines 1-50)
- Create: `backend/src/types/tokens.ts`

### Implementation Notes

**Implementer**: Claude - 2026-04-08 14:25
- Analyzed existing auth pattern in backend/src/services/AuthService.ts
- Created TokenService with sign() and verify() methods
- Implemented symmetric approach: sign with JWT_SECRET, verify same
- Added type safety with TokenPayload interface
- Tested locally with test request file
- All endpoints returning correct status codes

[Additional notes as needed]

## Verification

### Verification Notes

**Verifier**: [To be filled by verifier]
- [Test results]

**Result**: [PENDING]
```

## Mode-Specific Behavior

### Mode 1 (Step-by-Step)

- Implementer works on step-N
- When complete, updates step-N status to `verification`
- Waits for verifier
- If verifier approves: done for this step, human approves before step N+1
- If verifier reports issues: implementer returns to fix

### Mode 2 (End-to-End)

- **Before starting step N**: Implementer checks that step N-1 has status `complete`
- If step N-1 not complete: waits (orchestrator gates advancement)
- Works on step-N
- When complete, updates step-N status to `verification`
- **Does NOT advance to step N+1 yet**
- Waits for verifier to mark step-N `complete`
- Verifier passing unblocks advancement to step N+1

**Key difference:** Mode 2 requires previous step verification before advancing.

## Error Handling & Recovery

If implementation encounters issues:

1. **Fixable Issue** (e.g., test fails):
   - Document issue in step-N.md
   - Make necessary changes
   - Re-test locally
   - Update verification section
   - Mark step status `verification` for re-test

2. **Blocker** (e.g., architectural question, external dependency):
   - Document in step-N.md **Notes & Escalations** section
   - Update PLAN.md with cross-step note
   - Signal orchestrator/human for guidance

3. **Wrong Step Definition**:
   - Document mismatch in step-N.md
   - Request clarification from human
   - Do NOT proceed to next step

## Verification Criteria as Implementation Guide

The verification criteria define "done". Use them to:

- **Understand intent** — each criterion explains what should work
- **Guide testing** — run local checks for each criterion
- **Verify completeness** — before marking `verification`, check all criteria

Example: if criterion is "API returns 200 on valid input", ensure you test that specific case before asking verifier to test.

## File Management

Implementer should:

- Create files following project conventions
- Use existing patterns (copy from similar files when starting)
- Follow TypeScript/language practices of the project
- Add inline comments for non-obvious code
- Update type definitions when creating new data structures

## Iteration Tracking

If verifier reports issues and sends back for fixes:

1. Update step status back to `implementation`
2. Increment `iteration` counter in frontmatter
3. Add new entry to **History** section:
   ```markdown
   - Iteration 2: [date] - Fix reported issues
   ```
4. Make fixes
5. Return to `verification` status
6. Add notes about what was fixed

## Related Skills

- `workflow:bootstrap` — Created the step definition this skill executes
- `workflow:verifier` — Tests the implementation after status reaches `verification`
- `workflow:execute` — Orchestrates when implementer works (Mode 1 vs Mode 2)
