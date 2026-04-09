# Workflow System Quick Start Guide

Get up and running with the Workflow System in 5 minutes.

## Prerequisites

- Claude Code installed and running
- Workflow plugin installed (see [INSTALLATION.md](INSTALLATION.md))
- A project with git initialized
- Basic familiarity with your project structure

## Step 1: Create Your First Workflow (2 minutes)

### 1.1 Open Claude Code

Start Claude Code or bring it to focus:

```bash
claude-code
```

### 1.2 Initialize Bootstrap

In Claude Code, invoke the bootstrap skill:

```
/workflow:bootstrap
```

The skill will prompt you for:

1. **Task Name** — e.g., `feature-user-profile`
   - Use lowercase, hyphens for spaces
   - This becomes your workflow directory name

2. **Task Title** — e.g., "Add User Profile Feature"
   - Human-readable title

3. **Mode Selection** — Type `1` or `2`:
   - **Mode 1:** Step-by-step with human approval after each step
   - **Mode 2:** All steps run automatically, approve at end
   - For your first workflow, **choose Mode 1** (safer, more learning)

4. **Task Description** — Brief explanation of what you're building

5. **Number of Steps** — How many implementation steps? (e.g., 4)

6. **Step Definitions** — For each step, provide:
   - **Step Name** (e.g., "Create database schema")
   - **Goal** (e.g., "Add user_profile table with columns: bio, avatar_url, updated_at")
   - **Verification Criteria** (e.g., "Table created, migration runs without errors, schema matches design spec")

### 1.3 Review Generated Files

Bootstrap creates:

```
.workflow/
└── feature-user-profile/
    ├── PLAN.md                      # Main plan
    ├── .workflow-config.json        # Configuration
    └── steps/
        ├── step-1-create-schema.md
        ├── step-2-add-api-endpoints.md
        ├── step-3-add-validation.md
        └── step-4-write-tests.md
```

Open `.workflow/feature-user-profile/PLAN.md` to review the plan bootstrap created.

## Step 2: Start Execution (1 minute)

In Claude Code:

```
/workflow:execute
```

The execute skill:
1. Reads your PLAN.md
2. Identifies the first pending step
3. Signals the implementer to start work

## Step 3: Implement First Step (5-10 minutes)

The implementer agent will:

1. Read `step-1-*.md` to understand what to build
2. Make code changes
3. Run local tests to verify
4. Update `step-1-*.md` with implementation notes
5. Mark step status as `verification`

**What you do:** Watch Claude Code implement the step. It will:
- Read files to understand the codebase
- Create or modify files
- Test locally
- Document what it did

When the implementer finishes, check the step file to see the notes.

## Step 4: Verify First Step (2-5 minutes)

The verifier agent will:

1. Read the verification criteria from `step-1-*.md`
2. Review the code changes
3. Run tests and linting
4. Check against all criteria
5. Mark as `complete` or `needs-fix`

**What you do:** 
- If PASS (complete): Step 1 is approved! ✓
- If FAIL (needs-fix): Implementer will fix issues and re-verify

## Step 5: Approve and Continue (1 minute)

In Mode 1, after verification:

1. **Review Step 1** — Open `.workflow/feature-user-profile/steps/step-1-*.md`
   - Read implementer notes
   - Read verifier results
   - Understand what was built

2. **Approve** — Signal to continue:
   ```
   /workflow:execute
   ```
   
   Execute will advance to step 2

3. **Repeat** — For each remaining step:
   - Execute signals implementer
   - Implementer works
   - Verifier tests
   - You approve
   - Next step

## Step 6: Complete Workflow (1 minute)

After all steps are verified and approved:

1. Review all step files:
   ```
   ls -la .workflow/feature-user-profile/steps/
   ```

2. Check overall PLAN.md status

3. Finalize the workflow:
   ```
   /workflow:finalize
   ```

   Finalize will:
   - Create comprehensive commit message
   - Summarize all changes across steps
   - Mark workflow as complete
   - Update PLAN.md with completion date

## Example Workflow: User Profile Feature

Here's what a complete workflow might look like:

### PLAN.md

```
Task: Add User Profile Feature
Mode: 1 (Step-by-Step)
Steps: 4

1. Create database schema
2. Create API endpoints (GET/POST/PUT)
3. Add input validation
4. Write comprehensive tests
```

