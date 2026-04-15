# Execution Modes: Subagent vs Manual

When you run `/workflow:execute`, you choose how to execute your workflow:

## Quick Comparison

| Aspect | Subagent | Current Session |
|--------|--------------|-------------|
| **What runs** | `Agent(implementer)` then `Agent(verifier)` | `Skill(workflow:implementer)` then `Skill(workflow:verifier)` |
| **Isolation** | Isolated subagent context | This main session |
| **Speed** | Fast (parallel possible) | Slow (sequential) |
| **Visibility** | Less visible | Very visible |
| **Best for** | Speed, automation | Debugging, transparency |
| **Risk** | If Agent breaks, unclear | Low (you see everything) |

---

## Subagent (Recommended) ⚡

When you choose **Subagent**, I dispatch implementer and verifier as isolated Agent subagents.

### How It Works

```
You run: /workflow:execute
    ↓
Choose: "Subagent Mode"
    ↓
I dispatch: Agent(implementer) ← Isolated context
    ↓ (agent runs, returns)
Read step-1.md: status is now verification
    ↓
I dispatch: Agent(verifier) ← Isolated context
    ↓ (agent runs, returns)
Read step-1.md: status is now complete
    ↓
[Ask for approval in Mode 1]
    ↓
[Continue to next step]
```

### Advantages

✅ Fast - agents can work in parallel (future enhancement)
✅ Hands-off - set and forget execution
✅ Isolated - subagents don't affect main session
✅ Cleaner - clear separation of roles

### Disadvantages

❌ Less visible - can't see agent working
❌ Harder to debug - agent output less visible
❌ If Agent dispatch breaks - unclear error

### When to Use Subagent Mode

```
✅ You trust the Agent mechanism is working
✅ You want fast execution
✅ You don't need to inspect each step
✅ This is a "set it and forget it" task
✅ You have other work to do while it runs
```

---

## Current Session 🎯

When you choose **Current Session**, I invoke implementer and verifier as Skills in this main session.

### How It Works

```
You run: /workflow:execute
    ↓
Choose: "Manual Mode"
    ↓
I invoke: Skill(workflow:implementer) ← In this session
    ↓
[You implement step 1...]
[Skill guides you...]
[You update step file...]
↓
Read step-1.md: status is now verification
    ↓
I invoke: Skill(workflow:verifier) ← In this session
    ↓
[You verify step 1...]
[Skill guides you...]
[You update step file...]
    ↓
Read step-1.md: status is now complete
    ↓
[Ask for approval in Mode 1]
    ↓
[Continue to next step]
```

### Advantages

✅ Maximum visibility - you see every action
✅ Maximum control - you make every decision
✅ Easy to debug - you see all output
✅ Transparent - clear what's happening
✅ No Agent issues - no agent dispatch problems

### Disadvantages

❌ Slower - back-and-forth with prompts
❌ More tedious - lots of confirmations
❌ Not automated - requires interaction

### When to Use Manual Mode

```
✅ You want full visibility
✅ You're debugging a workflow
✅ Agent dispatch isn't working
✅ You prefer step-by-step execution
✅ You want to inspect each step closely
✅ You're learning how workflows work
```

---

## Decision Matrix

### Use Subagent If:

- [ ] You've used workflows before and they worked
- [ ] You're confident in your task definition
- [ ] You don't need to inspect each step
- [ ] You want fast execution
- [ ] You have time for something else while it runs

**→ Choose Subagent**

### Use Current Session If:

- [ ] This is your first workflow
- [ ] You're debugging a problem
- [ ] You want maximum visibility
- [ ] You've had Agent dispatch issues
- [ ] You want to review each step
- [ ] You like hands-on control

**→ Choose Current Session**

---

## Switching Modes

If you start with one mode and want to switch:

1. Note which step you're on (check progress.json)
2. Run `/workflow:execute` again
3. Choose the other mode
4. Execute will resume from the incomplete step

The modes are interchangeable - a Manual step can be followed by a Subagent step, or vice versa.

---

## Examples

### Example 1: Subagent - Feature Workflow

```
Task: Add authentication system (3 steps)
Mode: 1 (Step-by-Step approval)
Execution: Subagent (fast)

Step 1: Add token service
  → Agent(implementer) [10 min]
  → Agent(verifier) [5 min]
  → User approval [2 min]

Step 2: Add login endpoint
  → Agent(implementer) [12 min]
  → Agent(verifier) [5 min]
  → User approval [2 min]

Step 3: Add tests
  → Agent(implementer) [10 min]
  → Agent(verifier) [5 min]
  → User approval [2 min]

Final: Finalize → commit

Total: ~55 min (mostly automated)
```

### Example 2: Current Session - Debugging

```
Task: Fix login timeout bug (2 steps)
Mode: 2 (End-to-End)
Execution: Manual (transparent)

Step 1: Fix session expiration
  → I invoke Skill(workflow:implementer)
  → You read & implement
  → You update step file [15 min]
  → I read step file ✓
  → I invoke Skill(workflow:verifier)
  → You test & verify [10 min]
  → I read step file ✓

Step 2: Add regression test
  → I invoke Skill(workflow:implementer)
  → You write test [10 min]
  → I invoke Skill(workflow:verifier)
  → You verify test works [5 min]

Final: Finalize → commit

Total: ~40 min (fully transparent)
```

---

## Troubleshooting by Mode

### Subagent: "Agent didn't work"

**Check:**
1. Can you run a simple `Agent()` call? (Test with /verify or similar)
2. Does Agent() return at all?
3. Check step file - did agent update status?

**Solutions:**
1. Switch to Current Session (transparent, no agent issues)
2. Check Agent tool is working
3. Inspect step file Implementation/Verification sections for error notes

### Current Session: "Skill execution is slow"

**This is expected** - Manual Mode is slower because:
- You read the goal
- You implement
- You run tests
- You update file
- I read file
- Repeat

This is by design for transparency.

**Solutions:**
1. Switch to Subagent Mode if you want speed
2. Or accept slower, more visible execution

### Both Modes: "Step file status isn't changing"

**Check:**
1. Is the skill completing without errors?
2. Did you (or the agent) set the status field?
3. Is the YAML frontmatter correct?

**Solutions:**
1. Read the step file Implementation/Verification section for error notes
2. Check the skill ran successfully
3. Manually update status if needed and continue

---

## Key Insight

Both modes execute the **same implementer and verifier** procedures. The **only difference** is:

- **Subagent:** Implementer/Verifier run as isolated Agent subagents (faster, less visible)
- **Current Session:** Implementer/Verifier run as Skills in this session (slower, more visible)

Choose based on your needs:
- Need speed? **Subagent**
- Need visibility? **Current Session**
- Not sure? **Start with Current Session** (transparent), switch to Subagent once working

---

See [MODE_GUIDE.md](MODE_GUIDE.md) for workflow modes (Mode 1 vs Mode 2).
See [EXECUTE_AGENT_DISPATCH.md](EXECUTE_AGENT_DISPATCH.md) for technical dispatch patterns.
