# Claude Code Configuration

## Workflow System Permissions

**This project uses an automated workflow system with isolated subagents for implementation and verification.**

### Automatic Permissions for Workflow Skills

When running workflow skills (`workflow:implementer`, `workflow:verifier`, `workflow:execute`), the following tools are **automatically permitted without user confirmation**:

#### Read Tool
- ✅ Reading any project files (source code, configs, tests)
- ✅ Reading workflow state files (`.workflow/*/PLAN.md`, `.workflow/*/steps/*.md`)
- ✅ Reading documentation and markdown files

#### Edit & Write Tools
- ✅ Modifying project source files for implementation
- ✅ Creating new project files
- ✅ Updating workflow step files (`.workflow/*/steps/*.md`)
- ✅ Updating workflow PLAN.md status fields
- ✅ Creating test files

#### Bash Tool
- ✅ Running build commands (`npm run build`, `cargo build`, etc.)
- ✅ Running test commands (`npm test`, `pytest`, etc.)
- ✅ Running project setup commands (`npm install`, migrations, etc.)
- ✅ Running Docker commands (`docker-compose up/down`, `docker exec`, etc.)
- ✅ Running git commands (commits, checking status, etc.)
- ✅ Installing dependencies
- ✅ Running linters and formatters

#### Why These Permissions Are Auto-Approved
- **Implementer subagents** need full access to write code and run local checks
- **Verifier subagents** need to build, test, and verify code actually works
- **Orchestrator** needs to track workflow state and dispatch subagents
- These permissions are scoped to the project directory only
- All work is tracked in git and visible in workflow state files

### What Requires User Approval
- Pushing to remote repositories (git push)
- Creating pull requests
- Destructive operations outside workflow context (force push, deleting branches, etc.)
- Actions outside this project directory

### Workflow Execution Guarantee
When a workflow is in execution:
- Subagents will **NOT** ask for permission on any of the auto-approved tools above
- Subagents will **NOT** pause for confirmation
- Subagents will complete implementation, testing, and verification independently
- User is only asked for approval when:
  - A step completes successfully (Mode 1)
  - Final workflow approval needed
  - Critical blockers occur that can't be resolved

## Subagent Independence

Workflow subagents (`implementer`, `verifier`) are designed to be **fully independent**:

- They have isolated contexts (don't see orchestrator's conversation)
- They don't rely on orchestrator's state or approvals during execution
- They make independent decisions about code quality and test passing
- They report back with brief summaries, not full output

This keeps the main conversation clean while subagents do the heavy lifting.

## No Manual Workflow File Editing

Users should NOT manually edit:
- `.workflow/*/PLAN.md` step table rows (orchestrator manages this)
- `.workflow/*/steps/step-*.md` frontmatter status fields (subagents manage this)

These files are managed by the workflow system. Manual edits can break the automation.

It's safe to:
- Read these files to understand progress
- Create workflow files via `workflow:bootstrap`
- Modify step goals/descriptions if needed (before implementation starts)
