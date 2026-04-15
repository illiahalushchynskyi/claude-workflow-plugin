---
name: workflow:verifier
description: SUBAGENT ONLY - Verify workflow step verification criteria
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Verifier

⚠️ **EXECUTION GUARD: SUBAGENT ONLY**

| Execution Context | Action |
|---|---|
| Running as **subagent** (dispatched by execute) | ✅ Proceed with full skill |
| Running in **main session** directly | ❌ STOP—use `workflow:execute` instead |
| Called without Implementation section | ❌ STOP—implementer must run first |
| Called without step criteria | ❌ STOP—require step-N.md with criteria |

**Pre-entry checks:**
- Am I running as a subagent via Agent() dispatch? **YES** → Continue
- Does step-N.md have Verification Criteria section? **YES** → Continue
- Does step-N.md have Implementation section (from implementer)? **YES** → Continue
- Do I have .workflow-config.json with build/test commands? **YES** → Continue
- **NO to any?** → Report error and exit

This skill is invoked by execute ONLY after implementer completes and status = verification.

---

## Language-Specific Command Reference

Use this table when you encounter different project types:

| Type | Build Command | Test Command | Install Command | Notes |
|------|---|---|---|---|
| `node` | `npm run build` | `npm test` | `npm install` | Check for npm/yarn/pnpm |
| `php` | (varies) | `composer test` or `phpunit` | `composer install` | PHP usually no build step |
| `python` | `python setup.py build` | `pytest` or `python -m pytest` | `pip install -e .` | Check pytest.ini or setup.cfg |
| `rust` | `cargo build --release` | `cargo test` | (auto via cargo) | Cargo manages dependencies |
| `go` | `go build ./...` | `go test ./...` | `go mod download` | Go has no install step |
| `java-maven` | `mvn clean compile` | `mvn test` | (auto via mvn) | Maven central repo |
| `java-gradle` | `gradle build` | `gradle test` | `gradle build` | Gradle wrapper in project |
| `ruby` | (varies) | `rspec` or `rake test` | `bundle install` | Check Gemfile |
| `cpp-cmake` | `cmake --build .` | `ctest` | (auto) | CMake generates makefiles |
| `generic-make` | `make` | `make test` | (varies) | Depends on Makefile |

---

## Procedure

### 1. Load Project Config

**ACTION:** Read `.workflow-config.json` from task directory

**EXTRACT:**
- `projectType` - Language/framework (node, php, python, rust, go, java-maven, java-gradle, ruby, cpp-cmake, generic-make, other)
- `buildCommand` - Exact command to build (e.g., `cargo build`, `go build`, `python setup.py build`)
- `testCommand` - Exact command to run tests (e.g., `pytest`, `cargo test`, `go test ./...`)
- `migrateCommand` - Migration command if applicable (else null)

**USE:** These exact commands from config, NOT guesses or defaults

### 2. Read Task File

Read: `{TASK_DIR}/steps/step-{N}.md`

Extract: Goal, Verification Criteria, Implementation Notes

### 3. Setup & Build

**PROCEDURE:**

1. **Install dependencies (if needed):**
   - Use install command inferred from projectType
   - Reference **Language-Specific Command Reference** table above
   - Example: `npm install`, `composer install`, `pip install -e .`, `go mod download`

2. **Run build:**
   - Use `${buildCommand}` from config.json (NOT guessed)
   - Example: `npm run build`, `cargo build --release`, `python setup.py build`

**FAILURE HANDLING:**

If build fails:
- ❌ STOP immediately
- ❌ Do NOT attempt tests
- Document error: what command, what error output
- Report to execute: build failed, cannot verify

**SUCCESS SIGNAL:**

- ✅ Build command exits with code 0
- ✅ No compiler/build errors in output
- ✅ Build artifacts created (if expected)

### 4. Docker Setup (if docker-compose.yml exists)

If project has docker-compose.yml:

