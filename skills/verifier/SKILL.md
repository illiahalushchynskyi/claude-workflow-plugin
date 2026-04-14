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

### 1. Read Task File

Read: `{TASK_DIR}/steps/step-{N}.md`

Extract: Goal, Verification Criteria, Implementation Notes

### 2. Setup & Build

```bash
npm install
npm run build
```

Fail immediately if build errors.

### 3. Docker Setup (if docker-compose.yml exists)

If project has docker-compose.yml:

1. Validate config: `docker-compose config`
2. Start containers: `docker-compose up -d && sleep 5`
3. Check all services UP: `docker-compose ps` (all showing "Up")
4. Verify no exited containers: `docker-compose ps` (no "Exited" status)
5. Cleanup: `docker-compose down`

Fail immediately if any step fails.

Document: Docker verification results with service names.

### 4. Migrations Check (if applicable)

Check only—do NOT run migrations.

```bash
psql -c "SELECT * FROM schema_migrations"  # Should show completed migrations
psql -c "\dt"                              # Should show expected tables
```

Fail if incomplete. Don't fix—report what's missing.

### 5. Run Tests

```bash
npm test
```

Fail immediately if any test fails. Don't fix code—report which test.

### 6. Manual Test Each Criterion

For EACH criterion in step-{N}.md:

- **API endpoint**: curl with request, verify status/response
- **Database**: insert data, query, verify persisted
- **Page/route**: curl, verify 200 + content
- **Docker/service**: docker-compose ps, curl health check

Document each with evidence: what tested, expected, got, PASS/FAIL.

### 7. Document Results

Update {TASK_DIR}/steps/step-{N}.md with Verification section:

```markdown
## Verification

**Verifier**: Claude - {DATE}

**Setup & Build:**
- ✓ npm install successful
- ✓ npm run build successful
- ✓ All tests passed: {count}

**Criteria Results:**

1. **Criterion: [name]**
   - Status: ✓ PASSED
   - Evidence: [what tested, result]

2. **Criterion: [name]**
   - Status: ✗ NOT PASSED
   - Reason: [what failed]
   - Evidence: [proof]
```

### 8. Report to Execute

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
- ✅ Build project—fail if errors
- ✅ Run all tests—fail if any fail
- ✅ Check migrations ran (don't run them)
- ✅ Verify docker (if docker-compose.yml exists)
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
