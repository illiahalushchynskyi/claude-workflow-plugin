---
name: finalize
description: Create final commit after workflow completion - summarizes all steps and marks workflow as complete
---

# Workflow Finalize Skill

## Overview

The `finalize` skill creates the final commit for a completed workflow. It reads all step files to generate a comprehensive summary of the changes made, creates a commit with detailed context, and updates the PLAN.md to mark the workflow as `complete`.

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
   - Commit preferences, push settings

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

### Step 7: Optional: Push to Remote

If `.workflow-config.json` has `"push": true`:

```bash
git push origin workflow-TASK_NAME
```

Else: just commit locally.

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

## Finalize Output Files

Creates:
- **Commit**: Single commit with all workflow changes
- **Updated PLAN.md**: Status = `complete`, completion date set
- **Git history**: Marked with workflow metadata

Does NOT create:
- Merge commits (manual PR/merge)
- Release tags (separate process)
- Documentation files (separate process)

## Post-Finalize Steps

After finalize completes, next steps are:

1. **Code Review** (if not done):
   ```bash
   git log workflow-TASK_NAME -p
   ```

2. **Run Full Test Suite**:
   ```bash
   npm test
   ```

3. **Create Pull Request** (if using GitHub):
   ```bash
   gh pr create --head workflow-TASK_NAME --base main
   ```

4. **Merge When Approved**:
   ```bash
   git checkout main
   git merge --ff-only workflow-TASK_NAME
   git push origin main
   ```

5. **Cleanup**:
   ```bash
   git branch -d workflow-TASK_NAME
   git push origin --delete workflow-TASK_NAME
   ```

## Related Skills

- `workflow:execute` — Calls finalize after human final approval
- `workflow:verifier` — Verified all steps that finalize commits
- `workflow:bootstrap` — Created the PLAN.md that finalize updates
