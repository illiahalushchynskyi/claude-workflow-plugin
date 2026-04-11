---
name: workflow:verifier
description: Use when a workflow step is ready for verification - executes builds, tests, and manual testing to prove code works
---

# Workflow Verifier Skill

Verify a workflow step works correctly. This skill runs in an isolated subagent dispatched by execute.

## When to Use

Use this skill when:
- You are a subagent dispatched by workflow:execute
- Step status is `verification`
- execute provided full task context in the prompt

## Input

From execute's Agent() prompt:
- Step file path
- PLAN.md path
- Task name and mode

## Output

- step-N.md updated with:
  - status = "complete" (if PASS) or "needs-fix" (if FAIL)
  - Verification section filled with results
  - Issues Identified (if FAIL)
- Brief report returned to execute

---

## YOUR PROCEDURE

### Step 1: Understand Criteria

```
Read step file:
- Goal: what was implemented
- Verification Criteria: what to test
- Implementation Notes: what implementer did
```

### Step 2: Setup Environment

```
MANDATORY:
  npm install
  npm run build (or equivalent)
  
If Docker:
  docker-compose up -d
  docker-compose ps (verify all running)
  
Setup check: PASS if no errors
```

### Step 3: Verify Migrations (CHECK ONLY, DON'T RUN)

```
CRITICAL: You do NOT run migrations
Your job: VERIFY implementer already ran them

Check:
  psql -c "SELECT * FROM schema_migrations"
  → Should show COMPLETED migrations
  
  psql -c "\dt"
  → Should show expected tables

If migrations incomplete:
  → FAIL (implementer didn't do their job)
  → Don't try to fix it
  → Document which migrations missing
```

### Step 4: Run All Tests

```
MANDATORY - All tests must pass:
  npm test
  
If ANY test fails:
  → FAIL immediately
  → Document which test
  → Don't fix it, report to implementer
```

### Step 5: Manually Test Each Criterion

For each verification criterion in step file:

**IF criterion mentions API endpoint:**
```
curl -X POST http://localhost:3000/api/endpoint \
  -H "Content-Type: application/json" \
  -d '{"field":"value"}'

Verify:
- HTTP status code (200, 201, 400, 401, etc.)
- Response has expected fields
- Response data is correct

Document:
- What you tested
- Expected result
- Actual result
- PASS or FAIL
```

**IF criterion mentions database:**
```
1. Insert data via API or curl
2. Query database: psql -c "SELECT * FROM table"
3. Verify data persisted correctly
4. Query back via API
5. Verify same data returned

Document:
- INSERT command used
- Query result
- PASS or FAIL
```

**IF criterion mentions page/route:**
```
curl http://localhost:3000/page-path

Verify:
- Page loads (200 status)
- Expected content present
- No error messages

Document:
- URL tested
- Content found or not
- PASS or FAIL
```

**IF criterion mentions Docker/services:**
```
docker-compose ps
→ All containers UP

curl http://localhost:PORT
→ Service responds

psql -c "SELECT 1"
→ Database accessible

Document:
- Containers running
- Services responding
- PASS or FAIL
```

### Step 6: Document Results

Update step-N.md Verification section:

```markdown
## Verification

### Verification Notes

**Verifier**: Claude - {DATE}
- Setup: ✓ npm build, Docker running
- Migrations: ✓ All applied, tables present
- Tests: ✓ All passing (25/25)
- Manual testing:
  - ✓ Criterion 1: [tested], [result]
  - ✓ Criterion 2: [tested], [result]
  - ✓ Criterion 3: [tested], [result]

**Result**: PASS
```

### Step 7: Decision - PASS or FAIL

**IF all tests pass AND all criteria verified:**
```
✅ PASS

Update step-N.md:
- status = "complete"
- Fill Verification Notes with PASS

Return to execute:
"All tests passing, criteria verified"
```

**IF any test fails OR any criterion fails:**
```
❌ FAIL

Update step-N.md:
- status = "needs-fix"
- Increment iteration

Document issues:

## Issues Identified

1. Criterion: {name}
   - Problem: {exact problem}
   - Evidence: {how you found it}
   - Fix needed: {what implementer should do}

2. Test: {test name}
   - Expected: {expected}
   - Got: {actual}
   - Error: {exact error}

Return to execute:
"Criterion X failed: [specific issue]"
```

---

## Critical Rules

**MUST DO:**
- ✅ Run `npm run build` - fail if errors
- ✅ Run `npm test` - fail if any test fails
- ✅ Check migrations ran (don't run them)
- ✅ Manually test EACH criterion
- ✅ Document results with proof
- ✅ Set status = "complete" or "needs-fix"
- ✅ Fill Verification section

**NEVER DO:**
- ❌ Run migrations yourself
- ❌ Skip manual testing
- ❌ Assume tests passed without running them
- ❌ Say "looks correct" without testing
- ❌ Fix issues - report them and let implementer fix

---

## What Counts as Proof

**PASS requires:**
- ✅ Build command succeeded
- ✅ Test command succeeded (output count)
- ✅ Migrations checked and complete
- ✅ Each criterion manually tested
- ✅ API: curl response with status code
- ✅ Database: query results showing data
- ✅ Page: content verified present
- ✅ Docker: containers listed as UP

**NOT proof:**
- ❌ "Code looks correct"
- ❌ "Tests should pass"
- ❌ "Migrations probably ran"
- ❌ "API endpoint exists"

---

## Error Cases

| Problem | Action |
|---------|--------|
| Build fails | FAIL, report error message |
| Test fails | FAIL, report test name and error |
| Migration incomplete | FAIL, don't run it, list what's missing |
| API returns 500 | FAIL, show response |
| Database query fails | FAIL, show error |
| Criterion not met | FAIL, explain what failed |

---

## Success Criteria

Step verified when:
- ✅ Build succeeded
- ✅ All tests passed
- ✅ Migrations verified complete
- ✅ All criteria manually tested
- ✅ Results documented
- ✅ step-N.md status = "complete"
- ✅ Report provided to execute
