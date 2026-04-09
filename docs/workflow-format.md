# Workflow File Formats Reference

This document defines the complete file format specifications for workflow tasks. All files use structured markdown with YAML frontmatter and markdown content.

## Directory Structure

```
project-root/
└── .workflow/
    └── TASK_NAME/
        ├── PLAN.md                    # Main plan file
        ├── .workflow-config.json      # Configuration
        └── steps/
            ├── step-1-<name>.md       # Step detail file
            ├── step-2-<name>.md
            └── ... (one per step)
```

## 1. PLAN.md — Main Workflow Plan

**Location:** `.workflow/TASK_NAME/PLAN.md`

**Purpose:** Central documentation of workflow goals, steps, progress, and status.

### Frontmatter (YAML)

```yaml
---
task_name: feature-auth-system
title: Build JWT Authentication System
mode: 1
status: executing
created: 2026-04-08
started: 2026-04-08
completed: null
workflow_type: feature
author: Developer Name
description: |
  Implement complete JWT-based authentication system including
  token generation, validation middleware, and user logout.
  This enables secure API access for frontend applications.
---
```

### Frontmatter Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task_name` | string | Yes | Unique task identifier (alphanumerics, hyphens, underscores) |
| `title` | string | Yes | Human-readable task title |
| `mode` | integer (1\|2) | Yes | Execution mode: 1=step-by-step, 2=end-to-end |
| `status` | string | Yes | Current status: planning, executing, ready-for-review, approved, complete |
| `created` | date (YYYY-MM-DD) | Yes | Creation date |
| `started` | date or null | No | When execution began |
| `completed` | date or null | No | When workflow completed |
| `workflow_type` | string | No | Type: feature, bugfix, refactor, documentation, testing, other |
| `author` | string | No | Primary implementer |
| `description` | string | No | Detailed description of what this accomplishes |

### Markdown Content

Standard markdown sections:

```markdown
# [Task Title]

## Goal

One sentence describing what this task accomplishes.

## Architecture & Approach

2-3 sentences explaining the overall strategy and approach.
Why this design? What are the key decisions?

## Mode Selected

- **Mode 1 (Step-by-Step)**: [Explain if selected]
  Verification and human approval happen after EACH step.
  Use for risky changes requiring high oversight.

- **Mode 2 (End-to-End)**: [Explain if selected]
  All steps execute with verification gates between them,
  single human approval at the end. Use for low-risk changes.

## Steps Overview

| Step | Name | Status | Issues |
|------|------|--------|--------|
| 1 | Add JWT token generation | pending | — |
| 2 | Add validation middleware | pending | — |
| 3 | Add logout endpoint | pending | — |
| 4 | Write comprehensive tests | pending | — |

Status options: pending, implementation, verification, complete, needs-fix

## Overall Progress

Checkbox list tracking completion:

- [ ] Step 1 Complete
- [ ] Step 2 Complete
- [ ] Step 3 Complete
- [ ] Step 4 Complete
- [ ] Human Approval
- [ ] Commit & Merge

## Step Details Location

All step detail files are in: `.workflow/TASK_NAME/steps/`

See the individual step-N-<name>.md files for detailed implementation notes,
verification results, and any issues encountered.

## Notes & Escalations

[Any cross-step notes, blockers, or escalations]

Example:
- Database connection pool needs to be initialized in Step 2
  before tokens are issued in Step 1 (rework step ordering)
- Step 3 blocked by: waiting for Docker environment (external blocker)
```

## 2. step-N-<name>.md — Step Detail File

**Location:** `.workflow/TASK_NAME/steps/step-N-<name>.md`

**Purpose:** Detailed tracking of single step implementation, verification, and iterations.

**Naming:** Files named `step-1-<name>.md`, `step-2-<name>.md`, etc., where:
- Number matches step order in PLAN.md
- Name is URL-safe version of step title (hyphens, lowercase)

Example: `step-1-add-jwt-token-endpoint.md`, `step-2-add-validation-middleware.md`

### Frontmatter (YAML)

```yaml
---
step_number: 1
name: Add JWT Token Generation Endpoint
status: verification
iteration: 1
created: 2026-04-08T10:30:00Z
started: 2026-04-08T10:45:00Z
completed: null
estimated_effort: medium
blocked: false
blocked_reason: null
tags:
  - authentication
  - api
depends_on: []
---
```

