# Workflow Modes: Complete Guide

This guide explains the two operational modes of the Workflow System and how to choose which mode to use.

## Overview

The Workflow System supports two distinct execution modes:

- **Mode 1: Step-by-Step Verification** — Human approval after each step
- **Mode 2: End-to-End Execution** — Single human approval at the end

Both modes use the same agents and file structure. The difference is when humans make approval decisions and how tightly agents are gated.

## Mode 1: Step-by-Step Verification

### How It Works

```
For each step (in order):
  1. Implementer works on step N (status: implementation)
  2. Implementer finishes and commits (status: verification)
  3. Verifier tests step N (still status: verification)
  4. Verifier passes/fails:
     - Tests pass: status → awaiting-approval
     - Tests fail: status → needs-fix
  
  5. **[IF AWAITING-APPROVAL]** → Human approval gate
     - Workflow pauses (workflow_status: paused)
     - Human reviews step N
     - [APPROVE] → status = complete, continue to step N+1
     - [REQUEST CHANGES] → status = needs-fix, iteration++
  
  6. **[IF NEEDS-FIX]** → Implementer fixes (loop back)
     - Implementer re-works: status → implementation → verification
     - Verifier re-tests

[After all steps reach status: complete]
  → execute asks for final approval
  → Finalize creates commit
```

### Status Transitions in Mode 1

```
Step 1:
  pending → implementation → verification → awaiting-approval ↓ PAUSE
                                               ↓         ↓
                                           approve   request-changes
                                               ↓         ↓
                                            complete   needs-fix → implementation

Step 2:
  pending → implementation → verification → awaiting-approval ↓ PAUSE
                                               (repeat)

[All steps complete]
    ↓
Execute asks for final approval
    ↓
[FINAL APPROVAL]
    ↓
Finalize → Commit
```

**Key differences from old Mode 1:**
- No automatic status updates; execute controls all transitions
- Step stops at `awaiting-approval` and pauses workflow
- Iteration counter increments on `needs-fix`
- All timestamps recorded in progress.json


### Key Characteristics

- **Frequency of human review:** After EVERY step
- **Execution speed:** Slower (human pauses between steps)
- **Risk level:** Lower (more oversight)
- **Approval points:** N+1 (one per step plus final)
- **Reversal:** Easy (stop at any step, revert if needed)

### Pause and Resume in Mode 1

When a step is in `awaiting-approval`:
- `workflow_status` = `paused`
- User can close Claude Code and come back later
- Run `/workflow:execute` again to resume from same step
- No re-execution of completed steps
- All timestamps preserved in progress.json

This enables asynchronous review workflows where approval may take time.

## Mode 2: End-to-End Execution

### How It Works

```
For each step (in order):
  1. Implementer works on step N (status: implementation)
  2. Implementer finishes and commits (status: verification)
  3. Verifier tests step N (still status: verification)
  4. Verifier passes/fails:
     - Tests pass: status → complete (AUTO, no human pause)
     - Tests fail: status → needs-fix
  
  5. **[IF COMPLETE]** → Continue to next step IMMEDIATELY
     - No workflow pause
     - No human approval gate
     - execute automatically continues loop
  
  6. **[IF NEEDS-FIX]** → Implementer fixes (loop back)
     - Implementer re-works: status → implementation → verification
     - Verifier re-tests

[After all steps reach status: complete]
  → execute asks for final approval
  → Finalize creates commit
```

**Key difference:** When verification passes, status is set directly to `complete` without any `awaiting-approval` pause.

### Status Transitions in Mode 2

```
Step 1:
  pending → implementation → verification → complete (AUTO)
                                              ↓ (no pause)
Step 2:
  pending → implementation → verification → complete (AUTO)
                                              ↓ (no pause)
Step 3:
  pending → implementation → verification → complete (AUTO)
                                              ↓ (no pause)
[All steps complete]
    ↓
Execute asks for final approval
    ↓
[SINGLE HUMAN APPROVAL]
    ↓
Finalize → Commit

[Alternative path if tests fail:]
  verification → needs-fix → implementation → verification → complete
```

**Key difference from Mode 1:**
- No `awaiting-approval` status
- Verification pass directly completes step
- No workflow pauses between steps
- Continuous execution until all complete

### Key Characteristics

- **Frequency of human review:** Once at the end
- **Execution speed:** Faster (no human pauses between steps)
- **Risk level:** Medium (trust-based, but verifier gates each step)
- **Approval points:** 1 (final approval only)
- **Reversal:** Harder (commitment to all steps)

### No Pause/Resume in Mode 2

