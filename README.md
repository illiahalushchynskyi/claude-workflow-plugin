# Workflow System Plugin

A comprehensive AI-driven development workflow system for Claude Code that manages multi-step implementation tasks with two operational modes, persistent state tracking, and cross-machine portability.

## What is the Workflow System?

The Workflow System provides a structured framework for executing complex development tasks using paired AI agents:

- **Implementer Agent** — executes code changes for each step
- **Verifier Agent** — tests changes and gates advancement to the next step

All progress is tracked in markdown files with structured frontmatter, making workflows portable across machines and git-trackable.

## Key Features

### Two Operational Modes

**Mode 1: Step-by-Step Verification** (safer, more oversight)
- After each step completes: implementer works → verifier tests → human reviews
- Requires explicit human approval before advancing to next step
- Full visibility at every checkpoint

**Mode 2: End-to-End Execution** (faster, trust-based)
- All steps execute automatically without human pauses between steps
- Verifier gates each step sequentially (previous step must be `complete` before next starts)
- Single human review/approval at the end of all steps
- Each step still verified, but no mid-workflow interruptions

### Advanced Features

- **Schema Validation** — All PLAN.md and step files validated against JSON schemas
- **Cross-Machine Portability** — File-based state, works on any machine with git and Claude Code
- **Persistent Audit Trail** — Every iteration timestamped, all decisions recorded
- **Issue Tracking & Fixes** — Verifier reports issues, implementer fixes in same cycle
- **Mode Flexibility** — Choose mode before work begins, applies to entire workflow

## Installation

### Method 1: Clone to Claude Code Plugins Directory (Recommended)

```bash
# Create plugins directory if it doesn't exist
mkdir -p ~/.claude/plugins

# Clone the plugin repository
git clone https://github.com/yourusername/workflow-plugin.git ~/.claude/plugins/workflow

# Verify installation
claude-code --list-skills | grep workflow
```

You should see 5 skills: `workflow:bootstrap`, `workflow:implementer`, `workflow:verifier`, `workflow:execute`, `workflow:finalize`

### Method 2: Download Released Plugin Package

```bash
cd ~/.claude/plugins
# Download latest release from GitHub
unzip workflow-plugin-v1.0.0.zip
mv workflow-plugin-v1.0.0 workflow
```

### Method 3: Add Plugin Path to Claude Code Settings

Edit `~/.claude/settings.json`:

```json
{
  "plugins": {
    "paths": [
      "~/.claude/plugins/workflow",
      "/path/to/workflow-plugin"
    ]
  }
}
```

See [INSTALLATION.md](docs/INSTALLATION.md) for detailed setup and troubleshooting.

## Quick Start Example

### 1. Bootstrap a New Workflow

```bash
# Use the workflow:bootstrap skill in Claude Code
/workflow:bootstrap

# Prompts you for:
# - Task name (e.g., "feature-auth-system")
# - Task description and requirements
# - Mode selection (1 or 2)
# - Number of steps

# Creates:
# .workflow/feature-auth-system/PLAN.md
# .workflow/feature-auth-system/.workflow-config.json
# .workflow/feature-auth-system/steps/step-1-*.md, step-2-*.md, etc.
```

### 2. Execute the Workflow

```bash
# Use the workflow:execute skill
/workflow:execute

# Reads your PLAN.md, determines mode, and:
# - Mode 1: Pauses after each step for human approval
# - Mode 2: Runs all steps, pauses only at the end
```

### 3. View Progress and Approve

After execution:
- Read `.workflow/feature-auth-system/steps/step-*.md` to review implementer and verifier notes
- Check `.workflow/feature-auth-system/PLAN.md` for overall status
- Approve or request changes

### 4. Finalize

```bash
# Use the workflow:finalize skill
/workflow:finalize

# Creates final commit with summary of all changes
# Updates PLAN.md status to "complete"
```

## Documentation

