---
name: workflow:finalize
description: Use when a workflow is approved and ready to finalize - creates completion commit and generates project documentation
---

# Workflow Finalize Skill

## Overview

The `finalize` skill completes a workflow by:
1. **Creating final commit** with comprehensive summary of all changes
2. **Updating PLAN.md** to mark workflow as `complete`
3. **Generating project documentation** with architecture, setup, and API/feature reference

Documentation output is multi-file for easy navigation:
- `README.md` — Project overview and quick-start
- `ARCHITECTURE.md` — Overall logic and system design  
- `SETUP.md` — Installation, configuration, how to run
- `TESTING.md` — Test strategy and how to run tests
- `routes/` or `endpoints/` directory — Individual endpoint/route documentation (one file per endpoint)
- `API.md` or `FEATURES.md` — Summary index

Finalize is called only after:
- All steps have status `complete` in their step-N.md files
- Verifier has approved all steps
- Human has given final approval
- PLAN.md status is `approved`

## When to Use

Use `finalize` when:

- All workflow steps are verified and approved
- PLAN.md status shows `approved`
- You're ready to merge the workflow changes
- Execute skill signals to call finalize

## Input

The finalize skill:

1. **Reads** `.workflow/TASK_NAME/PLAN.md`:
   - Task name, title, description
   - Mode used (1 or 2)
   - Overall progress summary
   - Start and end dates

2. **Reads** All `.workflow/TASK_NAME/steps/step-N.md` files:
   - Step names and goals
   - Files created/modified
   - Implementation notes
   - Verification results
   - Any iteration history

3. **Analyzes Git State**:
   - Checks current feature branch (should be `workflow-TASK_NAME`)
   - Determines files changed since workflow started
   - Generates commit summary

4. **Reads Config**:
   - `.workflow/TASK_NAME/.workflow-config.json`
   - Commit preferences, push settings, project type

## Finalization Process

### Step 1: Gather Information

- Read PLAN.md front matter (created, started, completed dates)
- Read all step files to extract:
  - Files created per step
  - Files modified per step
  - Key decisions from implementation notes
  - Verification results
  - Any issues encountered and fixed

Example aggregation:

```
Step 1: Add JWT token generation endpoint
  Files Created: backend/src/services/TokenService.ts, backend/src/types/tokens.ts
  Files Modified: backend/src/routes/auth.ts
  Key Notes: Symmetric JWT signing approach, added type safety
  Verified: PASS

Step 2: Add token validation middleware
  Files Created: backend/src/middleware/tokenAuth.ts
  Files Modified: backend/src/server.ts
  Key Notes: Integrated with global middleware stack
  Verified: PASS

...
```

### Step 2: Generate Commit Summary

Build multi-paragraph commit message:

```
[task-type]: [Task Title]

[One-sentence description of overall goal]

## Workflow Summary

Mode: [1|2] (Step-by-Step|End-to-End)
Status: Complete
Duration: [started] to [completed]
Steps: [N] total, [N] verified

## Changes by Step

### Step 1: [Name]
- Goal: [One-sentence goal]
- Created: [files]
- Modified: [files]
- Key Decisions: [summary from implementation notes]
- Verification: ✓ PASS

### Step 2: [Name]
- Goal: [One-sentence goal]
- Created: [files]
- Modified: [files]
- Key Decisions: [summary from implementation notes]
- Verification: ✓ PASS

[... repeat for each step ...]

## Testing Summary

All steps verified against criteria:
- [✓ Criterion 1]
- [✓ Criterion 2]
- [... all criteria listed ...]

## Notes

[Any significant blockers, architectural decisions, or known limitations]
[Any deviations from original plan]

Co-Authored-By: Workflow System <workflow@claude.ai>
```

### Step 3: Verify Branch State

- Confirm on feature branch: `workflow-TASK_NAME`
- If not: print error and stop
- Check if any changes pending (should be none — all committed during implementation)

### Step 4: Create Commit

```bash
# Stage workflow metadata files (should already be committed)
git add .workflow/TASK_NAME/

# Create commit with generated message
git commit -m "[generated comprehensive message]"
```

