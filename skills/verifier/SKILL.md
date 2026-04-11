---
name: workflow:verifier
description: Test a workflow step and mark complete - runs verification criteria and gates advancement to the next step
---

# Workflow Verifier Skill

## Overview

The `verifier` skill tests and validates a completed workflow step. It reads the verification criteria from `step-N.md`, runs comprehensive tests, and is the **only agent** authorized to mark a step as `complete`. The verifier gates advancement to the next step in both Mode 1 and Mode 2.

## Critical Responsibility

**The verifier is the ONLY agent that can mark a step as `complete`.** This is a critical safety gate:

- Implementer NEVER marks a step complete — only as `verification`
- Verifier checks against verification criteria
- On PASS: marks `complete`, unblocks next step
- On FAIL: marks `needs-fix`, returns to implementer

**This skill is designed to run inside a subagent dispatched by `workflow:execute` via the `Agent` tool.** The orchestrator invokes an `Agent` call whose prompt tells the subagent to load this skill. The subagent has its own isolated context, which keeps test output and code reads out of the orchestrator's conversation. Do NOT invoke this skill directly in the main orchestrator conversation.

## When to Use

Use `verifier` when:
- You are a subagent dispatched by the orchestrator with an instruction to load `workflow:verifier`
- A step has status `verification` in its step-N.md file
- The orchestrator's prompt has given you the absolute path to the step file

## Input

The verifier skill:

1. **Reads** `.workflow/TASK_NAME/steps/step-N.md` to understand:
   - Verification criteria (success conditions)
   - What implementer built (from implementation notes)
   - Files created/modified
   - Any specific test instructions

2. **Reads Project Context**:
   - Project structure and conventions
   - Test setup (test runner, test files)
   - Build/compilation requirements
   - Code review patterns

3. **Accesses Current Code**:
   - Reviews code changes made by implementer
   - Checks for issues, bugs, or deviations

## Verification Process

### Step 1: Preparation

- Read step-N.md verification criteria section
- Identify all criteria that must pass
- Review implementer notes for context and any known issues
- Check what files were modified/created

### Step 2: Manual Code Review

- Read the code changes made by implementer
- Check for:
  - Correctness against verification criteria
  - Consistency with project patterns
  - Type safety (if TypeScript)
  - Error handling
  - Edge cases

### Step 3: Build & Compile

Run compilation/build steps:

```bash
# Example for TypeScript project
npm run build
tsc --noEmit  # Type check without output
```

If build fails: **FAIL** — document error, return to implementer

### Step 4: Run Tests

Execute test suites for affected code:

```bash
# Example: run tests for modified module
npm run test -- backend/src/services/TokenService.ts
npm run test -- --coverage  # Get coverage metrics if available
```

Check:
- All tests passing
- No regressions
- Coverage adequate for changes

### Step 5: Manual Testing

Test against verification criteria. Example criteria verification:

**Criterion:** "POST /api/auth/token responds with 200 on valid credentials"
- **Test:** Call endpoint with valid user credentials → expect 200
- **Check:** Response includes expected fields

**Criterion:** "Token contains user ID and role in payload"
- **Test:** Decode returned JWT → check payload structure
- **Check:** User ID matches request, role is correct

**Criterion:** "Invalid credentials return 401"
- **Test:** Call endpoint with bad password → expect 401
- **Check:** No token returned

### Step 6: Linting & Code Quality

Run linting if available:

```bash
npm run lint
npm run format --check
```

### Step 7: Documentation Check

Verify documentation is present:
- Code comments for complex logic
- Function/type documentation
- API endpoint documentation if applicable
- Updated README if user-facing changes

## Output

Updates `.workflow/TASK_NAME/steps/step-N.md`:

### On PASS

```markdown
## Verification

### Verification Notes

**Verifier**: Claude - 2026-04-08 15:30
- Code review: No issues found
- Build: ✓ No TypeScript errors
- Tests: ✓ All tests passing (8 new tests added)
- Manual testing: ✓ All criteria verified
  - ✓ POST /api/auth/token returns 200 on valid credentials
  - ✓ Token payload contains userId and role
  - ✓ Invalid credentials return 401
  - ✓ Token verified with correct JWT_SECRET
- Linting: ✓ No issues
- Documentation: ✓ Endpoint documented in API docs

**Result**: PASS

---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → PASS ✓
```

