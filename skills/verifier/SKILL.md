---
name: workflow:verifier
description: SUBAGENT ONLY - Use when a workflow step is ready for verification, dispatched via workflow:execute skill
execution-context: subagent-only
dispatch-via: workflow:execute
---

# Workflow Verifier Skill

⚠️ **CRITICAL: SUBAGENT ONLY - DO NOT CALL DIRECTLY**

**This skill runs in isolated subagent context and MUST be dispatched by workflow:execute.**

**If you are the main orchestrator:**
- ❌ DO NOT call this skill directly
- ✅ Call `workflow:execute` instead
- ✅ `workflow:execute` will dispatch this skill to an isolated subagent

**If you are a subagent:**
- ✅ You were correctly dispatched by execute
- ✅ Follow the procedure below
- ✅ You own the verification work

---

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

### Step 2B: Verify Docker Setup (if docker-compose.yml exists)

If project uses Docker (has docker-compose.yml):

**1. Check docker-compose.yml exists and is valid:**

```bash
if [ ! -f "docker-compose.yml" ]; then
  echo "⊘ Project does not use Docker (docker-compose.yml not found)"
  # Skip to next step
else
  docker-compose config > /dev/null 2>&1
  if [ $? -ne 0 ]; then
    # FAIL: docker-compose.yml is invalid
    docker-compose config  # Show the error
    # Report error to implementer
  else
    echo "✓ docker-compose.yml is valid"
  fi
fi
```

**2. Start all containers:**

```bash
echo "Starting Docker containers..."
docker-compose up -d

if [ $? -ne 0 ]; then
  # FAIL: docker-compose up failed
  docker-compose logs
  # Stop verification
fi

echo "Waiting for containers to stabilize..."
sleep 5
```

**3. Verify ALL containers are running:**

```bash
echo "Checking container status..."

# Get list of services defined in docker-compose.yml
SERVICES=$(docker-compose config --services)
echo "Expected services: $SERVICES"

# Get list of currently running containers
RUNNING=$(docker-compose ps --services --filter "status=running")
echo "Running services: $RUNNING"

# Check each service is running
for service in $SERVICES; do
  if ! echo "$RUNNING" | grep -q "^${service}$"; then
    echo "✗ Container '$service' is NOT running"
    docker-compose ps  # Show full status
    docker-compose logs "$service" --tail=20  # Show last 20 lines of logs
    # FAIL: Service "$service" failed to start
    docker-compose down
    # Stop verification
  else
    echo "✓ Container '$service' is running"
  fi
done

echo "✓ All required containers are running"
```

**4. Verify no container has exited with error:**

```bash
echo "Checking for exited containers..."

# Check for any containers in Exit or Exited state
EXITED=$(docker-compose ps | grep -E "Exit|Exited" | wc -l)

if [ "$EXITED" -gt 0 ]; then
  echo "✗ Some containers have exited with errors:"
  docker-compose ps | grep -E "Exit|Exited"
  echo ""
  echo "Container logs:"
  docker-compose logs --tail=50
  # FAIL: One or more containers exited with errors
  docker-compose down
  # Stop verification
fi

echo "✓ No containers have exited with errors"
```

**5. Verify service health (optional but recommended):**

```bash
# If your docker-compose.yml has healthchecks, verify them
if docker-compose ps --filter "status=healthy" | grep -q .; then
  echo "Checking healthchecks..."
  UNHEALTHY=$(docker-compose ps --filter "status=unhealthy" | wc -l)
  if [ "$UNHEALTHY" -gt 0 ]; then
    echo "⚠ Some services show unhealthy status (may be normal if still starting)"
    docker-compose ps
  else
    echo "✓ All services report healthy status"
  fi
fi
```

**6. Clean up after verification:**

```bash
echo "Cleaning up Docker environment..."
docker-compose down

if [ $? -ne 0 ]; then
  echo "⚠ Warning: docker-compose down failed"
  # Don't FAIL here, just warn
else
  echo "✓ Docker containers cleaned up"
fi
```

**Document results in step file:**

