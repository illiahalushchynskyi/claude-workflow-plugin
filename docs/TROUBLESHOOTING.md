# Troubleshooting Guide

Solutions for common issues when using the Workflow System.

## Installation & Setup Issues

### "workflow skills not appearing"

**Problem:** You installed the plugin but `/workflow:bootstrap` isn't recognized.

**Cause:** Plugin not properly loaded by Claude Code.

**Solution:**

1. Verify plugin is in correct location:
   ```bash
   ls -la ~/.claude/plugins/workflow/
   # Should show: plugin.json, skills/, docs/, etc.
   ```

2. Verify plugin.json exists:
   ```bash
   cat ~/.claude/plugins/workflow/plugin.json
   # Should be valid JSON with "workflow" as name
   ```

3. Verify skills directory:
   ```bash
   ls ~/.claude/plugins/workflow/skills/
   # Should show: bootstrap.md, implementer.md, verifier.md, execute.md, finalize.md
   ```

4. Restart Claude Code entirely (not just clear):
   ```bash
   pkill -f claude-code
   sleep 2
   claude-code
   ```

5. Check Claude Code settings:
   ```bash
   cat ~/.claude/settings.json | jq .plugins
   # Should include ~/.claude/plugins path
   ```

**Still not working?** Try Method 3 (settings.json path) from [INSTALLATION.md](INSTALLATION.md)

---

### "Cannot create .workflow directory (permission denied)"

**Problem:** Running `/workflow:bootstrap` fails with permission error.

**Cause:** No write permission to project directory.

**Solution:**

1. Check permissions:
   ```bash
   ls -ld /path/to/project
   # Should show your user (not "root")
   ```

2. Fix permissions (if you own the directory):
   ```bash
   chmod 755 /path/to/project
   # Or recursively:
   chmod -R u+rwx /path/to/project
   ```

3. If project is owned by someone else, request access or create symbolic link to owned directory

---

### "Git is not initialized in project"

**Problem:** Workflow skips git setup or fails on git commands.

**Cause:** Project directory is not a git repository.

**Solution:**

```bash
cd /path/to/project
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Create initial commit
git add .
git commit -m "initial commit"
```

Then retry `/workflow:bootstrap`

---

## Workflow Creation Issues

### "task_name already exists"

**Problem:** Bootstrap says `.workflow/feature-auth-system/` already exists.

**Cause:** You're trying to create a workflow with the same name.

**Solution:**

1. **Option A:** Use different task name:
   ```
   /workflow:bootstrap
   Task Name: feature-auth-system-v2
   ```