Commit will include:
- All code changes from step implementations
- Updated step-N.md files with verification results
- Updated PLAN.md
- .workflow-config.json

### Step 5: Update PLAN.md

Update PLAN.md frontmatter:

```yaml
status: complete
completed: YYYY-MM-DD
```

Add completion note to end:

```markdown
## Completion Summary

Workflow completed successfully on [date].

Final Status:
- All [N] steps verified ✓
- Total iterations: [count of fix cycles if any]
- Issues found and fixed: [count]
- Human approvals: [count]

All changes committed in workflow completion merge commit.
See git history for full step-by-step progression.
```

### Step 6: Output Summary

Print to user:

```
✓ Workflow Complete!

Workflow: feature-auth-system
Mode: 1 (Step-by-Step)
Completed: 2026-04-08 16:45

Summary:
  ✓ 4 steps completed and verified
  ✓ 8 files created, 12 files modified
  ✓ 0 issues found in verification
  ✓ 1 fix cycle applied to step 3

Commit: workflow: initialize feature-auth-system
  - Added JWT authentication system
  - 8 new files, 12 modified
  - All verification criteria passing

Next Steps:
  1. Review changes: git log --stat
  2. Run full test suite: npm test
  3. Create pull request for code review
  4. Merge when approved
```

### Step 7: Generate Project Documentation

After committing, create comprehensive project documentation:

#### 7.1: Determine Project Type

Analyze the files created/modified to identify project type:
- **API**: HTTP routes/endpoints → create `docs/API.md` + `docs/routes/` structure
- **Library**: Exported functions/classes → create `docs/REFERENCE.md`  
- **Application**: Features/pages → create `docs/FEATURES.md` + `docs/features/` structure
- **CLI Tool**: Commands → create `docs/COMMANDS.md`

#### 7.2: Create Core Documentation Files

**Location**: Create in `docs/PROJECT_NAME/` or `PROJECT_DOCS/` directory

**File: `README.md`** (Overview & Quick Start)
```markdown
# [Project Name]

[One paragraph overview]

## Quick Start

1. Installation/Setup: [steps from SETUP.md]
2. Example usage: [minimal working code]
3. Next: [Where to go from here - links to other docs]

## Directory Structure

[Brief overview of key directories]

## Key Features

- [Feature 1]
- [Feature 2]

## Documentation

- [SETUP.md](./SETUP.md) — Installation and configuration
- [ARCHITECTURE.md](./ARCHITECTURE.md) — System design and logic
- [API.md](./API.md) or [FEATURES.md](./FEATURES.md) — Complete reference
- [TESTING.md](./TESTING.md) — Test strategy and how to run
```

**File: `ARCHITECTURE.md`** (System Design)
```markdown
# Architecture

## Overview

[Description of overall system logic, main components, how they interact]

## Data Flow

[Diagram or description of how data moves through system]

## Key Components

### [Component 1 Name]
- **Purpose**: [What it does]
- **Key Files**: [src/component1.ts, src/component1.test.ts]
- **Dependencies**: [What it depends on]

### [Component 2 Name]
[Same structure]

## Design Decisions

- **Decision 1**: [Why we chose X over Y]
- **Decision 2**: [Technical reasoning]

## Technology Stack

- [Tech]: [Purpose]
- [Tech]: [Purpose]
```

**File: `SETUP.md`** (Installation & Running)
```markdown
# Setup & Installation

## Prerequisites

- [Requirement 1]: [version/details]
- [Requirement 2]: [version/details]

## Installation

\`\`\`bash
# Clone and install
git clone [repo]
cd [project]
npm install  # or equivalent for your tech stack
\`\`\`

## Configuration

1. **Environment Variables**
   - Create `.env` file
   - Required: [VAR1, VAR2]
   - Optional: [VAR3]

2. **Database Setup** (if applicable)
   \`\`\`bash
   npm run migrate
   npm run seed  # optional
   \`\`\`

## Running the Project

### Development

\`\`\`bash
npm run dev
# Server runs on [port/URL]
\`\`\`

### Production

\`\`\`bash
npm run build
npm start
\`\`\`

## Troubleshooting

- **Issue 1**: [Solution]
- **Issue 2**: [Solution]
```