```markdown
**Docker Verification:**
- ✓ docker-compose.yml is valid
- ✓ All {N} services started successfully
  - Service 1: [name]
  - Service 2: [name]
- ✓ No containers exited with errors
- ✓ All containers cleaned up after verification
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

Read acceptance criteria from step-N.md and test EACH ONE explicitly.

**For each criterion: Test → Record → Mark as ✓ passed or ✗ not passed**

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

**Setup & Build:**
- ✓ npm install successful
- ✓ npm run build successful
- ✓ All migrations applied
- ✓ Tests: 45 passing, 0 failing

**Acceptance Criteria Results:**

For each criterion from step-N.md, mark passed/not passed with evidence:

1. **Criterion: "API POST /users returns 201 status"**
   - Status: ✓ PASSED
   - Evidence: curl -X POST http://localhost:3000/api/users returned 201
   - Details: Response body: {"id": 123, "name": "test"}

2. **Criterion: "User record inserted in database"**
   - Status: ✓ PASSED
   - Evidence: psql query: SELECT * FROM users WHERE id=123 returned record
   - Details: Record has correct name, email fields

3. **Criterion: "Page /users displays user count"**
   - Status: ✗ NOT PASSED
   - Reason: Page shows error "Cannot read property 'length' of undefined"
   - Evidence: curl http://localhost:3000/users returned HTML with error
   - Fix needed: Verifier found template bug - implementer must fix

**Summary**: 2/3 criteria passed, 1 needs fix
```

### Step 7: Files You Work With

**DO modify:**
- {TASK_DIR}/steps/step-{N}.md - Add Verification section with detailed results and notes

**Do NOT modify:**
- Step frontmatter status field (execute manages this)
- progress.json (execute manages this)
- iteration count (execute increments on needs-fix)

---

### Step 8: Report Decision to Execute

Do NOT update step status - execute skill manages all status transitions.

Instead, report your verification result:

**IF all tests passed AND all criteria verified:**

```
✓ Step {N} verified
- All tests passing
- All criteria verified  
- Ready for approval
```

**IF any test failed OR any criterion failed:**

```
✗ Step {N} failed verification
- Test: [name] failed: [reason]
- Criterion: [name] failed: [reason]
- Implementer must fix: [specific items]
```

Execute will read these results and update progress.json accordingly.

---

## Critical Rules

**MUST:**
- ✅ Build project - fail if errors
- ✅ Run all tests - fail if any fail
- ✅ Check migrations ran (don't run them)
- ✅ If docker-compose.yml exists: verify ALL containers running
- ✅ If docker-compose.yml exists: verify NO containers exited
- ✅ Manually test EACH criterion with actual execution
- ✅ For EACH criterion: Mark ✓ passed or ✗ not passed with specific reason
- ✅ Document with proof (curl output, psql results, test names)
- ✅ Report clear PASS or FAIL result
- ✅ Fill Verification section with structured criteria results

**NEVER:**
- ❌ Run migrations yourself
- ❌ Assume tests passed without running
- ❌ Say "code looks correct" without testing
- ❌ Skip manual testing
- ❌ Skip docker-compose validation if it exists
- ❌ Fix issues (report them, implementer fixes)
- ❌ Update step status field (execute manages this)
- ❌ Update progress.json (execute manages this)

---

## What Counts as Proof

**PASS requires proof:**
- ✓ Build succeeded (no errors)
- ✓ Tests succeeded (npm test output)
- ✓ Migrations checked complete
- ✓ API tested: curl output shows correct status/data
- ✓ Database: psql query shows data persisted
- ✓ Page: curl output shows expected content

**Docker verification:**
- ✓ docker-compose config command succeeded (valid YAML)
- ✓ docker-compose ps shows all services with "Up" status (not "Exit" or "Exited")
- ✓ All services from config --services are listed in ps output
- ✓ No containers show exit codes or "Exited" status
- ✓ docker-compose down succeeded (cleanup)

**NOT proof:**
- ✗ "Code looks correct"
- ✗ "Should work"
- ✗ "API probably responds"
- ✗ "Tests should pass"
- ✗ "Docker is running"
- ✗ "Containers started"
- ✗ "docker-compose up succeeded" (without checking all services)
- ✗ "Some containers running" (what about the others?)

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
- ✅ Clear PASS or FAIL result reported
- ✅ Summary reported to execute