### Frontmatter Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `step_number` | integer | Yes | Step sequence (1-based, matches file name) |
| `name` | string | Yes | Human-readable step name |
| `status` | string | Yes | pending, implementation, verification, complete, needs-fix |
| `iteration` | integer | No | Current iteration (incremented on fixes, default=1) |
| `created` | datetime | Yes | ISO 8601 datetime when step was created |
| `started` | datetime or null | No | When implementer started working |
| `completed` | datetime or null | No | When verifier approved (status=complete) |
| `estimated_effort` | string | No | xs, small, medium, large, xl |
| `blocked` | boolean | No | Is step currently blocked? (default=false) |
| `blocked_reason` | string or null | No | Why is step blocked? |
| `tags` | array | No | Tags for categorization (e.g., ["auth", "api"]) |
| `depends_on` | array | No | Step numbers this depends on |

### Markdown Content

Standard markdown sections:

```markdown
# Step N: [Step Name]

## Goal

What this step accomplishes. One paragraph.

Example:
Create a POST endpoint at `/api/auth/token` that generates JWT tokens
for authenticated users, returning tokens with proper expiration and
payload structure for subsequent API calls.

## Verification Criteria

How implementer knows it's done. How verifier knows it's correct.
Listed as clear, testable criteria.

Example:

- [ ] Endpoint creates and returns valid JWT token
- [ ] Token payload contains userId and role fields
- [ ] Token expires after 7 days
- [ ] Endpoint returns 401 on invalid credentials
- [ ] Endpoint returns 400 on missing parameters
- [ ] No TypeScript compilation errors
- [ ] All unit tests for endpoint pass
- [ ] Endpoint documented in API docs

## Implementation

### Files to Modify/Create

List of files that will be created or modified, with line numbers if modifying.

Example:

- Create: `backend/src/services/TokenService.ts`
- Create: `backend/src/types/tokens.ts`
- Modify: `backend/src/routes/auth.ts` (lines 1-50)
- Modify: `backend/src/server.ts` (register route)

### Implementation Notes

**Implementer** updates this section as they work.

Format:

```
**Implementer**: [Name/Claude - timestamp]
- [Decision made about X]
- [File Y created with approach Z]
- [Any blockers encountered]
- [Local test result]
- [Next action or status]
```

Example:

```
**Implementer**: Claude - 2026-04-08 14:25:00
- Analyzed existing AuthService pattern
- Created TokenService with symmetric JWT signing
- Added TypeScript types: TokenPayload, TokenOptions
- Tested locally: token generation works, expiration verified
- All criteria ready for verification
```

## Verification

### Verification Notes

**Verifier** updates this section after testing.

Format:

```
**Verifier**: [Name/Claude - timestamp]
- [Code review result]
- [Build/compile result]
- [Test results]
- [Manual test results]
- [Overall assessment]
```

Example:

```
**Verifier**: Claude - 2026-04-08 15:30:00
- Code review: ✓ Follows project patterns, proper error handling
- Build: ✓ TypeScript compilation successful
- Tests: ✓ All unit tests passing (8 tests)
- Manual: ✓ All criteria verified
  - ✓ POST /api/auth/token returns 200
  - ✓ Response includes valid JWT token
  - ✓ Token payload has userId and role
  - ✓ Expired tokens rejected
  - ✓ Invalid credentials return 401
- Overall: Ready to merge
```

**Result**: [PASS | FAIL]

Example:

```
**Result**: PASS ✓
```

### If Issues Found

If verifier marks FAIL, document issues and fixes:

```
### Issues Found (Iteration 2)

**Issue 1: Missing error handling**
- Problem: TokenService.sign() throws uncaught error on DB failure
- Fix: Added try/catch, returns {error: "database_error"}
- File: backend/src/services/TokenService.ts (lines 18-22)
- Status: Fixed

**Issue 2: Test missing edge case**
- Problem: No test for empty password
- Fix: Added test case "should reject empty password"
- File: backend/src/services/__tests__/TokenService.test.ts
- Status: Fixed