2. **Option B:** Delete existing workflow (if you don't need it):
   ```bash
   rm -rf .workflow/feature-auth-system
   git add .workflow/
   git commit -m "workflow: remove old workflow"
   ```

3. **Option C:** Continue existing workflow:
   ```bash
   /workflow:execute
   # Will resume from last state
   ```

---

### "Invalid task_name format"

**Problem:** Bootstrap rejects your task name.

**Cause:** Task name contains invalid characters.

**Valid format:** Alphanumerics, hyphens, underscores only. Max 128 characters.

**Solution:** Use only these characters:
- ✓ `feature-auth-system`
- ✓ `bugfix_timeout_issue`
- ✓ `refactor123`
- ✗ `feature auth system` (spaces)
- ✗ `feature-auth-system!` (punctuation)
- ✗ `FEATURE-AUTH` (uppercase OK but prefer lowercase)

---

### "No steps defined"

**Problem:** Bootstrap fails saying you need at least 1 step.

**Cause:** You didn't define any workflow steps.

**Solution:** When bootstrapping, define at least 1 step:

```
Number of Steps: 1

Step 1:
  Name: Implement login endpoint
  Goal: Create POST /api/login that returns JWT
  Criteria: Returns 200 on valid credentials
```

---

## Workflow Execution Issues

### "Step file is corrupted/unreadable"

**Problem:** Executing workflow gives error about step file format.

**Cause:** Step file YAML frontmatter is invalid.

**Solution:**

1. Open the step file:
   ```bash
   cat .workflow/TASK_NAME/steps/step-1-*.md
   ```

2. Check frontmatter (between `---` markers):
   - Should be valid YAML
   - Check indentation (use spaces, not tabs)
   - Verify all required fields present

3. Compare with example:
   ```bash
   cat ~/.claude/plugins/workflow/examples/feature-auth-system/steps/step-1-*.md
   ```

4. Fix the file (edit in your editor, ensure valid YAML)

5. Retry execution:
   ```bash
   /workflow:execute
   ```

**Example:** YAML indent error
```yaml
# WRONG (tabs):
---
step_number:	1
---

# RIGHT (spaces):
---
step_number: 1
---
```

---

### "Implementer is not working on my step"

**Problem:** You ran `/workflow:execute` but implementer doesn't start.

**Cause:** Step is not in "pending" or "implementation" status.

**Solution:**

1. Check step status:
   ```bash
   head -20 .workflow/TASK_NAME/steps/step-1-*.md | grep "status:"
   # Should show: status: pending or status: implementation
   ```

2. If status is something else:
   - `complete` → Already done, move to next step
   - `verification` → Verifier is working/waiting
   - `needs-fix` → Previous iteration needs fixes

3. If correct status, manually run implementer:
   ```
   /workflow:implementer
   # And select your step
   ```

---

### "Verifier won't test my step"

**Problem:** Step marked `verification` but verifier doesn't start.

**Cause:** Verification criteria might be unclear or missing.

**Solution:**

1. Check verification criteria:
   ```bash
   grep -A 10 "## Verification Criteria" .workflow/TASK_NAME/steps/step-1-*.md
   ```

2. Criteria should be:
   - Clear and testable
   - Specific (not "it works", but "endpoint returns 200")
   - Measurable

3. If criteria are vague, update them:
   ```markdown
   ## Verification Criteria
   
   - [ ] POST /api/login responds with 200 on valid credentials
   - [ ] Response includes access_token field
   - [ ] Invalid credentials return 401
   - [ ] All unit tests pass
   ```

4. Retry verifier:
   ```
   /workflow:verifier
   ```

---

### "Step seems stuck (no progress for 30 min)"

**Problem:** Execute shows step is being worked on but nothing happens.

**Cause:** Agent might be blocked or waiting.

**Solution:**

1. Check step file for notes:
   ```bash
   grep -A 5 "Implementer\|Verifier" .workflow/TASK_NAME/steps/step-N-*.md | tail -20
   # Shows what agent was last doing
   ```

2. If implementer is stuck:
   - Look for "blocked" field in step frontmatter
   - Check implementation notes for error messages
   - Manually retry: `/workflow:implementer`

3. If verifier is stuck:
   - Check if build might be failing
   - Look for test timeouts
   - Manually retry: `/workflow:verifier`

4. If nothing helps:
   - Manually update step status
   - Mark as `needs-fix` if problems found
   - Retry from that state

---

### "Mode 1: Human approval never comes"

**Problem:** Step verified but you're not seeing approval prompt.

**Cause:** Execute might be waiting for input.

**Solution:**

1. Ensure you're in Claude Code interactive mode:
   ```bash
   claude-code
   # Not: claude-code < some_file.txt
   ```

2. Check step status:
   ```bash
   grep "status:" .workflow/TASK_NAME/steps/step-N-*.md
   # Should show: complete
   ```

3. Manually signal approval in Claude Code:
   ```
   Ready to approve? Type 'approve' or 'reject'
   approve
   ```

4. If stuck, check for issues in step file and address them

---

### "Mode 2: Previous step not complete but expected to advance"

**Problem:** Mode 2 won't let implementer start step N because step N-1 isn't verified.

**Expected behavior.** This is a safety feature!

**Solution:**

1. Verify previous step:
   ```bash
   grep "status:" .workflow/TASK_NAME/steps/step-N-1-*.md
   # If not "complete", run verifier on it
   ```

2. Check if verifier found issues:
   ```bash
   tail -30 .workflow/TASK_NAME/steps/step-N-1-*.md | grep -A 10 "Verification"
   ```

3. If `needs-fix`:
   - Implementer must fix that step first
   - Can't skip ahead
   - This is by design (sequential verification)

4. Once N-1 is `complete`, N-1 step 2+ implementation will auto-unblock

---

## File & State Issues

### "PLAN.md is out of sync with steps"

**Problem:** PLAN.md shows different status than individual step files.

**Cause:** Manual edits or crashed update.

**Solution:**

1. Check actual step statuses:
   ```bash
   for f in .workflow/TASK_NAME/steps/step-*.md; do
     echo -n "$f: "
     grep "^status:" "$f"
   done
   ```

2. Update PLAN.md step table to match:
   ```markdown
   | Step | Name | Status | Issues |
   |------|------|--------|--------|
   | 1 | [name] | [actual status] | — |
   ```

3. Update progress checkboxes:
   ```markdown
   - [x] Step 1 Complete  (if status: complete)
   - [ ] Step 1 Complete  (if status: anything else)
   ```

4. Retry execute:
   ```
   /workflow:execute
   ```

---

### "Accidentally deleted step file"

**Problem:** You deleted `step-2-*.md` by accident.

**Cause:** Filesystem mistake.

**Solution:**

1. **Option A:** Recover from git:
   ```bash
   git checkout .workflow/TASK_NAME/steps/step-2-*.md
   ```

2. **Option B:** Recover from backup:
   ```bash
   # If you have a backup
   cp /path/to/backup/.workflow/TASK_NAME/steps/step-2-*.md \
      .workflow/TASK_NAME/steps/
   ```

3. **Option C:** Recreate it:
   - Check PLAN.md for step definition
   - Manually create step file with proper frontmatter
   - Copy structure from similar step file

---

### "Workflow files are corrupted (git won't commit)"

**Problem:** Git says files are corrupted or won't commit workflow changes.

**Cause:** File encoding, line endings, or git conflict.

**Solution:**

1. Check file encoding:
   ```bash
   file .workflow/TASK_NAME/PLAN.md
   # Should show: UTF-8 or similar
   ```

2. Check line endings:
   ```bash
   file -i .workflow/TASK_NAME/PLAN.md
   # Should show: charset=utf-8
   ```

3. Fix line endings if needed:
   ```bash
   dos2unix .workflow/TASK_NAME/**/*.md
   # OR
   sed -i 's/\r$//' .workflow/TASK_NAME/**/*.md
   ```

4. Try commit again:
   ```bash
   git add .workflow/
   git commit -m "workflow: update state"
   ```

---

## Agent Behavior Issues

### "Implementer is making wrong changes"

**Problem:** Implementer modified wrong files or didn't understand requirements.

**Cause:** Unclear step definition or verification criteria.

**Solution:**

1. Stop execution: Don't approve the step
2. Review step definition:
   ```bash
   head -40 .workflow/TASK_NAME/steps/step-N-*.md
   ```
3. Make definition clearer:
   - More specific files to modify
   - Clearer goal statement
   - Better verification criteria
4. Mark step back to `pending`:
   ```bash
   # Edit step frontmatter
   status: pending
   iteration: 1
   ```
5. Retry implementer with better instructions:
   ```
   /workflow:implementer
   ```

---

### "Verifier is too strict/lenient"

**Problem:** Verification criteria are not catching real issues (lenient) or are unreasonable (strict).

**Cause:** Criteria not well-calibrated for the project.

**Solution:**

1. **Too lenient** (missing real issues):
   - Update verification criteria to be more specific
   - Add more test cases
   - Require higher code quality (linting, documentation)
   - Example: Instead of "API works", require "API returns correct HTTP status codes for all cases"

2. **Too strict** (unreasonable expectations):
   - Simplify criteria
   - Example: Don't require 100% test coverage, require 80%
   - Focus on behavior over implementation details

3. Update config:
   ```bash
   # Edit .workflow/TASK_NAME/.workflow-config.json
   "verification_strategy": "standard"  # instead of "strict"
   "require_tests": true
   "require_linting": true
   ```

---

## Mode-Specific Issues

### Mode 1: Too many approval pauses

**Problem:** Mode 1 is slowing you down with too many pauses.

**Solution:**

1. **Option A:** Switch to Mode 2:
   - Update PLAN.md: `mode: 2`
   - Update .workflow-config.json: `"mode": 2`
   - Retry execute

2. **Option B:** Keep Mode 1 but batch approvals:
   - Let steps 1-2 complete
   - Review all at once
   - Approve together

3. **Option C:** Adjust step granularity:
   - Combine small steps
   - Reduce total number of approval gates

---

### Mode 2: Want to insert a manual approval

**Problem:** Mode 2 is too hands-off; you want to review step 2 before step 3 starts.

**Solution:**

1. After step 2 verifies:
   ```bash
   # Don't run execute again yet
   # Manually review:
   cat .workflow/TASK_NAME/steps/step-2-*.md
   git diff HEAD -- [modified files]
   ```

2. When ready to proceed:
   ```
   /workflow:execute
   ```

3. This gives you selective approval while keeping Mode 2's speed

---

### "Mode changed mid-workflow, now confused"

**Problem:** Started in Mode 1, switched to Mode 2 or vice versa.

**Cause:** Shouldn't change modes mid-workflow.

**Solution:**

1. **Option A:** Continue with new mode:
   - Just proceed normally
   - New mode applies to remaining steps
   - PLAN.md reflects new mode

2. **Option B:** Switch back:
   - Update PLAN.md: `mode: [original]`
   - Update config
   - Continue with original mode

3. **Best practice:** Don't switch modes mid-workflow. Choose at bootstrap and stick with it.

---

## Recovery Procedures

### "I messed up step 3 badly, want to start over"

**Problem:** Workflow has steps 1-2 done but step 3 is really broken.

**Solution:**

1. **Option A:** Fix step 3 iteratively (recommended):
   ```bash
   # Mark step 3 back to pending
   # In .workflow/TASK_NAME/steps/step-3-*.md
   status: pending
   iteration: 1
   
   # Retry
   /workflow:execute
   ```

2. **Option B:** Start step 3 fresh:
   ```bash
   # Backup current step 3
   cp .workflow/TASK_NAME/steps/step-3-*.md \
      .workflow/TASK_NAME/steps/step-3-*.md.backup
   
   # Check out original from git (if you want to reset to bootstrap)
   git checkout .workflow/TASK_NAME/steps/step-3-*.md
   
   # Retry
   /workflow:execute
   ```

3. **Option C:** Remove and recreate step 3:
   ```bash
   rm .workflow/TASK_NAME/steps/step-3-*.md
   
   # Then use bootstrap to create step structure again
   # But you'll lose iteration history
   ```

---

### "All steps done but don't want to commit yet"

**Problem:** Workflow is complete but you want to keep working before finalize.

**Cause:** Ready to commit but want more testing first.

**Solution:**

1. Don't run finalize yet:
   ```bash
   # Don't run this:
   /workflow:finalize
   ```

2. Keep working:
   ```bash
   # Make additional changes outside workflow
   git add src/
   git commit -m "additional fixes for step 3"
   ```

3. When ready to finalize:
   ```bash
   /workflow:finalize
   # Creates comprehensive commit with all workflow changes
   ```

---

### "I want to rollback entire workflow"

**Problem:** Workflow completed but you want to undo all changes.

**Cause:** Found critical issue after approval.

**Solution:**

1. **Option A:** Git revert (if already committed):
   ```bash
   # Find workflow commits
   git log --oneline .workflow/
   
   # Revert to before workflow started
   git revert HEAD~5..HEAD  # adjust range based on your commits
   
   # Or reset (destructive):
   git reset --hard origin/main
   ```

2. **Option B:** Manual cleanup (if not committed yet):
   ```bash
   # Delete workflow directory
   rm -rf .workflow/TASK_NAME
   
   # Revert code changes
   git checkout .  # careful! reverts all changes
   
   # Or selectively:
   git checkout src/  # reverts just src/
   ```

3. **Option C:** Start new workflow to fix issues:
   ```bash
   # Don't delete the workflow
   # Create new workflow for the fixes:
   /workflow:bootstrap
   Task Name: bugfix-workflow-issues
   ```

---

## Performance Issues

### "Workflow is slow"

**Problem:** Steps taking much longer than expected.

**Cause:** Could be many things.

**Solution:**

1. **Check implementer speed:**
   - Is Claude Code responsive?
   - Are you throttled (API limits)?
   - Are code changes large?

2. **Check verifier speed:**
   - Are tests running slowly?
   - Do you need more test infrastructure?
   - Are criteria too complex to verify?

3. **Optimize:**
   - Use smaller steps (easier to verify)
   - Parallelize unrelated steps if possible
   - Consider Mode 2 for faster execution

4. **Monitor:**
   ```bash
   # Check time per step
   grep "created\|completed" .workflow/TASK_NAME/steps/step-*.md
   ```

---

## Getting Help

### Before asking for help:

1. **Check this guide** — Your issue likely covered here
2. **Check [workflow-format.md](workflow-format.md)** — File format reference
3. **Check [MODE_GUIDE.md](MODE_GUIDE.md)** — Mode explanation
4. **Check example workflows** — See examples/ directory
5. **Check logs** — What's the exact error message?

### If still stuck:

1. Open an issue with:
   - Exact error message
   - Steps to reproduce
   - What you expected to happen
   - What actually happened
   - Your OS and Claude Code version

2. Provide workflow files (sanitized):
   ```bash
   cat .workflow/TASK_NAME/PLAN.md
   cat .workflow/TASK_NAME/steps/step-N-*.md | head -50
   cat .workflow/TASK_NAME/.workflow-config.json
   ```

---

**Version:** 1.0.0
**Last Updated:** 2026-04-08