1. Validate config: `docker-compose config`
2. Start containers: `docker-compose up -d && sleep 5`
3. Check all services UP: `docker-compose ps` (all showing "Up")
4. Verify no exited containers: `docker-compose ps` (no "Exited" status)
5. Cleanup: `docker-compose down`

Fail immediately if any step fails.

Document: Docker verification results with service names.

### 5. Migrations Verification (if applicable)

Only check if `migrateCommand` is set in config. Implementer already ran migrations—verify they succeeded.

If database is available (project uses one):
```bash
# PostgreSQL
psql -c "SELECT * FROM schema_migrations"  # Should show completed migrations
psql -c "\dt"                              # Should show expected tables

# MySQL
mysql -e "SELECT * FROM migrations"
mysql -e "SHOW TABLES"
```

For other databases, verify migration artifacts exist and are up-to-date (sqlx_migrations, diesel migrations, alembic versions, etc).

If incomplete or failed: Fail verification and report which migrations are missing or in error state.

### 6. Run Tests

**PROCEDURE:**

1. **Run test command:**
   - Execute `${testCommand}` from config.json (exact command, not guessed)

2. **Check result by projectType:**

| Type | Success Signal | Failure Signal |
|------|---|---|
| `node` | Exit code 0, "passed" in output | Exit code ≠ 0 or failures listed |
| `php` | Exit code 0, "OK" or "passed" in output | Exit code ≠ 0 or FAIL in output |
| `python` | Exit code 0, "passed" in output | Exit code ≠ 0 or FAILED in output |
| `rust` | Exit code 0, "test result: ok" in output | Exit code ≠ 0 or "test result: FAILED" |
| `go` | Exit code 0, "ok" in output | Exit code ≠ 0 or "FAIL" in output |
| `java-maven` | Exit code 0, "BUILD SUCCESS" | Exit code ≠ 0 or BUILD FAILURE |
| `java-gradle` | Exit code 0, no test failures in output | Exit code ≠ 0 or "> Task :test FAILED" |
| `ruby` | Exit code 0, examples passed | Exit code ≠ 0 or failures listed |
| `cpp-cmake` | `ctest` exit code 0 | `ctest` exit code ≠ 0 |
| `generic-make` | `make test` exit code 0 | `make test` exit code ≠ 0 |

**FAILURE HANDLING:**

If ANY test fails:
- ❌ STOP immediately
- ❌ Do NOT proceed to manual criterion testing
- ❌ Do NOT try to fix code (that's implementer's job)
- Document: which test failed, what the error was
- Report to execute: tests failed, cannot verify

### 7. Manual Test Each Criterion

For EACH criterion in step-{N}.md:

- **API endpoint**: curl with request, verify status/response
- **Database**: insert data, query, verify persisted
- **Page/route**: curl, verify 200 + content
- **Docker/service**: docker-compose ps, curl health check

Document each with evidence: what tested, expected, got, PASS/FAIL.

### 8. Document Results

**ACTION:** Update `.workflow/{TASK_NAME}/steps/step-{N}.md` Verification section

**STRUCTURE:**

```markdown
## Verification

**Verifier**: Claude - {DATE}

**Build & Setup:**
- Build command: ${buildCommand}
- Build result: ✓ SUCCESS (or ✗ FAILED: [error])
- Tests command: ${testCommand}
- Tests result: ✓ {N} tests passed (or ✗ FAILED: [which tests])

**Docker Verification (if applicable):**
- ✓ docker-compose.yml validated
- ✓ Services started successfully
- ✓ Health checks passed

**Migration Verification (if applicable):**
- Migration command: ${migrateCommand}
- Result: ✓ All migrations applied (or ✗ Failed: [which migration])
- Evidence: [migration log or schema verification]

**Criteria Verification:**

1. **Criterion: {criterion text}**
   - Status: ✓ PASSED
   - Test: {what you tested}
   - Evidence: {proof - curl output, DB query result, test name, etc.}

2. **Criterion: {criterion text}**
   - Status: ✗ NOT PASSED
   - Reason: {what failed}
   - Evidence: {proof of failure}

[... more criteria ...]

**Summary:**
- Total criteria: N
- Passed: M
- Failed: K
- Overall: ✓ PASS (or ✗ FAIL)
```

