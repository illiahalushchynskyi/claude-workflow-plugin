# Workflow Execute: Execution Modes

When you run `/workflow:execute`, Claude will ask you to choose how to execute your workflow:

```
How should implementer and verifier work?

A) Subagent Mode (Recommended)
   I dispatch implementer/verifier as isolated subagents via Agent().
   You handle orchestration only.

B) Manual Mode
   I guide you to run workflow:implementer and workflow:verifier directly.
   You invoke each skill manually.
```

## Subagent Mode (Recommended) ⚡

**How it works:**
1. You run `/workflow:execute`
2. Execute asks: "Subagent Mode or Manual Mode?"
3. You choose: "Subagent Mode (Recommended)"
4. Execute automatically dispatches implementer and verifier as Agent subagents
5. Implementer runs, updates step file with status `verification`
6. Verifier runs, updates step file with status `complete` or `needs-fix`
7. Execute reads files, determines next action
8. Repeat until all steps complete
9. Execute asks for final approval and runs finalize

**Visual flow:**
```
/workflow:execute
    ↓
"Choose mode?"
    ↓
Subagent Mode (user clicks)
    ↓
Claude dispatches Agent(implementer)  ← Isolated subagent
    ↓
Step file status: verification
    ↓
Claude dispatches Agent(verifier)  ← Isolated subagent
    ↓
Step file status: complete/needs-fix
    ↓
Loop or approve or ask user for Mode 1 approval
```

**Advantages:**
- ✅ Fastest execution
- ✅ Fully automated once started
- ✅ Cleaner separation of concerns
- ✅ Orchestrator never writes code
- ✅ Recommended approach

**Disadvantages:**
- ❌ Less visible during execution
- ❌ If Agent dispatch breaks, workflow may fail
- ❌ Harder to debug if Agent doesn't work

**Best for:**
- You trust the Agent mechanism
- You want fast, hands-off execution
- You're confident the workflow will work
- You don't need step-by-step visibility

---

## Manual Mode 🎯

**How it works:**
1. You run `/workflow:execute`
2. Execute asks: "Subagent Mode or Manual Mode?"
3. You choose: "Manual Mode"
4. Execute asks: "Ready to implement step 1?"
5. You run: `/workflow:implementer` (in another Claude session or tab)
6. You return and confirm: "Done implementing"
7. Execute reads step file to verify status
8. Execute asks: "Ready to verify step 1?"
9. You run: `/workflow:verifier` (in another Claude session or tab)
10. You return and confirm: "Done verifying"
11. Execute reads step file to verify status
12. Repeat for each step
13. Execute asks for final approval
14. You run: `/workflow:finalize`
15. You return and confirm: "Done finalizing"

**Visual flow:**
```
/workflow:execute
    ↓
"Choose mode?"
    ↓
Manual Mode (user clicks)
    ↓
"Ready to implement step 1?"
    ↓
Open new tab/session
    ↓
/workflow:implementer (you run this in new session)
    ↓
Return to execute tab
    ↓
"Have you run /workflow:implementer?"
    ↓
✓ Yes, confirmed
    ↓
Execute reads step file
    ↓
Status should be: verification
    ↓
"Ready to verify step 1?"
    ↓
Open new tab/session
    ↓
/workflow:verifier (you run this in new session)
    ↓
Return to execute tab
    ↓
"Have you run /workflow:verifier?"
    ↓
✓ Yes, confirmed
    ↓
Loop to next step...
```

**Advantages:**
- ✅ Full visibility at every step
- ✅ Complete control over timing
- ✅ Easy to debug if something goes wrong
- ✅ Can inspect/review between steps
- ✅ More transparent process
- ✅ No Agent dispatch issues

**Disadvantages:**
- ❌ Much slower (lots of back-and-forth)
- ❌ Requires multiple Claude sessions/tabs
- ❌ More tedious (more clicking)
- ❌ Not recommended for large workflows

**Best for:**
- You want full visibility and control
- You're debugging a workflow
- Agent dispatch isn't working
- You prefer transparent, step-by-step execution
- You have time and want to inspect each step

---

## How to Choose

### Use Subagent Mode If:

```
✅ You trust the Agent mechanism
✅ You want fast execution
✅ You don't need to inspect each step
✅ You're confident in your workflow
✅ This is a "run and forget" task
✅ You have other work to do while it runs
```

### Use Manual Mode If:

