# Execute Skill: Agent Dispatch Rules

**CRITICAL PATTERN: This prevents the most common workflow execution error.**

## The Problem

When running `/workflow:execute`, Claude must dispatch `implementer` and `verifier` as **isolated subagents**, NOT call them directly as skills.

### ❌ WRONG Pattern (Fails)

```python
# Claude in main session trying to work directly:
Skill(skill: "workflow:implementer")  # ❌ WRONG - Direct skill call
Skill(skill: "workflow:verifier")     # ❌ WRONG - Direct skill call

# Or: Claude trying to implement/verify itself
# (reading code, writing changes, running tests)  # ❌ WRONG - Orchestrator shouldn't work
```

### ✅ CORRECT Pattern (Works)

```python
# Claude dispatches subagent with skill reference:
Agent(
  description: "Implement step 1: add auth",
  subagent_type: "general-purpose",
  prompt: """
You are a subagent implementing step 1 of auth-system.

**YOUR TASK:**
1. Read: .workflow/auth-system/steps/step-1.md
2. Invoke: Skill(skill: "workflow:implementer")
3. Follow the skill procedure exactly

**CONTEXT:**
- Task: auth-system
- Step: 1 (Add token service)
- projectType: node
- buildCommand: npm run build
- testCommand: npm test
"""
)
```

## Why This Matters

The workflow system has distinct roles:

| Role | Who | What They Do |
|------|-----|--------------|
| **execute** | Claude in main session | Orchestrator: reads state, dispatches agents, asks user |
| **implementer** | Subagent (isolated) | Worker: writes code, runs tests, commits, updates step file |
| **verifier** | Subagent (isolated) | Worker: tests implementation, verifies criteria, updates step file |

**The orchestrator never does implementation or verification work itself.**

## The Three Agent Patterns

### Pattern 1: Dispatch Implementer (from execute)

```python
# In main session, execute reads workflow config:
PROJECT_TYPE = "node"      # from .workflow-config.json
BUILD_CMD = "npm run build"
TEST_CMD = "npm test"
TASK_NAME = "auth-system"
STEP = 1

# Then dispatch Agent:
Agent(
  description: f"Implement {TASK_NAME} step {STEP}",
  subagent_type: "general-purpose",
  prompt: f"""
You are implementing step {STEP} of {TASK_NAME}.

**TASK:**
1. Read: .workflow/{TASK_NAME}/steps/step-{STEP}.md
2. Invoke: Skill(skill: "workflow:implementer")
3. Follow the skill procedure

**CONFIG:**
- Project: {PROJECT_TYPE}
- Build: {BUILD_CMD}
- Test: {TEST_CMD}
"""
)

# Wait for Agent() to complete
# Then read: .workflow/{TASK_NAME}/steps/step-{STEP}.md
# Check status field is now: verification
```

### Pattern 2: Dispatch Verifier (from execute)

```python
# In main session, execute dispatches verifier:
Agent(
  description: f"Verify {TASK_NAME} step {STEP}",
  subagent_type: "general-purpose",
  prompt: f"""
You are verifying step {STEP} of {TASK_NAME}.

**TASK:**
1. Read: .workflow/{TASK_NAME}/steps/step-{STEP}.md
2. Invoke: Skill(skill: "workflow:verifier")
3. Follow the skill procedure

**CONFIG:**
- Project: {PROJECT_TYPE}
- Build: {BUILD_CMD}
- Test: {TEST_CMD}
"""
)

# Wait for Agent() to complete
# Then read: .workflow/{TASK_NAME}/steps/step-{STEP}.md
# Check status field is now: complete OR needs-fix
```

### Pattern 3: Finalize (from execute)

```python
# In main session, execute can call this directly:
Skill(skill: "workflow:finalize")  # ✅ CORRECT - Main session direct call

# This is the ONLY direct Skill() call from execute
```

## Detection: How to Know If You're Doing It Wrong

| Question | Yes = Problem | No = Correct |
|----------|---------------|-------------|
| "Am I calling `Skill(skill: "workflow:implementer")` directly?" | ❌ Wrong | ✅ Right |
| "Am I calling `Skill(skill: "workflow:verifier")` directly?" | ❌ Wrong | ✅ Right |
| "Am I writing code myself in the main session?" | ❌ Wrong | ✅ Right |
| "Am I running `npm test` myself in the main session?" | ❌ Wrong | ✅ Right |
| "Did I dispatch Agent() for implementer?" | ✅ Right | ❌ Wrong |
| "Did I dispatch Agent() for verifier?" | ✅ Right | ❌ Wrong |
| "Does my Agent prompt include `Invoke: Skill(...)`?" | ✅ Right | ❌ Wrong |

## Recovery: If Execute Isn't Working

1. **Check the dispatch pattern** — Is execute using `Agent()` for implementer/verifier?
2. **Check the prompt** — Does it include `Invoke: Skill(skill: "workflow:implementer")`?
3. **Check the flow** — Does execute wait for Agent() to return before reading step files?
4. **Check role separation** — Is execute reading config and coordinating? Or is it trying to work?

If you see `Skill()` direct calls to implementer/verifier, that's the bug. Replace with `Agent()` dispatch.

## Key Insight

**The orchestrator (`execute`) is a coordinator, not a worker.**

- Coordinates step sequencing
- Manages state files (PLAN.md, progress.json)
- Delegates work to subagents
- Asks humans for approval
- Never writes code, runs tests, or makes implementation decisions

The subagents (implementer, verifier) are workers:
- implementer: Writes code, runs tests, commits, updates step file
- verifier: Tests code, verifies criteria, updates step file

When you see yourself writing code in the main session, you've broken the orchestrator pattern.

---

See [MODE_GUIDE.md](MODE_GUIDE.md) for overall workflow modes and [workflow-format.md](workflow-format.md) for file format details.