Also updates frontmatter:

```yaml
status: complete
```

### On FAIL

```markdown
## Verification

### Verification Notes

**Verifier**: Claude - 2026-04-08 15:35
- Code review: Found issues
  - Missing error handling for database connection failures
  - Type mismatch in TokenPayload interface
- Build: ✓ No TypeScript errors
- Tests: ✗ 2 tests failing
  - Test "should reject empty password" failed
  - Test "should handle expired tokens" not implemented
- Manual testing: ✗ Endpoint returns 500 on edge case

**Result**: FAIL

### If Issues Found

**Issues Identified:**
1. Database connection errors not caught
   - Missing try/catch in TokenService.sign()
   - Need to return 500 with error message
   
2. Empty password test not implemented
   - TokenService should validate password length
   - Test file missing edge case

3. Type mismatch
   - TokenPayload.role should be string | Role enum, currently just string

**Implementer Action Required:**
- Fix try/catch in TokenService.sign() (lines 15-20)
- Add password validation before signing token
- Update TokenPayload type definition

---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → FAIL ✗
  - Issues: [listed above]
  - Status: Awaiting implementer fixes
```

Also updates frontmatter:

```yaml
status: needs-fix
iteration: 2
```

## Mode-Specific Behavior

### Mode 1 (Step-by-Step)

- Verifier tests step-N after implementer marks `verification`
- On PASS: marks `complete`, signal to orchestrator
- Orchestrator pauses → human approves before moving to step N+1
- On FAIL: marks `needs-fix`, returns to implementer (no human pause yet)
- Implementer fixes, re-verifies in same iteration cycle

### Mode 2 (End-to-End)

- **Critical Gate:** Verifier is the ONLY advancement gate between steps
- Implementer completes step-N → marks `verification`
- Verifier tests step-N
- **On PASS:** marks `complete` → implementer can NOW start step N+1
- **On FAIL:** marks `needs-fix` → implementer fixes step-N (does NOT touch step N+1)
- Implementer re-marks `verification` → verifier re-tests
- Cycle repeats until PASS

**Key:** Implementer must wait for verifier's PASS status before advancing steps in Mode 2.

## Verification Checklist Template

Verifier should create a checklist for each step's verification. Example:

```markdown
## Verification Checklist

### Code Review
- [ ] Logic matches step goal
- [ ] Follows project patterns
- [ ] Type safety correct (if TypeScript)
- [ ] Error handling present
- [ ] No obvious bugs

### Build & Compile
- [ ] Builds without errors
- [ ] No TypeScript errors
- [ ] No warnings

### Automated Tests
- [ ] Tests run successfully
- [ ] No regressions
- [ ] Coverage adequate

### Manual Testing
- [ ] ✓ [Criterion 1]
- [ ] ✓ [Criterion 2]
- [ ] ✓ [Criterion 3]

### Code Quality
- [ ] Linting passes
- [ ] Documentation present
- [ ] Comments explain non-obvious code

### Final Decision
- [ ] PASS or FAIL?
```

## Detailed Issue Reporting

When reporting FAIL, be specific:

**Good:**
- "TokenService.sign() doesn't catch database errors on line 18"
- "Test 'should reject empty password' expects error but got success"
- "Type TokenPayload.role is string but should be 'admin' | 'user'"

**Vague (not helpful):**
- "Something's wrong with the auth"
- "Tests are failing"
- "Type issues"

**Implementer uses specific issues to know exactly what to fix.**

## Returning to the Orchestrator

After updating the step file, return a brief report (under ~150 words) to the orchestrator stating PASS or FAIL and the key evidence. Do NOT paste full test output, full diffs, or file contents — the orchestrator does not need them and relies on the step file for details.

## Related Skills

- `workflow:implementer` — Implements the changes this skill verifies
- `workflow:bootstrap` — Created the verification criteria this skill checks
- `workflow:execute` — Orchestrates when verifier works and gates advancement
