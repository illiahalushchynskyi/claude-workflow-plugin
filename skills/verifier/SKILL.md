---
name: workflow:verifier
description: Use when a workflow step is ready for verification - executes builds, tests, and manual testing to prove code actually works
---

# Workflow Verifier Skill

## Overview

The `verifier` skill tests and validates a completed workflow step. It reads the verification criteria from `step-N.md`, runs comprehensive tests, and is the **only agent** authorized to mark a step as `complete`. The verifier gates advancement to the next step in both Mode 1 and Mode 2.

## Critical Responsibility

**The verifier is the ONLY agent that can mark a step as `complete`.** This is a critical safety gate:

- Implementer NEVER marks a step complete — only as `verification`
- Verifier **MUST EXECUTE the code** to verify it actually works
- Verifier is NOT a code reviewer — it's a tester
- On PASS: marks `complete` with proof it was tested and works
- On FAIL: marks `needs-fix` with exact evidence of what broke

**MANDATORY EXECUTION RULE:**
You MUST NOT approve code just by reading it. You MUST:
- Build the project (if it doesn't compile → FAIL)
- Run all tests (if any fail → FAIL)
- Manually test each verification criterion by actually running the code
- If something can be tested by execution, you MUST execute it to test it
- Docker/database/service claims must be verified by actually starting and testing the service

**Do NOT say "looks correct" or "should work". Say "I tested it and it works" with proof.**

**This skill is designed to run inside a subagent dispatched by `workflow:execute` via the `Agent` tool.** The orchestrator invokes an `Agent` call whose prompt tells the subagent to load this skill. The subagent has its own isolated context, which keeps test output and code reads out of the orchestrator's conversation. Do NOT invoke this skill directly in the main orchestrator conversation.

## When to Use

Use `verifier` when:
- You are a subagent dispatched by the orchestrator with an instruction to load `workflow:verifier`
- A step has status `verification` in its step-N.md file
- The orchestrator's prompt has given you the absolute path to the step file

## Your Powers (See CLAUDE.md)

**You have automatic permissions to:**
- ✅ Read any project files (source code, tests, configs)
- ✅ Run builds, tests, linters without asking
- ✅ Start Docker containers and services
- ✅ Execute manual tests and verify code works
- ✅ Edit workflow step files to document results
- ✅ Make verification decisions independently

**You do NOT need to ask the user for permission** on any of these. Test independently, verify thoroughly, and report your findings. No pauses, no waiting for approval during testing.

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

**CRITICAL: You MUST actually execute the code. Do not just read it. Every step below is MANDATORY — no skipping any phase.**

### Step 1: Preparation

- Read step-N.md verification criteria section
- Identify ALL criteria that must pass (create a checklist)
- Review implementer notes for context and any known issues
- List all files that were modified/created
- Understand what should work

### Step 2: Set Up Test Environment

**MANDATORY**: Before testing anything:
- Verify you can run the project
- Install/start any dependencies (Docker containers, databases, services)
- Check environment variables are set
- Verify project structure is correct

```bash
# Always do this first
npm install  # or yarn, or language equivalent
# Start services if needed
docker-compose up -d  # if using Docker
# Check setup worked
npm run build  # or equivalent
```

If environment setup fails: **FAIL** — document exact error and blocker

### Step 3: Build & Compile (MANDATORY)

Run compilation/build steps - this is NOT optional:

```bash
# TypeScript/Node projects
npm run build
tsc --noEmit

# Other languages - use your equivalent
cargo build
go build
./gradlew build
```

**CRITICAL: If build fails for ANY reason → FAIL the step immediately.**
- Document exact error message
- Do not try to "work around" it
- Return for implementer to fix

### Step 4: Run Automated Tests (MANDATORY)

Execute ALL test suites:

```bash
# Run complete test suite
npm test

# If specific test framework:
npm run test -- --verbose  # See which tests pass/fail
npm run test -- --coverage  # Verify coverage

# For projects with multiple test suites:
npm run test:unit
npm run test:integration
npm run test:e2e
```

**CRITICAL:**
- All tests MUST pass
- Document exact test output: number passing, number failing
- If ANY test fails → **FAIL** the step
- "Test failures don't matter" is not an option

### Step 5: Manual Testing Against Criteria (MANDATORY)

**You MUST manually verify each criterion actually works:**

For each verification criterion in step-N.md:

**Example: "POST /api/auth/token responds with 200 on valid credentials"**
```bash
# Start the running service first
npm run dev &  # or docker run, or whatever starts the service

# Then actually test it
curl -X POST http://localhost:3000/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass"}'

# Verify: Check response is 200, not 500/400/401
# Verify: Check response has "access_token" field
```

**Example: "Docker setup with database works"**
```bash
# Start containers
docker-compose up -d

# Actually verify database is accessible
docker exec postgres_container psql -U user -d dbname -c "SELECT 1"
# Should return: 1 (not connection error, not "not found")

# Verify migrations ran
docker exec postgres_container psql -U user -d dbname -c "\\dt"
# Should show tables (not empty)

# Run application and test it talks to DB
npm start
# Then curl/test endpoints to verify data persistence
```

**Do NOT just read the code and assume it works.**
- If criterium is "database connection", actually test the connection
- If criterium is "API returns data", actually call the API and check response
- If criterium is "Docker runs", actually start Docker and verify it's healthy

Document results:
```markdown
- ✓ Criterion 1: [what you tested] → [result: working/not working]
- ✓ Criterion 2: [proof of execution]
- ✗ Criterion 3: [error message showing it failed]
```

### Step 6: Linting & Code Quality

Run linting if available (this is lower priority than test execution):

```bash
npm run lint
npm run format --check
```

If linting fails but tests pass: PASS (with note to implementer), don't fail the whole step

### Step 7: Documentation Check

Verify documentation is present:
- Code comments for complex logic
- Function/type documentation
- API endpoint documentation if applicable
- Updated README if user-facing changes

If doc is minimal: PASS (prefer working code to perfect docs), note in report

## Output

Updates `.workflow/TASK_NAME/steps/step-N.md`:

### On PASS

All phases completed successfully with evidence that code actually works:

```markdown
## Verification

### Verification Notes

**Verifier**: Claude - 2026-04-08 15:30
- Environment: ✓ Setup successful (Node 18, npm 8, Docker running)
- Build: ✓ npm run build completed with no errors
- Tests: ✓ All tests passing
  - 45 total tests
  - 8 new tests added for this step
  - 0 failures
  - Coverage: 92%
- Manual testing: ✓ All criteria verified by actual execution
  - ✓ POST /api/auth/token returns 200 on valid credentials
    - Tested: `curl -X POST http://localhost:3000/api/auth/token ...`
    - Response: `{"access_token":"eyJ...", "expires_in":3600}`
  - ✓ Token payload contains userId and role
    - Decoded token: `{userId:"123", role:"admin"}`
  - ✓ Invalid credentials return 401
    - Tested with wrong password: returned 401 Unauthorized
  - ✓ Token verified with correct JWT_SECRET
    - verify() method confirmed token signature
  - ✓ Docker container started and database accessible
    - `docker exec postgres_container psql ... SELECT 1` returned 1
    - Tables exist: users, tokens, roles (verified with \dt)
- Linting: ✓ No issues (ESLint clean)
- Documentation: ✓ Endpoint documented in API docs

**Result**: PASS

---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → PASS ✓
```

**PROOF that code works:**
- Build output (no errors)
- Test output (all passing)
- Manual test results (curl responses, database queries, container status)
- Not just "reviewed code and it looks correct"

Also updates frontmatter:

```yaml
status: complete
```

### On FAIL

Document exact proof that code doesn't work - not just opinions:

```markdown
## Verification

### Verification Notes

**Verifier**: Claude - 2026-04-08 15:35
- Environment: ✓ Setup successful
- Build: ✓ Compilation passed
- Tests: ✗ FAILED (2 tests failing out of 47)
  - Test "should reject empty password" FAILED
    - Expected: 401 Unauthorized
    - Got: 200 OK (test expected rejection)
    - File: backend/src/services/__tests__/TokenService.test.ts:45
  - Test "should handle expired tokens" FAILED
    - Expected: token verification to fail on expired token
    - Got: token verification succeeded (no expiry check)
    - File: backend/src/services/__tests__/TokenService.test.ts:89
- Manual testing: ✗ CRITICAL FAILURES
  - Docker container failed to start
    - Command: `docker-compose up -d`
    - Error: `ERROR: yaml parsing error in 'docker-compose.yml' line 5: mapping values are not allowed here`
    - Docker containers NOT running
  - Database not accessible
    - Tried: `docker exec postgres_container psql -U user -d dbname -c "SELECT 1"`
    - Error: `Error: No such container: postgres_container`
    - **Cannot test API because no database available**
  - API endpoint unreachable
    - Tried: `npm start` → returned error
    - Error output: `ENOENT: no such file or directory, open '.env'`
    - Missing .env file configuration

**Result**: FAIL

### Issues Identified (Proof of Failure)

**CRITICAL - Setup Broken:**
1. Docker Compose syntax error in docker-compose.yml (line 5)
   - Current: [paste the problematic line]
   - Expected: Valid YAML
   - Impact: Container won't start, database inaccessible

2. Missing .env file
   - Tried to start: `npm start`
   - Error: `ENOENT: no such file or directory, open '.env'`
   - Need: `.env` with DATABASE_URL and other required vars
   - Impact: Application cannot start

**Code Issues Found:**
1. TokenService.sign() doesn't validate empty passwords
   - File: backend/src/services/TokenService.ts:18-25
   - Current: Accepts any password, even ""
   - Should: Reject passwords shorter than 8 characters
   - Test failure proves this

2. No token expiry checking
   - File: backend/src/services/TokenService.ts
   - Current: No expiration field in JWT payload
   - Should: Include exp claim, verify on token use
   - Test failure proves this

**Implementer Action Required:**
1. Fix docker-compose.yml syntax (line 5 - check YAML formatting)
2. Create .env file with required variables (copy from .env.example or docs)
3. Add password validation to TokenService.sign() - reject if < 8 chars
4. Add token expiry: include exp in JWT payload, verify in token validation
5. Test Docker setup: `docker-compose up -d && docker-compose ps` should show containers running

---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → FAIL ✗
  - Status: Awaiting implementer fixes
  - Cannot proceed until: Docker runs, database accessible, all tests pass
```

Also updates frontmatter:

```yaml
status: needs-fix
iteration: 2
```

**KEY CHANGES:**
- **Before:** "Missing error handling" (vague opinion)
- **After:** "Tried to start with `npm start` → got `ENOENT: no such file .env`" (proof)

- **Before:** "Tests failing" (no details)
- **After:** "Test 'should reject empty password' expected 401, got 200 (line 45)" (exact failure)

- **Before:** "Docker doesn't work" (untested)
- **After:** "`docker-compose up -d` failed with YAML error on line 5` (proof of failure)

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

After updating the step file, return a brief report (under ~150 words) to the orchestrator stating PASS or FAIL and the key evidence. Include:
- **On PASS:** Build passed, all tests passed (with counts), manual testing completed
- **On FAIL:** Exact command that failed, exact error message, what's broken

Do NOT paste full test output dumps — summarize. But DO include enough evidence that someone reading the report would understand why you passed/failed it.

Example:
- ✅ GOOD: "Docker setup: `docker-compose up -d` succeeded, containers running, database accessible. 45 tests passed. Manual testing: verified all 6 criteria work correctly."
- ❌ BAD: "Looks good, Docker should work"
- ❌ BAD: "[100 lines of test output]"

## Related Skills

- `workflow:implementer` — Implements the changes this skill verifies
- `workflow:bootstrap` — Created the verification criteria this skill checks
- `workflow:execute` — Orchestrates when verifier works and gates advancement