**File: `TESTING.md`** (Test Strategy)
```markdown
# Testing

## Test Strategy

[Overall approach to testing: unit tests, integration tests, e2e tests, etc.]

## Running Tests

### All Tests
\`\`\`bash
npm test
\`\`\`

### Specific Suite
\`\`\`bash
npm test -- [pattern]
\`\`\`

### Watch Mode
\`\`\`bash
npm test -- --watch
\`\`\`

## Test Coverage

\`\`\`bash
npm run test:coverage
\`\`\`

Current coverage:
- Overall: [X]%
- Critical paths: [X]%

## Writing Tests

[Brief guide on test structure and patterns used in project]

## CI/CD

[Description of automated testing in CI pipeline if applicable]
```

#### 7.3: Create API/Routes/Features Documentation

**For API projects** → Create `docs/routes/` or `docs/endpoints/`:

**File: `docs/API.md`** (Index)
```markdown
# API Reference

[Overview of API structure and base URL]

## Endpoints

### Authentication
- [POST /auth/login](./routes/auth-login.md)
- [POST /auth/logout](./routes/auth-logout.md)
- [POST /auth/refresh](./routes/auth-refresh.md)

### Users
- [GET /users](./routes/users-list.md)
- [GET /users/:id](./routes/users-get.md)
- [POST /users](./routes/users-create.md)

[... organize by logical grouping ...]

## Rate Limiting

[If applicable: limits and headers]

## Authentication

[Required auth headers/methods]

## Error Codes

[Common error responses]
```

**File: `docs/routes/[endpoint-name].md`** (Individual Route)
```markdown
# [HTTP METHOD] /[endpoint-path]

[One-sentence description]

## Description

[Detailed description of what this endpoint does]

## Authentication

[Required? What type?]

## Parameters

### Path Parameters
- `id` (string, required): Description

### Query Parameters
- `sort` (string, optional): Description (default: "created")
- `limit` (number, optional): Description (default: 10, max: 100)

### Request Body
\`\`\`json
{
  "name": "string, required",
  "email": "string, required, unique",
  "role": "string, optional"
}
\`\`\`

## Response

### Success (200 OK)
\`\`\`json
{
  "id": "uuid",
  "name": "string",
  "email": "string",
  "created_at": "ISO 8601 timestamp"
}
\`\`\`

### Error (400 Bad Request)
\`\`\`json
{
  "error": "invalid_email",
  "message": "Email already exists"
}
\`\`\`

## Database Changes

- **Table**: users
- **Operation**: INSERT
- **Fields Changed**: name, email, role
- **Constraints**: email must be unique
- **Triggers**: None

## Examples

### cURL
\`\`\`bash
curl -X POST http://localhost:3000/users \\
  -H "Authorization: Bearer TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"name": "John", "email": "john@example.com"}'
\`\`\`

### JavaScript/Node
\`\`\`js
const response = await fetch('http://localhost:3000/users', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer TOKEN',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'John',
    email: 'john@example.com'
  })
});
const user = await response.json();
\`\`\`

## Related Endpoints

- [GET /users/:id](./users-get.md)
- [PATCH /users/:id](./users-update.md)
```

#### 7.4: Extract Documentation Content from Step Files

For each step, extract:
- Files created/modified with purpose
- Key decisions and implementation notes → goes into ARCHITECTURE.md
- Configuration or setup changes → goes into SETUP.md
- New endpoints/routes → generate individual route documentation files
- Test additions → summarize in TESTING.md

### Step 8: Optional: Push to Remote

If `.workflow-config.json` has `"push": true`:

```bash
git push origin workflow-TASK_NAME
```

Else: just commit locally and document it.

## Documentation Generation Guidelines

### Multi-File Structure

**Why**: One large documentation file is hard to navigate. Separate files allow users to find specific information quickly.