Mode 2 workflows don't pause between steps:
- If Claude Code closes unexpectedly, run `/workflow:execute` to resume
- Execute continues from the first non-complete step
- All previous step timestamps are preserved in progress.json

## Comparison Table

| Aspect | Mode 1 | Mode 2 |
|--------|--------|--------|
| **Human approvals** | After each step | Once at end |
| **Execution speed** | Slower | Faster |
| **Oversight level** | High | Medium |
| **Risk tolerance** | Low-risk situations | Higher-trust situations |
| **Verifier gates** | After each step | After each step (but no human pauses) |
| **Reversal difficulty** | Easy | Harder |
| **Best for** | Security, architecture | Bugfixes, refactoring |
| **Typical workflow** | Feature building | Quick iterations |

## Decision Matrix: Which Mode to Use?

### Use Mode 1 (Step-by-Step) When:

✓ **Security-sensitive changes**
  - Authentication system
  - Authorization/permissions
  - Payment processing
  - Data access controls

✓ **Architectural changes**
  - Database schema redesign
  - New microservice
  - API contract changes
  - Infrastructure changes

✓ **High-risk modifications**
  - Core business logic changes
  - Breaking changes to existing APIs
  - Large refactoring projects

✓ **Learning/mentoring**
  - Teaching new architecture patterns
  - Code review intensive work
  - Knowledge transfer

✓ **Regulatory/compliance**
  - Regulated industry (fintech, healthcare)
  - Audit trail requirement
  - Change control process

✓ **Uncertain scope**
  - Implementation might reveal issues
  - Requirements might need adjustment
  - Feedback loops important

### Use Mode 2 (End-to-End) When:

✓ **Low-risk bug fixes**
  - Bug fix with clear root cause
  - Simple one-liner fixes
  - Regression fixes

✓ **Well-understood features**
  - Implementation clearly defined
  - Similar to existing features
  - Requirements stable

✓ **Routine refactoring**
  - Code cleanup
  - Dependency updates
  - Test improvements
  - Documentation updates

✓ **Time-sensitive work**
  - Quick hotfixes
  - Small features
  - Urgent production issues

✓ **High-trust teams**
  - Experienced developers
  - Strong code review culture
  - Thorough test coverage already exists

✓ **Stable codebases**
  - Well-established patterns
  - Comprehensive test suite
  - Low bug rate

## Real-World Examples

### Example 1: Adding JWT Authentication

**Scenario:** Building a new authentication system for your API.

**Choice:** Mode 1 (Step-by-Step)

**Why:**
- Security-sensitive changes
- Architectural decisions matter
- Multiple steps with dependencies
- Good checkpoint to review token design

**Workflow:**
```
Step 1: Add token generation
  → [HUMAN APPROVAL] → Review token structure, algorithms

Step 2: Add validation middleware
  → [HUMAN APPROVAL] → Review security scope

Step 3: Add logout endpoint
  → [HUMAN APPROVAL] → Review token cleanup

Step 4: Add tests
  → [HUMAN APPROVAL] → Review test coverage

All approved → Final approval → Commit
```

### Example 2: Fixing Login Timeout Bug

**Scenario:** Users logged out after 2 hours instead of 7 days. It's a clear bug in session expiration logic.

**Choice:** Mode 2 (End-to-End)

**Why:**
- Clear bug, known root cause
- Low-risk fix
- Just modifying expiration config
- Changes are straightforward

**Workflow:**
```
Step 1: Identify session timeout location
  → Implementer changes config
  → Verifier tests with timeout simulation
  [NO HUMAN PAUSE - automatically proceed]

Step 2: Add regression test
  → Implementer writes test case
  → Verifier confirms test passes
  [NO HUMAN PAUSE - automatically proceed]

All steps verified → [SINGLE HUMAN APPROVAL] → Commit
```

### Example 3: Refactoring Database Layer

**Scenario:** Extract database queries into a repository layer for better testing and reusability.

**Choice:** Mode 2 (End-to-End)

**Why:**
- Well-defined refactoring
- Existing test suite validates correctness
- Low behavioral risk (same output)
- Experienced team with strong tests

**Workflow:**
```
Step 1: Extract read queries to repository
  → Verifier: Tests still passing ✓

Step 2: Extract write queries to repository
  → Verifier: Tests still passing ✓

Step 3: Update services to use repository
  → Verifier: Tests still passing ✓

Step 4: Remove direct query calls
  → Verifier: Tests still passing ✓

All steps verified → [SINGLE HUMAN APPROVAL] → Commit
```

## Within-Mode Flexibility