```
✅ You want full visibility
✅ You're debugging or testing the workflow
✅ Agent dispatch isn't working
✅ You want to review each step
✅ You prefer slow and steady execution
✅ You need to inspect/fix between steps
✅ You're learning how the workflow works
```

---

## Switching Modes Mid-Workflow

If you're in Manual Mode and want to switch to Subagent Mode:

1. Note which step you're on (check progress.json)
2. Stop the current execution
3. Run `/workflow:execute` again
4. Choose "Subagent Mode"
5. Execute will resume from the incomplete step

The opposite (Subagent → Manual) is also possible but less common.

---

## Practical Example: Manual Mode Walkthrough

Let's say you have a 3-step workflow: `feature-auth`

### Step 1: Start Execute

```bash
/workflow:execute
```

Claude asks:
```
How should implementer and verifier work?

A) Subagent Mode (Recommended)
B) Manual Mode
```

You click: **Manual Mode**

Claude asks:
```
Start workflow from step 1 (Mode 1, manual invocation)?

Options:
- Yes, I'll run skills manually
- No, stop
```

You click: **Yes, I'll run skills manually**

Claude says:
```
Step 1 status: pending

Ready to implement? Run: /workflow:implementer
```

### Step 2: Implement Step 1

You open a new Claude tab/window and run:

```bash
/workflow:implementer
```

The implementer skill guides you:
1. Read step-1.md goal
2. Make code changes
3. Run tests (must pass)
4. Commit changes
5. Update step-1.md Implementation section
6. **Set status: verification**

When done, you return to the execute tab.

### Step 3: Confirm Implement

Claude is waiting:
```
Have you run /workflow:implementer and confirmed status is verification?

Options:
- Yes, I've confirmed
- No, let me check
```

You click: **Yes, I've confirmed**

Claude reads step-1.md:
```
✓ Step 1 status is verification

Ready to verify? Run: /workflow:verifier
```

### Step 4: Verify Step 1

You open a new Claude tab/window and run:

```bash
/workflow:verifier
```

The verifier skill guides you:
1. Build project
2. Run tests (all must pass)
3. Verify acceptance criteria
4. Check migrations
5. Update step-1.md Verification section
6. **Set status: complete**

When done, you return to the execute tab.

### Step 5: Confirm Verify

Claude is waiting:
```
Have you run /workflow:verifier and confirmed status is complete?

Options:
- Yes, I've confirmed
- No, let me check
```

You click: **Yes, I've confirmed**

Claude reads step-1.md:
```
✓ Step 1 status is complete

**Mode 1 Approval Gate:**
Step 1 verification passed. Approve and continue to step 2?

Options:
- Approve and continue
- Request changes (back to implementation)
```

You click: **Approve and continue**

Claude:
```
✓ Step 1 approved
→ Step 2 status: pending

Ready to implement step 2? Run: /workflow:implementer
```

### Steps 2 & 3: Repeat Process

You repeat the same process for steps 2 and 3.

### Final Step: Finalize

When all steps are complete:

```
✓ All 3 steps complete

Ready to finalize? Run: /workflow:finalize
```

You run `/workflow:finalize` to create the final commit.

---

## Troubleshooting by Mode

### Subagent Mode Issues

**Problem:** Execute says "Dispatching implementer..." but nothing happens

**Solutions:**
1. Check if Agent() tool is working (test with a simple Agent call)
2. Try switching to Manual Mode instead
3. Check Claude Code logs for Agent errors

**Problem:** Execute completes but step file status didn't change

**Solutions:**
1. Switch to Manual Mode for better visibility
2. Manually run `/workflow:implementer` to check for errors
3. Review step file Implementation section for notes

### Manual Mode Issues

**Problem:** Execute asks "Have you run /workflow:implementer?" but you forgot

**Solutions:**
1. Click "No, let me check"
2. Open new tab and run `/workflow:implementer`
3. Return and click "Yes, I've confirmed"

**Problem:** Step file status is wrong after you ran the skill

**Solutions:**
1. Check what the skill output was
2. Manually update the step-N.md status field if needed
3. Return to execute and confirm

---

See [MODE_GUIDE.md](MODE_GUIDE.md) for workflow modes (Mode 1 vs Mode 2).
See [EXECUTE_AGENT_DISPATCH.md](EXECUTE_AGENT_DISPATCH.md) for Agent vs Skill patterns.