Structure by concern:
- **Setup/Running** → SETUP.md
- **Testing** → TESTING.md
- **Architecture/Design** → ARCHITECTURE.md
- **Individual APIs/Routes** → `routes/endpoint-name.md`
- **Overview** → README.md

### API Endpoint Documentation Best Practices

Each endpoint file should include:

1. **Method & Path** (H1 heading with `[METHOD] /path`)
2. **One-sentence description** (below heading)
3. **Description section** (detailed explanation)
4. **Authentication** (required or not)
5. **Parameters** (path, query, body - in separate subsections)
6. **Response section** with:
   - Success response (with HTTP status)
   - Common error responses
   - Example JSON
7. **Database changes** (what gets stored/modified)
8. **Examples** (cURL, JavaScript/Node, Python, etc.)
9. **Related endpoints** (links to similar routes)

### Detecting Project Type

Look for patterns in created/modified files:

| Pattern | Type |
|---------|------|
| `src/routes/`, `app.post()`, `router.get()` | API |
| `src/lib/`, exported functions, `index.ts` | Library |
| `src/pages/`, components, layouts | Application/SPA |
| `bin/cli.js`, command parsing | CLI Tool |
| `src/utils/`, helper functions | Utilities |

For **hybrid projects** (e.g., API + frontend):
- Create `docs/API.md` for backend
- Create `docs/FEATURES.md` for frontend features
- Cross-link them in `docs/README.md`

## Commit Message Format

The finalize skill generates commits with this structure:

```
[TYPE]: [TITLE]

[Description paragraph]

## Summary
[Key metrics and changes]

## Files
[Diff summary]

---
Generated by: Workflow System
Workflow: TASK_NAME
Mode: [1|2]
```

Example:

```
feature: implement JWT authentication system

Added complete JWT token generation, validation, and middleware
for user authentication. Implemented in 4 steps with full verification
at each stage. All criteria passing.

## Summary
- 4 steps implemented and verified
- JWT token service with sign/verify
- Authentication middleware for protected routes
- Comprehensive test coverage
- Total: 8 files created, 12 files modified

Mode: 1 (Step-by-Step Verification)
Workflow: feature-auth-system
All steps: ✓ PASS
```

## Workflow Type Tags

Finalize can optionally prefix commit based on workflow type:

- **Feature**: `feature: [title]`
- **Bugfix**: `fix: [title]`
- **Refactoring**: `refactor: [title]`
- **Documentation**: `docs: [title]`
- **Testing**: `test: [title]`

Type can be specified during bootstrap or in PLAN.md.

## Step History in Commit

The commit message includes high-level summaries of each step. For detailed history of implementation decisions and iterations, refer to:

- **Step files**: `.workflow/TASK_NAME/steps/step-*.md` — full implementation notes, all iterations
- **Git log**: `git log workflow-TASK_NAME` — individual step commits during implementation

## Error Handling

If finalize encounters errors:

1. **Not all steps complete**
   - Error: "Cannot finalize: Step N not complete"
   - Check step-N.md status

2. **PLAN.md status not 'approved'**
   - Error: "Workflow not yet approved by human"
   - Need human approval before finalize

3. **Not on feature branch**
   - Error: "Not on workflow feature branch: workflow-TASK_NAME"
   - Switch to correct branch

4. **Uncommitted changes**
   - May indicate mid-step changes
   - Error: "Uncommitted changes detected — commit step work first"

5. **Documentation generation issues**
   - Error: "Cannot determine project type — add .workflow-config.json: { projectType: 'api'|'library'|'app' }"
   - Solution: Specify `projectType` in .workflow-config.json to guide documentation generation

## Finalize Output Files

Creates:
- **Commit**: Single commit with all workflow changes
- **Updated PLAN.md**: Status = `complete`, completion date set
- **Git history**: Marked with workflow metadata
- **Documentation files** (NEW):
  - `docs/[PROJECT_NAME]/README.md`
  - `docs/[PROJECT_NAME]/ARCHITECTURE.md`
  - `docs/[PROJECT_NAME]/SETUP.md`
  - `docs/[PROJECT_NAME]/TESTING.md`
  - `docs/[PROJECT_NAME]/routes/` (for APIs) or `docs/[PROJECT_NAME]/features/` (for apps)
  - `docs/[PROJECT_NAME]/API.md` or `FEATURES.md`