- **[QUICKSTART.md](docs/QUICKSTART.md)** — Step-by-step guide to creating your first workflow
- **[MODE_GUIDE.md](docs/MODE_GUIDE.md)** — Detailed explanation of Mode 1 vs Mode 2, decision matrix
- **[workflow-format.md](docs/workflow-format.md)** — Reference for PLAN.md and step file formats
- **[INSTALLATION.md](docs/INSTALLATION.md)** — Detailed installation and machine setup
- **[TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** — Common issues and recovery procedures

## Example Workflows

The `examples/` directory includes three pre-built workflows:

1. **feature-auth-system/** — Building a complete authentication system (Mode 1 example)
2. **bugfix-login-timeout/** — Fixing a specific bug with quick iteration (Mode 2 example)
3. **refactor-db-layer/** — Refactoring database layer with comprehensive testing (Mode 1 example)

Each includes:
- Pre-written PLAN.md with realistic steps
- Step detail files showing expected structure
- Verification criteria examples

## Skills Reference

### `workflow:bootstrap`
Initialize a new workflow task with PLAN.md and step files.

**Usage:** `/workflow:bootstrap`

**Input:** Task description, mode, number of steps

**Output:** Complete workflow directory structure

### `workflow:implementer`
Execute implementation for a specific workflow step.

**Usage:** `/workflow:implementer`

**Input:** Reads from `.workflow/TASK_NAME/steps/step-N.md`

**Output:** Modified code, updated step file with implementation notes

### `workflow:verifier`
Test and verify a completed step.

**Usage:** `/workflow:verifier`

**Input:** Reads verification criteria from `.workflow/TASK_NAME/steps/step-N.md`

**Output:** Test results, updates step file; gates advancement to next step

### `workflow:execute`
Orchestrate entire workflow execution (Mode 1 or 2).

**Usage:** `/workflow:execute`

**Input:** Reads `.workflow/TASK_NAME/PLAN.md` for mode and step list

**Output:** Manages agent sequencing, pauses for human approval as needed

### `workflow:finalize`
Create final commit after workflow completion.

**Usage:** `/workflow:finalize`

**Input:** All `.workflow/TASK_NAME/steps/*.md` files

**Output:** Final commit with summary, updates PLAN.md status

## Plugin Structure

```
workflow-plugin/
├── plugin.json                 # Plugin manifest
├── README.md                   # This file
├── package.json                # npm metadata (for lib utilities)
├── skills/
│   ├── bootstrap.md           # Skill definitions with YAML frontmatter
│   ├── implementer.md
│   ├── verifier.md
│   ├── execute.md
│   └── finalize.md
├── lib/
│   ├── fileValidation.ts      # Schema validation utilities
│   └── utils.ts               # Helper functions
├── schema/
│   ├── plan-schema.json       # JSON Schema for PLAN.md
│   ├── step-schema.json       # JSON Schema for step-N.md
│   └── config-schema.json     # JSON Schema for .workflow-config.json
├── docs/
│   ├── workflow-format.md     # File format reference
│   ├── QUICKSTART.md          # Getting started guide
│   ├── MODE_GUIDE.md          # Mode explanation and decision matrix
│   ├── INSTALLATION.md        # Installation instructions
│   └── TROUBLESHOOTING.md     # Common issues and fixes
└── examples/
    ├── feature-auth-system/
    │   ├── PLAN.md
    │   └── steps/
    ├── bugfix-login-timeout/
    │   ├── PLAN.md
    │   └── steps/
    └── refactor-db-layer/
        ├── PLAN.md
        └── steps/
```

## Getting Started on a New Machine

Once installed, using workflows on a new machine is straightforward:

```bash
# 1. Install Claude Code
# (see INSTALLATION.md)

# 2. Install workflow plugin (same 3 methods as above)

# 3. Clone your project repo
git clone https://github.com/yourorg/your-project.git
cd your-project

# 4. Workflow directory is already there with all progress
ls -la .workflow/
# Shows: feature-auth-system/, bugfix-login-timeout/, etc.

# 5. Skills are available — continue from last state
/workflow:execute
```

No setup needed. All workflow state is in git.

## Architecture & Design

The Workflow System uses a **file-based state layer** with three core components:

1. **State Layer** — PLAN.md and step-N.md files with structured frontmatter
2. **Agent Layer** — Paired agents (Implementer + Verifier) per step
3. **Orchestration Layer** — Mode-aware step sequencing and human approval gates

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for deep dive into design decisions.

## Contributing

Contributions welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License — see LICENSE file for details.

## Support

- **Documentation:** See `docs/` folder
- **Examples:** See `examples/` folder
- **Issues:** Report bugs on GitHub
- **Troubleshooting:** See [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)

---

**Version:** 1.0.0
**Last Updated:** 2026-04-08
