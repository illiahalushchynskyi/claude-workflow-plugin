---
task_name: refactor-db-layer
title: Refactor Database Layer to Repository Pattern
mode: 1
status: planning
created: 2026-04-08
started: null
completed: null
workflow_type: refactor
author: Development Team
description: |
  Extract database queries into a repository layer to improve testability,
  reusability, and maintainability. This refactor introduces the Repository
  pattern while maintaining all existing functionality. Uses Mode 1 with
  comprehensive testing at each step to ensure no regressions.
---

# Refactor Database Layer to Repository Pattern

## Goal

Extract all database queries into a dedicated repository layer, improving code organization, testability, and enabling better separation of concerns.

## Architecture & Approach

Introduce a Repository pattern that abstracts all database queries. Each entity (User, Post, Comment, etc.) gets a corresponding Repository class (UserRepository, PostRepository, etc.). Services will use repositories instead of making direct database calls. This maintains existing behavior while improving code organization. All existing tests will pass, ensuring no regressions.

## Mode Selected

- **Mode 1 (Step-by-Step)**: Selected
  While this is a straightforward refactor, we're using Mode 1 to verify
  each component before moving to the next. This ensures:
  - No unexpected behavioral changes
  - Tests remain passing at each step
  - Architecture decisions can be reviewed
  - Regressions caught early (not after all steps)

## Steps Overview

| Step | Name | Status | Issues |
|------|------|--------|--------|
| 1 | Create repository base class | pending | — |
| 2 | Extract User repository | pending | — |
| 3 | Extract Post repository | pending | — |
| 4 | Extract Comment repository | pending | — |
| 5 | Update services to use repositories | pending | — |
| 6 | Remove direct DB calls from services | pending | — |
| 7 | Add integration tests | pending | — |

## Overall Progress

- [ ] Step 1 Complete
- [ ] Step 2 Complete
- [ ] Step 3 Complete
- [ ] Step 4 Complete
- [ ] Step 5 Complete
- [ ] Step 6 Complete
- [ ] Step 7 Complete
- [ ] Human Approval
- [ ] Commit & Merge

## Step Details Location

All step detail files are in: `.workflow/refactor-db-layer/steps/`

## Notes & Escalations

**Planning notes:**
- Existing test coverage is ~85% — adequate for this refactor
- Services layer currently couples business logic with DB calls
- No breaking changes to API contracts
- All existing endpoints must continue working

**Timeline consideration:** This is a larger refactor (7 steps). Expected total duration: 4-5 hours with Mode 1 oversight.

---

## Why Mode 1 for This Refactor?

Although refactoring might seem suitable for Mode 2, we're using Mode 1 here because:

1. **Large scope** — 7 steps is substantial; benefit from step-by-step oversight
2. **Architectural significance** — Pattern change warrants review at each stage
3. **Testing critical** — Want to verify tests still pass after each step
4. **Legacy code risk** — Changes to core database layer deserve careful review
5. **Learning opportunity** — Good chance to review repository pattern implementation

If this were a smaller refactor or more well-known pattern, Mode 2 would be appropriate.

---

## Execution Timeline

**Mode 1 (Step-by-Step):** Expected duration: 4-5 hours

```
Step 1: Create base class (20 min impl + 10 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 2: Extract User repo (25 min impl + 15 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 3: Extract Post repo (25 min impl + 15 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 4: Extract Comment repo (25 min impl + 15 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 5: Update services (30 min impl + 15 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 6: Remove direct DB (20 min impl + 15 min verify)
  → [REVIEW & APPROVE: 5 min]

Step 7: Add integration tests (20 min impl + 20 min verify)
  → [REVIEW & APPROVE: 5 min]

Final Review: 15 min
Finalize & Commit: 5 min

TOTAL: ~270 minutes ≈ 4.5 hours
Human involvement: ~7 approval gates + comprehensive reviews
```

---

## Configuration

```json
{
  "task_name": "refactor-db-layer",
  "mode": 1,
  "auto_advance": false,
  "verification_strategy": "comprehensive",
  "notification_method": "none",
  "branch_prefix": "workflow-",
  "commit_on_complete": true,
  "push_on_complete": false,
  "step_approval_required": true,
  "require_tests": true,
  "require_linting": true,
  "require_documentation": true
}
```

---

## Success Criteria

Refactor complete when:

- [x] All 7 steps have status: `complete`
- [x] All existing tests pass (no regressions)
- [x] Repository pattern implemented correctly
- [x] All database queries moved to repositories
- [x] Services use only repositories (no direct DB calls)
- [x] New integration tests added and passing
- [x] Code follows project conventions
- [x] Human approval obtained
- [x] Final commit created with summary

---

## Post-Refactor Validation

After workflow completion:

1. Run full test suite:
   ```bash
   npm test
   npm run test:integration
   ```

2. Check code coverage:
   ```bash
   npm run test:coverage
   # Should remain ~85% or better
   ```

3. Review database query patterns:
   ```bash
   grep -r "sequelize.query\|Model.findAll" src/services/
   # Should return nothing (all moved to repositories)
   ```

4. Performance testing:
   ```bash
   npm run performance:test
   # Verify no regression in response times
   ```

---

## Related Documentation

- See `steps/` directory for detailed step files
- See [MODE_GUIDE.md](../../docs/MODE_GUIDE.md) for refactoring best practices
- See [QUICKSTART.md](../../docs/QUICKSTART.md) for workflow guidance
- Design pattern reference: [Repository Pattern Guide](https://martinfowler.com/eaaCatalog/repository.html)
