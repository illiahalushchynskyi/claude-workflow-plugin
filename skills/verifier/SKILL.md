---
name: workflow:verifier
description: SUBAGENT ONLY - Verify workflow step acceptance criteria
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Verifier

⚠️ **EXECUTION GUARD: SUBAGENT ONLY**

This skill ONLY runs in subagent context via workflow:execute.
DO NOT call this skill directly from main session.

- If in main session → Use `workflow:execute` instead
- If you are a subagent → You were correctly dispatched, proceed

---

## Procedure

### 1. Load Project Config

Read `.workflow-config.json` from task directory. Extract:
- `projectType` - Language/framework (node, php, python, rust, go, java-maven, java-gradle, ruby, cpp-cmake, generic-make, other)
- `buildCommand` - Command to build (e.g., `cargo build`, `go build`, `python setup.py build`)
- `testCommand` - Command to run tests (e.g., `pytest`, `cargo test`, `go test ./...`)
- `migrateCommand` - Migration command if applicable (else null)

### 2. Read Task File

Read: `{TASK_DIR}/steps/step-{N}.md`

Extract: Goal, Verification Criteria, Implementation Notes

### 3. Setup & Build

Run: `${installCommand}` (inferred from projectType) if needed
Then: `${buildCommand}` (from config)

Fail immediately if build errors. Handle language-specific failures (e.g., compile vs. build errors).

Example commands by language:
- **Node:** `npm install` → `npm run build`
- **PHP:** `composer install` → (build often not needed)
- **Python:** `pip install -e .` → `python setup.py build`
- **Rust:** (auto) → `cargo build --release`
- **Go:** `go mod download` → `go build ./...`
- **Java:** (auto) → `mvn clean compile` or `gradle build`

### 4. Docker Setup (if docker-compose.yml exists)

If project has docker-compose.yml:

1. Validate config: `docker-compose config`
2. Start containers: `docker-compose up -d && sleep 5`
3. Check all services UP: `docker-compose ps` (all showing "Up")
4. Verify no exited containers: `docker-compose ps` (no "Exited" status)
5. Cleanup: `docker-compose down`

Fail immediately if any step fails.

Document: Docker verification results with service names.

### 5. Migrations Check (if applicable)

Only check if `migrateCommand` is set in config. Check only—do NOT run migrations.

If database is available (project uses one):
```bash
# PostgreSQL
psql -c "SELECT * FROM schema_migrations"  # Should show completed migrations
psql -c "\dt"                              # Should show expected tables

# MySQL
mysql -e "SELECT * FROM migrations"
mysql -e "SHOW TABLES"
```

For other databases, verify migration artifacts exist (sqlx_migrations, diesel migrations, alembic versions, etc).

Fail if incomplete. Don't fix—report what's missing.

### 6. Run Tests

Run: `${testCommand}` from config

Language-specific test output handling:
- **Node/npm:** Exit code 0, "passed" in output
- **PHP/phpunit:** Exit code 0, "OK" or "passed" in output
- **Python/pytest:** Exit code 0, "passed" in output
- **Rust/cargo:** Exit code 0, "test result:" line
- **Go:** Exit code 0, "ok" in output
- **Java:** Exit code 0, "BUILD SUCCESS" or tests summary
- **Ruby/rspec:** Exit code 0, examples passed

Fail immediately if any test fails. Don't fix code—report which test and why.

### 7. Manual Test Each Criterion

For EACH criterion in step-{N}.md:

- **API endpoint**: curl with request, verify status/response
- **Database**: insert data, query, verify persisted
- **Page/route**: curl, verify 200 + content
- **Docker/service**: docker-compose ps, curl health check

Document each with evidence: what tested, expected, got, PASS/FAIL.

### 8. Document Results

Update {TASK_DIR}/steps/step-{N}.md with Verification section:

```markdown
## Verification

**Verifier**: Claude - {DATE}

**Setup & Build:**
- ✓ ${projectType} project setup successful
- ✓ ${buildCommand} succeeded
- ✓ ${testCommand} passed: {count} tests

**Criteria Results:**

1. **Criterion: [name]**
   - Status: ✓ PASSED
   - Evidence: [what tested, result]

2. **Criterion: [name]**
   - Status: ✗ NOT PASSED
   - Reason: [what failed]
   - Evidence: [proof]
```

### 9. Report to Execute

Report result (do NOT update step status—execute manages it):

**All pass:**
```
✓ Step {N} verified
- All tests passing
- All criteria verified
```

**Any fail:**
```
✗ Step {N} failed verification
- Test: [name] failed: [reason]
- Criterion: [name] failed: [reason]
```

---

## Critical

**MUST:**
- ✅ Load .workflow-config.json and extract projectType, buildCommand, testCommand
- ✅ Build project using buildCommand—fail if errors
- ✅ Run tests using testCommand from config—fail if any fail
- ✅ Handle language-specific test output (don't assume npm format)
- ✅ Check migrations (if migrateCommand is set) - don't run them
- ✅ Verify docker (if docker-compose.yml exists) - language-independent
- ✅ Manually test EACH criterion with evidence
- ✅ Report clear PASS or FAIL

**NEVER:**
- ❌ Update step status
- ❌ Update progress.json
- ❌ Skip manual testing
- ❌ Fix issues (report them)

---

## Proof Required

**PASS needs:**
- ✓ Build succeeded
- ✓ All tests passed
- ✓ Each criterion tested with evidence

**NOT proof:**
- ✗ "Code looks good"
- ✗ "Should work"
- ✗ "Tests should pass"