### Execution Flow

```
Step 1: Create database schema
  Implementer: Add migration file, create table
  Verifier: Migration runs, schema correct ✓
  You: Review, approve

Step 2: Create API endpoints
  Implementer: Add route handlers, basic implementation
  Verifier: Endpoints respond, return correct status codes ✓
  You: Review, approve

Step 3: Add validation
  Implementer: Add input validation, error handling
  Verifier: Invalid input rejected, error messages correct ✓
  You: Review, approve

Step 4: Write tests
  Implementer: Add test suite for all endpoints
  Verifier: All tests passing, coverage >80% ✓
  You: Review, approve

All done! 
  Finalize: Create commit with summary
  Status: Complete ✓
```

## Common Commands

```bash
# View workflow status
cat .workflow/feature-user-profile/PLAN.md

# View specific step details
cat .workflow/feature-user-profile/steps/step-1-*.md

# Check git history of workflow
git log --oneline .workflow/feature-user-profile/

# See what code changed in step 1
git diff HEAD~1 -- backend/src/  # See changes from last commit
```

## Tips & Tricks

### Tip 1: Clear Step Definitions

The clearer your verification criteria, the better the verifier can test. 

**Good:**
- "API returns 200 on valid input"
- "All unit tests passing"
- "No TypeScript errors"

**Vague:**
- "It works"
- "Tests pass"
- "No errors"

### Tip 2: Reasonable Step Sizes

Don't make steps too big or too small.

**Too small:** "Add import statement"
**Too big:** "Build entire auth system"
**Just right:** "Add JWT token generation endpoint"

A step should take 15-60 minutes of implementation + 5-15 minutes of verification.

### Tip 3: Know Your Mode

- **Mode 1:** Use for high-risk changes, security features, architectural changes
- **Mode 2:** Use for low-risk changes, bug fixes, refactoring

### Tip 4: Review Before Approval

Don't blindly approve. Open the step file and review:
- Implementer notes — what decisions were made?
- Code changes — does the approach make sense?
- Verifier results — did all criteria pass?

### Tip 5: Keep Workflows Focused

One workflow = one feature/bugfix/refactor.

**Good:**
- Workflow: "Add JWT auth system"
- Workflow: "Fix login timeout bug"

**Too many things:**
- Workflow: "Refactor entire backend and add new UI"

## Next Steps

1. **Read MODE_GUIDE.md** — Understand Mode 1 vs Mode 2 in depth
2. **Check examples/** — See real example workflows
3. **See workflow-format.md** — Reference for file formats
4. **Troubleshooting** — See TROUBLESHOOTING.md if issues arise

## Getting Help

1. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues
2. Review the example workflows in `examples/`
3. Read [MODE_GUIDE.md](MODE_GUIDE.md) for detailed mode explanation

## What Happens Behind the Scenes

### File Structure Created

```
.workflow/TASK_NAME/
├── PLAN.md                      # 1 file: overall plan
├── .workflow-config.json        # 1 file: config
└── steps/
    ├── step-1-*.md              # N files: one per step
    ├── step-2-*.md
    └── step-N-*.md
```

### State Tracking

All state is in markdown files with structured frontmatter:

- **PLAN.md** tracks: task name, mode, status, created/started/completed dates
- **step-N.md** tracks: goal, verification criteria, implementation notes, verification results, issue history, iteration count

Every change is timestamped and recorded.

### Git Integration

- Bootstrap creates feature branch: `workflow-TASK_NAME`
- Each step commits locally during implementation
- Finalize creates final summary commit
- Everything is git-tracked for history and recovery

## Success Criteria

You've successfully used the Workflow System when:

- [ ] Created first workflow with `/workflow:bootstrap`
- [ ] Executed workflow with `/workflow:execute`
- [ ] Implementer completed step 1
- [ ] Verifier tested and approved step 1
- [ ] You approved step 1
- [ ] All steps completed
- [ ] Workflow finalized with `/workflow:finalize`
- [ ] Final commit created

Congratulations! You now understand the complete workflow cycle.

---

**Ready to learn more?**
- [MODE_GUIDE.md](MODE_GUIDE.md) — Deep dive into Mode 1 vs Mode 2
- [INSTALLATION.md](INSTALLATION.md) — How to set up on a new machine
- [workflow-format.md](workflow-format.md) — Complete format reference
