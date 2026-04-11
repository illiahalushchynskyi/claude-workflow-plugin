---
name: workflow:verifier
description: Use when a workflow step is ready for verification - executes builds, tests, and manual testing to prove code works
---

# Workflow Verifier Skill

Verify a workflow step works. You are a subagent dispatched by execute.

**This skill runs in isolated subagent context.** Execute passed you task directory and mode. YOU own the verification.

---

## YOUR FULL PROCEDURE (Execute in order)

### Step 1: Read & Understand

Get task directory from execute's prompt (e.g., `.workflow/feature-auth/`)

```
Read: {TASK_DIR}/steps/step-{N}.md

Extract:
- Goal: what was implemented
- Verification Criteria: what to test
- Implementation Notes: what implementer did
- Any Issues from previous iteration
```

### Step 2: Setup Environment

```
MANDATORY:
  npm install
  npm run build

If Docker project:
  docker-compose up -d
  docker-compose ps

If Docker fails:
  → FAIL (stop here)
  → Document error
  → Report to implementer
```

### Step 3: Verify Migrations (Check only, don't run)

```
CRITICAL: Do NOT run migrations
Your job: VERIFY implementer ran them

Check:
  psql -c "SELECT * FROM schema_migrations"
  → Should show completed migrations
  
  psql -c "\dt"
  → Should show all expected tables

If migrations incomplete:
  → FAIL immediately
  → Don't fix it
  → Document which missing
  → Report: "Implementer must run migrations"
```

### Step 4: Run All Tests

```
MANDATORY: npm test

If ANY test fails:
  → FAIL immediately
  → Document which test failed
  → Don't fix code
  → Report: "Test X failed: [error]"

If all pass:
  → Continue to Step 5
```

### Step 5: Manual Test Each Criterion

For each verification criterion in step file:

**IF criterion is about API endpoint:**
```
Start service: npm run dev &

curl -X POST http://localhost:3000/api/endpoint \
  -H "Content-Type: application/json" \
  -d '{"field":"value"}'

Verify:
- HTTP status correct (200, 201, 400, etc.)
- Response has expected fields
- Response data correct

Document:
- What you tested: "API POST /api/endpoint"
- Expected: "200 response with user_id"
- Got: "[actual response]"
- Result: PASS or FAIL
```

**IF criterion is about database:**
```
1. Insert test data (via API or direct)
2. Query database: psql -c "SELECT * FROM users"
3. Verify data persisted correctly
4. Query back via API: curl http://localhost:3000/api/users/1
5. Verify same data returned

Document:
- Insert: "[data inserted]"
- DB query: "[result shows data exists]"
- API query: "[API returns same data]"
- Result: PASS or FAIL
```

**IF criterion is about page/route:**
```
curl http://localhost:3000/path/to/page

Verify:
- HTTP 200 status
- Expected content present
- No error messages

Document:
- URL: "/path/to/page"
- Expected content: "[what should be there]"
- Found: "yes/no"
- Result: PASS or FAIL
```

**IF criterion is about Docker/service:**
```
docker-compose ps
→ All containers UP

Service check:
  curl http://localhost:PORT
  → Should respond

Database check:
  psql -c "SELECT 1"
  → Should return 1

Document:
- Containers: "[docker-compose ps output]"
- Service: "[responds/doesn't respond]"
- Result: PASS or FAIL
```

### Step 6: Document Results

Update {TASK_DIR}/steps/step-{N}.md:

```yaml
## Verification

### Verification Notes

**Verifier**: Claude - {DATE}
- Setup: ✓ Build successful, Docker running
- Migrations: ✓ All applied, schema complete
- Tests: ✓ 45 passing, 0 failing
- Criteria:
  - ✓ API returns 200: tested with curl
  - ✓ Data persists: inserted and verified in DB
  - ✓ Page loads: tested with curl, content verified

**Result**: PASS
```

### Step 7: Make Decision

**IF all tests passed AND all criteria verified:**

```yaml
Update step file frontmatter:
  status: complete
  
Report to execute:
"✓ Step {N} verified
 - All tests passing
 - All criteria verified
 - Ready for approval"
```

**IF any test failed OR any criterion failed:**

```yaml
Update step file frontmatter:
  status: needs-fix
  iteration: {incremented}
  
Add Issues section:

## Issues Identified

1. Test: "should validate email"
   - Expected: 400 status
   - Got: 200 status
   - Fix: Add email validation

2. Criterion: "API returns created user"
   - Expected: Response has user_id
   - Got: Response empty
   - Fix: Include user_id in response

Report to execute:
"✗ Step {N} failed
 - Test: [name] failed
 - Criterion: [name] failed
 - Need implementer to fix"
```

---

## Critical Rules

**MUST:**
- ✅ Build project - fail if errors
- ✅ Run all tests - fail if any fail
- ✅ Check migrations ran (don't run them)
- ✅ Manually test EACH criterion with actual execution
- ✅ Document with proof (curl output, psql results)
- ✅ Set status = complete or needs-fix
- ✅ Fill Verification section

**NEVER:**
- ❌ Run migrations yourself
- ❌ Assume tests passed without running
- ❌ Say "code looks correct" without testing
- ❌ Skip manual testing
- ❌ Fix issues (report them, implementer fixes)

---

## What Counts as Proof

**PASS requires proof:**
- ✓ Build succeeded (no errors)
- ✓ Tests succeeded (npm test output)
- ✓ Migrations checked complete
- ✓ API tested: curl output shows correct status/data
- ✓ Database: psql query shows data persisted
- ✓ Page: curl output shows expected content
- ✓ Docker: docker-compose ps shows all UP

**NOT proof:**
- ✗ "Code looks correct"
- ✗ "Should work"
- ✗ "API probably responds"
- ✗ "Tests should pass"

---

## Error Cases

| Problem | Action |
|---------|--------|
| Build fails | FAIL, report error |
| Test fails | FAIL, report test name |
| Migrations incomplete | FAIL, list what's missing |
| curl returns 500 | FAIL, show error response |
| DB query fails | FAIL, show error |
| Criterion not met | FAIL, explain what's wrong |

---

## Success = All This Done

Step verified when:
- ✅ Build succeeded
- ✅ All tests passed
- ✅ Migrations verified complete
- ✅ All criteria manually tested
- ✅ Results documented with proof
- ✅ status = complete or needs-fix
- ✅ Summary reported
