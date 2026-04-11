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

### Step 2: Set Up Test Environment & Verify Migrations

**MANDATORY**: Before testing anything:

1. **Install and build project:**
```bash
npm install  # or yarn, or language equivalent
npm run build  # or equivalent
```

2. **Start services:**
```bash
docker-compose up -d  # if using Docker
# Wait for containers to be healthy
docker-compose ps  # verify all running
```

3. **VERIFY migrations were applied (DO NOT RUN MIGRATIONS):**

**IMPORTANT:** Your job is to VERIFY that implementer already ran migrations, not to run them yourself.

Check if migrations exist and were applied:
```bash
# Check if migration files exist
ls -la migrations/ || echo "No migrations directory"

# Verify migrations ran (check migration history)
psql -U user -d database -c "SELECT * FROM schema_migrations;"
# Should show completed migrations, not pending

# Verify schema is complete
psql -U user -d database -c "\dt"  # list tables
# Check against step-N.md expected tables

# If using other ORMs:
sqlite3 db.sqlite3 "SELECT * FROM django_migrations;"  # Django
rails db:migrate:status  # Rails
```

**If migrations are INCOMPLETE or MISSING:**
- ❌ This is a FAIL - implementer didn't run migrations
- Document exact state: which migrations are missing, which tables don't exist
- Do NOT try to fix it - return to implementer with findings

4. **Check environment variables:**
```bash
# Verify .env exists and has required vars
test -f .env && echo ".env exists" || echo "ERROR: .env missing"
```

If environment setup fails: **FAIL** — document exact error and blocker

**CRITICAL RULE: Verify migrations, don't run them. If they're incomplete, FAIL and let implementer fix.**

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

### Step 5: Manual Testing Against Criteria (MANDATORY EXECUTION)

**You MUST manually verify each criterion by ACTUALLY RUNNING the code:**

For each verification criterion in step-N.md:

#### Testing API Endpoints

**If criterion mentions API endpoint (e.g., "POST /api/auth/token responds with 200 on valid credentials"):**

```bash
# 1. Start the service
npm run dev &  # or npm start, or docker run

# 2. Wait for service to be ready
sleep 2
curl http://localhost:3000/health  # or health check endpoint

# 3. Execute the API call with actual data
curl -X POST http://localhost:3000/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass"}'

# 4. Capture and verify response
# - Check HTTP status code (200, 201, 400, 401, 500, etc.)
# - Check response body has expected fields
# - Check data types are correct

# 5. Test edge cases
curl -X POST http://localhost:3000/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"","password":""}'
# Should return 400 or 401 (error case)

# 6. Test with real data if applicable
curl -X GET http://localhost:3000/api/users/123 \
  -H "Authorization: Bearer TOKEN"
# Verify it returns user data or 404 if user doesn't exist
```

#### Testing Database Operations

**If criterion mentions database (e.g., "Data is persisted to database"):**

```bash
# 1. Check migrations ran
psql -U user -d dbname -c "SELECT * FROM schema_migrations;"

# 2. Run the operation that should persist data
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'
# Should return success with new user ID (e.g., 123)

# 3. Verify data actually persisted
psql -U user -d dbname -c "SELECT * FROM users WHERE id = 123;"
# Should show: id | name  | email              | created_at
#             123 | John | john@example.com | 2026-04-11 ...

# 4. Query via API to verify it returns the data
curl http://localhost:3000/api/users/123
# Should return the same data you just inserted
```

#### Testing Web Pages/Routes

**If criterion mentions web pages (e.g., "/dashboard loads and displays user data"):**

```bash
# 1. Start the service
npm run dev &

# 2. Test page loads and has expected content
curl http://localhost:3000/dashboard

# 3. If page requires authentication, include auth header
curl http://localhost:3000/dashboard \
  -H "Authorization: Bearer YOUR_TOKEN"

# 4. Check response contains expected elements
# Example: page should have "Welcome John" text
curl http://localhost:3000/dashboard | grep "Welcome"
# If found, criterion passes

# 5. Check for errors
curl http://localhost:3000/dashboard 2>&1 | grep -i "error"
# Should return nothing (no errors)
```

#### Testing Docker/Services

**If criterion mentions Docker or services:**

```bash
# 1. Start containers
docker-compose up -d

# 2. Verify they're running
docker-compose ps
# All should show "Up"

# 3. Test service is accessible
docker exec postgres_container psql -U user -d dbname -c "SELECT 1"
# Should return: 1

# 4. Verify service responds to requests
curl http://localhost:5432 2>&1 || echo "Database ready (port 5432)"
```

**Document ALL results:**
```markdown
- ✓ Criterion 1: POST /api/auth/token
  - Tested: curl -X POST http://localhost:3000/api/auth/token with valid credentials
  - Expected: 200 response with access_token field
  - Got: 200, response: {"access_token":"eyJ...", "expires_in":3600}
  - Result: PASS
  
- ✓ Criterion 2: Data persists to database
  - Created user via API: POST /api/users with name="John"
  - Verified in DB: psql ... SELECT * FROM users WHERE name='John'
  - Found: 1 row with correct data
  - Queried via API: GET /api/users/1 returns same data
  - Result: PASS

- ✗ Criterion 3: API error handling
  - Tested: POST /api/auth/token with empty password
  - Expected: 400 Bad Request with error message
  - Got: 200 OK (should reject empty password)
  - Result: FAIL
```

