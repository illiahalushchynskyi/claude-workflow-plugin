---
task_name: bugfix-login-timeout
title: Fix User Login Timeout Issue
mode: 2
status: planning
created: 2026-04-08
started: null
completed: null
workflow_type: bugfix
author: Development Team
description: |
  Users are being logged out after 2 hours instead of the configured 7 days.
  Investigation shows the session timeout is being set to 7200 seconds (2 hours)
  instead of 7 days (604800 seconds). This is a configuration error in the
  session middleware. Fix will require updating the timeout constant and
  adding a regression test.
---

# Fix User Login Timeout Issue

## Goal

Identify and fix the root cause of users being logged out after 2 hours instead of 7 days, and add regression test to prevent recurrence.

## Architecture & Approach

The issue is a misconfiguration in the session timeout calculation. Sessions are currently set to 7200 seconds (2 hours) when they should be 604800 seconds (7 days). The fix involves updating the constant in the middleware and verifying the change doesn't break existing sessions. A regression test will ensure this doesn't happen again.

## Mode Selected

- **Mode 2 (End-to-End)**: Selected
  This is a low-risk, well-understood bugfix with a clear root cause.
  All steps can execute with verification gates between them, with a single
  human approval at the end. No mid-workflow pauses needed.

## Steps Overview

| Step | Name | Status | Issues |
|------|------|--------|--------|
| 1 | Locate timeout configuration | pending | — |
| 2 | Update timeout constant | pending | — |
| 3 | Add regression test | pending | — |
| 4 | Verify with manual testing | pending | — |

## Overall Progress

- [ ] Step 1 Complete
- [ ] Step 2 Complete
- [ ] Step 3 Complete
- [ ] Step 4 Complete
- [ ] Human Approval (single gate at end)
- [ ] Commit & Merge

## Step Details Location

All step detail files are in: `.workflow/bugfix-login-timeout/steps/`

## Notes & Escalations

**Reported by:** Support team on 2026-04-07
**Impact:** Users frustrated with frequent logouts
**Urgency:** High (customer-facing issue)

---

## Why Mode 2 for This Bugfix?

This is an ideal Mode 2 candidate because:

1. **Clear root cause** — Not a mystery, known misconfiguration
2. **Small scope** — Only 4 small steps, all interdependent
3. **Low risk** — Changing a timeout constant is low-risk
4. **High confidence** — Similar fixes done before, well-understood pattern
5. **Time pressure** — Mode 2 is faster (no mid-workflow pauses)
6. **Test coverage** — Existing test suite will validate the fix

## Execution Timeline

**Mode 2 (End-to-End):** Expected duration: 45-60 minutes

```
Step 1: Find timeout config (10 min impl + 5 min verify)
  → [VERIFY: ✓ Found in middleware]
  → [NO PAUSE — auto-proceed to Step 2]

Step 2: Update timeout (5 min impl + 5 min verify)
  → [VERIFY: ✓ Changed to 604800 seconds]
  → [NO PAUSE — auto-proceed to Step 3]

Step 3: Add regression test (15 min impl + 10 min verify)
  → [VERIFY: ✓ Test passes, covers 7-day timeout]
  → [NO PAUSE — auto-proceed to Step 4]

Step 4: Manual verification (10 min impl + 5 min verify)
  → [VERIFY: ✓ Verified session timing]

[All steps verified]
  → [SINGLE HUMAN APPROVAL: 5 min]
  → Finalize & Commit: 5 min

TOTAL: ~55 minutes
Human involvement: 1 approval gate (minimal overhead)
```

---

## Configuration

The `.workflow-config.json` for this workflow:

```json
{
  "task_name": "bugfix-login-timeout",
  "mode": 2,
  "auto_advance": true,
  "verification_strategy": "standard",
  "notification_method": "none",
  "branch_prefix": "workflow-",
  "commit_on_complete": true,
  "push_on_complete": true,
  "step_approval_required": false,
  "require_tests": true,
  "require_linting": true
}
```

Note: `"push_on_complete": true` — for urgent bugfix, auto-push to remote.

---

## Success Criteria

Workflow complete when:

- [x] Root cause identified (timeout configuration)
- [x] Fix applied (604800 seconds = 7 days)
- [x] Regression test added and passing
- [x] Manual testing confirms 7-day timeout
- [x] All existing tests still pass
- [x] Human approval obtained
- [x] Changes committed and pushed

---

## Related Documentation

- See `steps/` directory for detailed step files
- See [MODE_GUIDE.md](../../docs/MODE_GUIDE.md) for why Mode 2 for bugfixes
- See [QUICKSTART.md](../../docs/QUICKSTART.md) for first-time workflow guidance