Even within a chosen mode, you have flexibility:

### Mode 1: Skip Approval for Non-Critical Steps

You can approve multiple steps together:
```
Step 1: Complete and reviewed
Step 2: Complete and reviewed
Step 3: Complete and reviewed
[Review all three] → [Single approval to proceed]
```

### Mode 2: Escalate to Manual Review

If verifier finds serious issues:
```
Step 2: Verifier finds critical issue
         [ESCALATE] → Manual investigation
         [After resolution] → Resume execution
```

## Switching Modes Mid-Workflow

**Not recommended** for ongoing workflows, but possible:

1. Stop execution
2. Update PLAN.md mode field
3. Update .workflow-config.json mode field
4. Resume with new mode

This affects:
- How many more approval gates you'll hit
- Pacing of remaining steps
- Agent signaling behavior

## Performance Implications

### Mode 1 Typical Timeline (4 steps)

```
Step 1: Implementation (20 min) + Verification (10 min)
        → [HUMAN REVIEW] (5 min) → [APPROVAL] (2 min)
        Subtotal: ~37 min

Step 2: Implementation (25 min) + Verification (10 min)
        → [HUMAN REVIEW] (5 min) → [APPROVAL] (2 min)
        Subtotal: ~42 min

Step 3: Implementation (20 min) + Verification (10 min)
        → [HUMAN REVIEW] (5 min) → [APPROVAL] (2 min)
        Subtotal: ~37 min

Step 4: Implementation (15 min) + Verification (10 min)
        → [HUMAN REVIEW] (5 min) → [APPROVAL] (2 min)
        Subtotal: ~32 min

Final Approval: 5 min
Finalize: 2 min

TOTAL: ~155 minutes (2.6 hours)
Human involvement: 4 approval gates + 4 reviews = ~28 minutes
```

### Mode 2 Typical Timeline (4 steps)

```
All steps execute sequentially:
Step 1: Implementation (20 min) + Verification (10 min)
        → [NO PAUSE] Continue to step 2

Step 2: Implementation (25 min) + Verification (10 min)
        → [NO PAUSE] Continue to step 3

Step 3: Implementation (20 min) + Verification (10 min)
        → [NO PAUSE] Continue to step 4

Step 4: Implementation (15 min) + Verification (10 min)
        → [ALL COMPLETE]

Final Approval: 10 min (single comprehensive review)
Finalize: 2 min

TOTAL: ~132 minutes (2.2 hours)
Human involvement: 1 final approval = ~10 minutes
```

**Mode 2 is ~23 minutes faster** (saves human approval time, less pauses).

## Advanced: Custom Decision Points

While not officially supported in v1.0, you can manually create approval points within Mode 2:

1. After a step, **mark it complete**
2. **Stop execution** (don't run `/workflow:execute`)
3. **Review manually** (read step files, code changes)
4. **Resume** when ready (run `/workflow:execute`)

This gives you Mode 2's speed with selective approval points.

## Recovery and Rollback

### Mode 1 Rollback (Easier)

If you catch an issue during human approval:

```
Step N: Complete but you see a problem
  → [REJECT] Don't approve
  → Implementer goes back to fix
  → Verifier re-tests
  → Previous steps already committed
  → Easy to rollback by reverting one commit
```

### Mode 2 Rollback (Harder)

If you approve all steps but then find issues:

```
All steps: Verified and approved
  → You review and find issue
  → [REQUEST CHANGES]
  → Implementer fixes step N
  → Verifier re-tests step N
  → All steps re-verified
  → All steps re-approved
  → Many commits to manage, harder to rollback
```

**Strategy:** In Mode 2, do a thorough final review before approval.

## Recommended Practices

### Before Choosing Mode:

1. **Assess risk** — What's the failure impact?
2. **Assess complexity** — Are requirements clear?
3. **Assess team experience** — How well-known is this pattern?
4. **Check codebase stability** — Do similar changes exist? How reliable are tests?
5. **Consider time** — How much time can you invest in approvals?

### During Execution:

1. **In Mode 1:** Read each step carefully before approval
2. **In Mode 2:** Do final review comprehensively (read all step files and code diffs)
3. **In both:** Don't approve if you're unsure

### After Completion:

1. **In Mode 1:** Ensure test suite covers the changes
2. **In Mode 2:** Run full integration tests after approval
3. **Both modes:** Merge carefully, monitor in production

---

**Next Steps:**
- [QUICKSTART.md](QUICKSTART.md) — Get started with your first workflow
- [workflow-format.md](workflow-format.md) — Understand file formats
- See `examples/` for real workflow examples