**CRITICAL RULE: If you can execute it, you MUST execute it. Do not assume it works.**

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
- Migrations: ✗ FAILED
  - Ran: `npm run migrate`
  - Error: `Migration 003_add_users_table.sql failed: column "email" already exists`
  - Status: Migrations did not complete
  - Impact: Database schema is in inconsistent state
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
  - Docker container started ✓
  - Database accessible ✓
  - API endpoint tested:
    - Tried: `curl -X POST http://localhost:3000/api/auth/token ...`
    - Got: 500 Internal Server Error
    - Error: `database error: column "user_id" does not exist`
    - Cause: Migration failed, schema incomplete
  - Database state:
    - Expected tables: users, tokens, roles
    - Found tables: users, tokens (roles missing)
    - `SELECT * FROM schema_migrations` shows only 2/3 migrations completed

**Result**: FAIL

### Issues Identified (Proof of Failure)

**CRITICAL - Migrations Failed:**
1. Migration 003_add_users_table.sql has syntax error
   - File: migrations/003_add_users_table.sql
   - Error: `column "email" already exists`
   - Cause: Column defined twice (duplicated in migration)
   - Fix: Remove duplicate column definition
   - Proof: `npm run migrate` failed with this exact error

2. Database schema is incomplete
   - Expected tables: users, tokens, roles
   - Actual tables: users, tokens (missing: roles)
   - Verified with: `psql -c "\dt"` shows only 2/3 tables
   - Impact: API endpoints referencing roles table will fail

**Code Issues Found:**
1. TokenService.sign() doesn't validate empty passwords
   - File: backend/src/services/TokenService.ts:18-25
   - Current: Accepts any password, even ""
   - Should: Reject passwords shorter than 8 characters
   - Test failure proves this

2. API endpoint returns 500 on missing database column
   - File: backend/src/routes/auth.ts:42
   - Error: `database error: column "user_id" does not exist`
   - Cause: Migration didn't create user_id column (migration failed)
   - Fix: Fix migration, re-run migrations, then API will work

**Implementer Action Required:**
1. Fix migration syntax error in migrations/003_add_users_table.sql
   - Remove duplicate column definition
   - Test locally: `npm run migrate` should complete without errors
2. Verify all migrations run: `psql -c "SELECT * FROM schema_migrations"` should show all 3
3. Verify schema: `psql -c "\dt"` should show users, tokens, roles tables
4. Fix TokenService.sign() password validation
5. Test API endpoint: `curl http://localhost:3000/api/auth/token` should return valid response (not 500)
6. Run full test suite again: `npm test` should pass

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

## Red Flags - Stop If You See These

If any of these happen, FAIL the step immediately:

**Database/Migrations:**
- ❌ `npm run migrate` failed with error → FAIL (schema incomplete)
- ❌ Migration files aren't created when ORM models changed → FAIL
- ❌ `psql \dt` doesn't show expected tables → FAIL
- ❌ `schema_migrations` table shows incomplete migrations → FAIL
- ❌ Database connection error → FAIL
- ❌ "column does not exist" error from API → FAIL (schema mismatch)

**API/Endpoint Testing:**
- ❌ API endpoint returns 500 error → FAIL (don't accept it)
- ❌ "Could not establish connection" → FAIL
- ❌ API expected but doesn't respond → FAIL
- ❌ Response is empty or malformed → FAIL
- ❌ You assume API works without testing it → FAIL (you MUST curl/test it)

**Code Review Without Execution:**
- ❌ "Code looks correct" without running tests → FAIL
- ❌ "Migrations should work" without running `npm run migrate` → FAIL
- ❌ "API endpoint exists" without calling it with curl → FAIL
- ❌ Skipping manual testing "because tests pass" → FAIL

**What Counts as Proof:**
- ✅ `npm run migrate` completed successfully
- ✅ All tables exist: `psql -c "\dt"` shows them
- ✅ API test result: `curl ... ` returns 200 with correct data
- ✅ Database persistence verified: Data inserted via API, confirmed in DB
- ✅ Test output: `npm test` shows 0 failures
- ✅ Error message captured: Exact error text with file and line number

## Detailed Issue Reporting

When reporting FAIL, be specific with PROOF:

**Good:**
- "Migration failed: `npm run migrate` error: column 'email' already exists in migrations/003_add_users_table.sql"
- "API endpoint returned 500: curl http://localhost:3000/api/auth/token gave error 'column user_id does not exist'"
- "Test 'should reject empty password' expected 401 but got 200 at backend/src/services/__tests__/TokenService.test.ts:45"

**Vague (not helpful):**
- "Something's wrong with the auth"
- "Tests are failing"
- "Migrations might not work"

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