**Re-verification**: 2026-04-08 16:00:00
- Code review: ✓ Issues resolved
- Tests: ✓ All 9 tests passing (1 new test added)
- Result: PASS ✓
```

---

## History

Tracks all iterations of the step:

```markdown
---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → PASS ✓
- Iteration 2: 2026-04-09 - Fix error handling → Verification → PASS ✓
```

## 3. .workflow-config.json — Task Configuration

**Location:** `.workflow/TASK_NAME/.workflow-config.json`

**Purpose:** Configuration settings for this specific workflow task.

### Schema

```json
{
  "task_name": "feature-auth-system",
  "mode": 1,
  "auto_advance": false,
  "verification_strategy": "comprehensive",
  "notification_method": "none",
  "notification_target": null,
  "branch_prefix": "workflow-",
  "commit_on_complete": true,
  "push_on_complete": false,
  "step_approval_required": true,
  "max_retries": 3,
  "timeout_minutes": 60,
  "require_tests": true,
  "require_linting": true,
  "require_documentation": true,
  "slack_webhook": null,
  "github_token": null,
  "custom_hooks": {
    "on_step_complete": null,
    "on_workflow_complete": null
  }
}
```

### Configuration Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `task_name` | string | — | Task identifier (must match PLAN.md) |
| `mode` | int (1\|2) | — | Execution mode |
| `auto_advance` | boolean | false | Auto-advance without human approval (Mode 2 only) |
| `verification_strategy` | string | comprehensive | How strict: minimal, standard, comprehensive, strict |
| `notification_method` | string | none | none, console, email, webhook |
| `notification_target` | string or null | null | Email or webhook URL |
| `branch_prefix` | string | workflow- | Git branch prefix |
| `commit_on_complete` | boolean | true | Auto-commit after workflow completes |
| `push_on_complete` | boolean | false | Push to remote after commit |
| `step_approval_required` | boolean | true | Require human approval per step (Mode 1) |
| `max_retries` | integer | 3 | Max verification retry attempts |
| `timeout_minutes` | integer | 60 | Timeout for agent responses |
| `require_tests` | boolean | true | Require tests for changes |
| `require_linting` | boolean | true | Require linting to pass |
| `require_documentation` | boolean | true | Require docs for changes |
| `slack_webhook` | string or null | null | Slack webhook for notifications |
| `github_token` | string or null | null | GitHub API token |
| `custom_hooks.on_step_complete` | string or null | null | Script to run on step completion |
| `custom_hooks.on_workflow_complete` | string or null | null | Script to run on workflow completion |

## Example: Complete Workflow

### File: `.workflow/feature-auth-system/PLAN.md`

```markdown
---
task_name: feature-auth-system
title: Build JWT Authentication System
mode: 1
status: executing
created: 2026-04-08
started: 2026-04-08
completed: null
workflow_type: feature
author: Developer A
---

# Build JWT Authentication System

## Goal

Implement JWT-based authentication for API access.

## Architecture & Approach

Use symmetric JWT signing with shared secret. Token issued on successful
login with 7-day expiration. Middleware validates all protected routes.

## Mode Selected

- **Mode 1 (Step-by-Step)**: Selected
  Each step receives human approval before proceeding.
  Ensures security review at each checkpoint.

## Steps Overview

| Step | Name | Status | Issues |
|------|------|--------|--------|
| 1 | Add JWT token generation | complete | — |
| 2 | Add validation middleware | verification | — |
| 3 | Add logout endpoint | pending | — |

## Overall Progress

- [x] Step 1 Complete
- [ ] Step 2 Complete
- [ ] Step 3 Complete
- [ ] Human Approval
- [ ] Commit & Merge

## Step Details Location

All step detail files are in: `.workflow/feature-auth-system/steps/`

## Notes & Escalations

None yet.
```

### File: `.workflow/feature-auth-system/steps/step-1-add-jwt-generation.md`

```markdown
---
step_number: 1
name: Add JWT Token Generation Endpoint
status: complete
iteration: 1
created: 2026-04-08T10:00:00Z
started: 2026-04-08T10:15:00Z
completed: 2026-04-08T15:30:00Z
estimated_effort: medium
---

# Step 1: Add JWT Token Generation Endpoint

## Goal

Create POST /api/auth/token endpoint that generates JWT tokens.

## Verification Criteria

- [x] Endpoint returns 200 on valid credentials
- [x] Response includes access_token field
- [x] Token verifiable with JWT_SECRET
- [x] Invalid credentials return 401

## Implementation

### Files to Modify/Create

- Create: backend/src/services/TokenService.ts
- Modify: backend/src/routes/auth.ts

### Implementation Notes

**Implementer**: Claude - 2026-04-08 14:30:00
- Created TokenService with sign() method
- Added /api/auth/token POST endpoint
- All local tests passing
- Ready for verification

## Verification

### Verification Notes

**Verifier**: Claude - 2026-04-08 15:30:00
- Code review: ✓ Good structure
- Build: ✓ No errors
- Tests: ✓ All passing
- Manual: ✓ All criteria met

**Result**: PASS ✓

---

**History**: 
- Iteration 1: 2026-04-08 - Implementation → Verification → PASS ✓
```

## Validation

All files must:

1. **Have valid YAML frontmatter** — parseable as valid YAML
2. **Have required fields** — per schema definitions
3. **Use consistent formatting** — markdown standards
4. **Have valid status values** — per enum definitions
5. **Match file naming** — step-N-*.md format

Use `workflow:bootstrap` to validate on creation.

## Version & Updates

This format specification is Version 1.0.0.

Workflow system is compatible with format version 1.x.

Changes that would require version bump:
- Adding required fields to frontmatter
- Changing status enum values
- Removing sections

Backward-compatible changes:
- Adding optional fields
- Adding new sections
- Clarifying descriptions