Does NOT create:
- Merge commits (manual PR/merge)
- Release tags (separate process)
- Deployment scripts (separate process)

## Post-Finalize Steps

After finalize completes, next steps are:

1. **Review Generated Documentation**:
   ```bash
   # Review key docs
   cat docs/[PROJECT_NAME]/README.md
   cat docs/[PROJECT_NAME]/ARCHITECTURE.md
   cat docs/[PROJECT_NAME]/SETUP.md
   
   # For APIs: check route docs
   ls -la docs/[PROJECT_NAME]/routes/
   ```
   
   Fix any incomplete or inaccurate documentation before committing

2. **Code Review** (if not done):
   ```bash
   git log workflow-TASK_NAME -p
   ```

3. **Run Full Test Suite**:
   ```bash
   npm test
   ```

4. **Verify Documentation Builds** (if using doc generator):
   ```bash
   npm run docs:build  # or equivalent
   ```

5. **Create Pull Request** (if using GitHub):
   ```bash
   gh pr create --head workflow-TASK_NAME --base main \
     --body "$(cat docs/[PROJECT_NAME]/README.md)"
   ```

6. **Merge When Approved**:
   ```bash
   git checkout main
   git merge --ff-only workflow-TASK_NAME
   git push origin main
   ```

7. **Publish Documentation** (optional):
   ```bash
   npm run docs:publish  # if using doc hosting
   # or commit to gh-pages, deploy to docs site, etc.
   ```

8. **Cleanup**:
   ```bash
   git branch -d workflow-TASK_NAME
   git push origin --delete workflow-TASK_NAME
   ```

## Common Documentation Mistakes

### ❌ Incomplete Endpoint Documentation
- Missing error response examples
- No database changes documented
- Unclear parameter descriptions

**Fix**: Include all sections: Description, Parameters (path/query/body), Response (success + errors), Database changes, Examples

### ❌ One Large Document
- `API.md` with 500+ lines
- Hard to search and maintain
- Changes require full-file edits

**Fix**: Use multi-file structure:
- `docs/API.md` as index
- `docs/routes/*.md` for each endpoint

### ❌ Missing Setup/Testing Documentation
- User can't figure out how to run project
- No test examples
- Dependencies not listed

**Fix**: Always create SETUP.md and TESTING.md, list prerequisites clearly

### ❌ Architecture Documentation Without Diagrams
- Hard to understand component relationships
- Data flow unclear

**Fix**: Add ASCII diagrams or descriptions of how components interact

### ❌ Examples Only in One Language
- User can't copy-paste code they're using
- Incomplete documentation

**Fix**: Provide examples in multiple languages (cURL, JavaScript, Python) based on project audience

## Documentation From Step Files

The finalize skill extracts documentation content from step files:

**In each step-N.md, include**:

```markdown
## Implementation Details

### New Endpoints/Routes Created
- `POST /api/users` - Create new user
- `GET /api/users/:id` - Get user by ID

### Database Changes
- Created `users` table with columns: id, name, email, created_at
- Added unique constraint on email

### Key Architectural Decisions
- Used JWT for token-based auth
- Implemented middleware for all protected routes
- Separated route logic into service layer

### Setup/Configuration Changes
- Added `JWT_SECRET` environment variable
- Database migration file: `migrations/001_create_users.sql`
- New npm script: `npm run migrate`

### Testing Approach
- Added 20 unit tests for service layer
- Added 10 integration tests for API endpoints
- Coverage: [X]%
```

The finalize skill reads these sections and populates the generated documentation files.

## Related Skills

- `workflow:execute` — Calls finalize after human final approval
- `workflow:verifier` — Verified all steps that finalize commits
- `workflow:bootstrap` — Created the PLAN.md that finalize updates

**Note:** Finalize includes built-in documentation verification via the "Post-Finalize Steps" section. Human review of generated documentation is the final quality gate before merging.