**CRITICAL REQUIREMENTS:**
- ✅ Every criterion must have PASS or FAIL status
- ✅ Every criterion must have evidence
- ✅ Build and test results documented
- ✅ Migration results documented (if applicable)
- ✅ NO status field updated (execute manages it)
- ✅ NO progress.json modified

### 9. Signal Completion to Execute

**COMPLETION SIGNAL:** step-{N}.md Verification section is FULLY FILLED

Execute will:
1. Read Verification section
2. Check if all criteria have PASS/FAIL status
3. Count passed vs failed
4. Set progress.json status:
   - All criteria PASSED → status = `complete`
   - Any criterion FAILED → status = `needs-fix`

**HOW IT WORKS:**
- In subagent mode: Agent naturally completes when this step finishes
- Execute sees Verification section filled with results → knows we're done → updates status

---

## Rules - Critical for Success

### MUST DO ✅

| Rule | Why | How to Verify |
|------|-----|---|
| Load .workflow-config.json | Different projects use different commands | Check buildCommand/testCommand from config |
| Build using buildCommand | Must ensure code compiles/builds | Run exact command from config, check exit code 0 |
| Run testCommand from config | Tests validate implementation | Run exact command from config, check exit code 0 |
| Handle language-specific output | Different languages report results differently | Use table above to detect PASS/FAIL by type |
| Test each criterion | Each criterion has testable proof | Test with curl, DB query, file check, etc. |
| Collect evidence | Proof must be concrete, not assumed | Document what you tested and result |
| Verify migrations if set | Data structure changes must work | Check if migrateCommand != null, verify if set |
| Fill Verification section fully | Execute reads this to determine status | Every section filled: build, tests, criteria |
| Report PASS or FAIL for each criterion | No ambiguity about what passed/failed | Each criterion has explicit PASS or FAIL |

### NEVER DO ❌

| Rule | Why | Consequence |
|---|---|---|
| Update step status field | Execute manages all state | Causes sync problems |
| Update progress.json | Only execute modifies this | Causes state corruption |
| Skip manual testing | Automated tests don't cover all requirements | Criteria not validated |
| Fix code | That's implementer's job, not verifier | Blurs responsibilities |
| Assume "should work" | Need concrete proof | Fail step unexpectedly |
| Use hardcoded test command | Projects use different test frameworks | Tests don't run |

## Completion Checklist

Before finishing, verify:

```
✅ Build step completed successfully
✅ All tests passed (testCommand exit code 0)
✅ Migrations verified if migrateCommand set
✅ Docker verified if docker-compose.yml exists
✅ Each criterion tested with concrete evidence
✅ Each criterion has explicit PASS or FAIL status
✅ Verification section FULLY FILLED with:
   - Build and test results
   - Migration results (or "not needed")
   - Criterion verification with evidence
✅ Status field NOT touched
✅ progress.json NOT modified
```

If ANY item unchecked: RETURN to that step, fix it, re-verify

## Evidence Standards

**What counts as evidence (PASS examples):**
- ✓ curl response: `HTTP/1.1 201 Created`
- ✓ Database query: `SELECT * FROM users WHERE id=5` returns row
- ✓ Test output: `12 passed in 1.5s`
- ✓ File exists: `ls -l target/application.jar` shows size > 0
- ✓ Schema check: `\dt` shows expected tables

**What does NOT count (FAIL examples):**
- ✗ "Code looks correct"
- ✗ "Should work fine"
- ✗ "Tests should pass"
- ✗ "Seems good to me"
- ✗ "Likely works"
