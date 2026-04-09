# Installation & Setup Guide

Complete guide to installing the Workflow System plugin on your machine and setting up your first project.

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Installation Methods](#installation-methods)
3. [Verification](#verification)
4. [Project Setup](#project-setup)
5. [Cross-Machine Setup](#cross-machine-setup)
6. [Troubleshooting](#troubleshooting)

## System Requirements

### Required

- **Claude Code** (see [claude.ai/code](https://claude.ai/code))
- **Git** v2.0 or later (for version control)
- **Bash** or compatible shell (zsh, fish, etc. work fine)

### Supported Operating Systems

- Linux (Ubuntu, Debian, RHEL, Alpine, etc.)
- macOS (Intel and Apple Silicon)
- Windows (WSL2 recommended)
- Docker/container environments

### Optional (for specific project types)

- **Node.js** v16+ (if implementing Node.js workflows)
- **Python** 3.7+ (if implementing Python workflows)
- **Docker** (if implementing containerized workflows)

## How the Plugin Works

Once installed, the 5 workflow skills become available in Claude Code:

- `/workflow:bootstrap` — Create a new workflow task
- `/workflow:implementer` — Execute implementation for a step
- `/workflow:verifier` — Test and verify a step
- `/workflow:execute` — Orchestrate workflow execution
- `/workflow:finalize` — Create final commit

Type any of these in Claude Code to start using them.

## Installation Methods

Choose **one** of the three methods below:

### Method 1: Clone from Git Repository (Recommended)

Best for: Developers, keeping up with updates

```bash
# Create plugins directory if it doesn't exist
mkdir -p ~/.claude/plugins

# Clone the workflow plugin
git clone https://github.com/illiahalushchynskyi/claude-workflow-plugin.git ~/.claude/plugins/workflow

# Verify (should show workflow skills)
ls ~/.claude/plugins/workflow/skills/ | grep -E "^[a-z]+\.md$"
```

**Output should show:**
```
bootstrap.md
implementer.md
verifier.md
execute.md
finalize.md
```

**Updating in the future:**
```bash
cd ~/.claude/plugins/workflow
git pull
```

### Method 2: Download and Extract Release Package

Best for: Non-developers, offline installations

```bash
# Download latest release
cd /tmp
wget https://github.com/yourusername/workflow-plugin/releases/download/v1.0.0/workflow-plugin-v1.0.0.zip
# OR using curl:
curl -L -O https://github.com/yourusername/workflow-plugin/releases/download/v1.0.0/workflow-plugin-v1.0.0.zip

# Extract to plugins directory
mkdir -p ~/.claude/plugins
unzip workflow-plugin-v1.0.0.zip -d ~/.claude/plugins
mv ~/.claude/plugins/workflow-plugin-v1.0.0 ~/.claude/plugins/workflow

# Verify
ls ~/.claude/plugins/workflow/skills/ | grep -E "^[a-z]+\.md$"
```

### Method 3: Add Plugin Path to Settings

Best for: Development, custom locations

Edit `~/.claude/settings.json`:

```json
{
  "plugins": {
    "paths": [
      "~/.claude/plugins",
      "/path/to/workflow-plugin",
      "/opt/workflow-plugin"
    ]
  }
}
```

Then restart Claude Code to load the new plugin path from settings.json.

Verify:
```bash
ls ~/.claude/plugins/workflow/
# Should show: plugin.json, README.md, skills/, docs/, schema/, examples/
```

## Verification

### Quick Verification (30 seconds)

Check that skill files are present:

```bash
ls ~/.claude/plugins/workflow/skills/
```

Should show all 5 skill files:
- `bootstrap.md`
- `implementer.md`
- `verifier.md`
- `execute.md`
- `finalize.md`

### Detailed Verification (2 minutes)

**In Claude Code:** Type `/workflow:bootstrap` or any workflow skill to test

If the skill loads and shows its prompt, the plugin is installed correctly.

Alternatively, verify plugin structure:

Check plugin directory structure:

```bash
ls -la ~/.claude/plugins/workflow/
```

Should show:
- `plugin.json`
- `README.md`
- `skills/` (directory)
- `docs/` (directory)
- `schema/` (directory)
- `examples/` (directory)

Check all skill files present:

```bash
ls ~/.claude/plugins/workflow/skills/
```

Should show 5 files:
- `bootstrap.md`
- `implementer.md`
- `verifier.md`
- `execute.md`
- `finalize.md`

**Test in Claude Code:** Try typing `/workflow:bootstrap` — if it loads, the plugin is working!

## Project Setup

After installing the plugin, set up your project for workflows.

### 1. Initialize Git (if not already done)

```bash
cd /path/to/your/project
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### 2. Create Workflow Directory

```bash
mkdir -p .workflow
git add .workflow/.gitkeep
git commit -m "feat: initialize workflow directory"
```

### 3. Create First Workflow

In Claude Code:

```
/workflow:bootstrap
```

Follow prompts:
- Task Name: `my-first-workflow`
- Task Title: "My First Workflow"
- Mode: 1 (recommended for first time)
- Steps: 2 (keep it small for learning)

## Cross-Machine Setup

### Scenario: You started work on Machine A, continuing on Machine B

**Machine A (Initial):**

```bash
# Have the plugin installed
# Created first workflow with bootstrap
# Implemented step 1, verified
git commit -m "workflow: complete step 1"
git push origin feature-branch
```

**Machine B (Continuation):**

```bash
# 1. Install Claude Code
# (see Claude Code documentation)

# 2. Install workflow plugin (any of 3 methods above)
# Method 1 recommended:
mkdir -p ~/.claude/plugins
git clone https://github.com/illiahalushchynskyi/claude-workflow-plugin.git ~/.claude/plugins/workflow

# 3. Clone your project
git clone https://github.com/yourorg/your-project.git
cd your-project

# 4. Pull latest changes (including .workflow/)
git pull origin feature-branch

# 5. Check workflow status
cat .workflow/my-first-workflow/PLAN.md
cat .workflow/my-first-workflow/steps/step-1-*.md

# 6. Continue execution
/workflow:execute

# Implementer will work on step 2
```

**Key point:** All workflow state is in `.workflow/` and tracked by git. No database, no external state.

## Multi-Machine Workflow

### Team Example: 3 developers on same feature

**Developer A (Day 1):**
```bash
# Clone project, install plugin
git clone <project>
cd <project>

# Create workflow
/workflow:bootstrap
# → Creates feature-auth feature workflow

# Execute step 1
/workflow:execute
# → Step 1 implemented and verified

# Commit and push
git push origin workflow-feature-auth

# Slack: "Step 1 complete, ready for step 2"
```

**Developer B (Day 2, picks up from A):**
```bash
# Clone project, install plugin
git clone <project>
cd <project>

# Pull latest workflow
git pull origin workflow-feature-auth

# Check status
cat .workflow/feature-auth/PLAN.md
# Shows: Step 1 complete, Step 2 pending

# Continue
/workflow:execute
# → Step 2 implemented and verified

# Push
git push origin workflow-feature-auth
```

**Developer C (Day 3, final step):**
```bash
# Same process
git clone <project>
cd <project>
git pull origin workflow-feature-auth

# Check status
cat .workflow/feature-auth/PLAN.md
# Shows: Steps 1-2 complete, Step 3 pending

# Execute step 3
/workflow:execute
# → Step 3 implemented and verified

# Finalize workflow
/workflow:finalize
# → Creates final commit

# Create PR and merge
git push origin workflow-feature-auth
# Create PR from GitHub/GitLab/Gitea
# Merge when approved
```

## Troubleshooting Installation

### Problem: "Skills not loading" or "/workflow:bootstrap not found"

**Solution:**
1. Verify plugin is installed:
   ```bash
   ls -la ~/.claude/plugins/workflow/plugin.json
   # Should exist and be readable
   ```
2. Verify all skills are present:
   ```bash
   ls ~/.claude/plugins/workflow/skills/ | wc -l
   # Should show 5
   ```
3. Restart Claude Code (close and reopen the application)
4. Try using a skill: `/workflow:bootstrap`

### Problem: "Plugin directory not found"

**Solution:**
1. Verify plugin path:
   ```bash
   ls -la ~/.claude/plugins/workflow/
   # Should have: plugin.json, skills/, docs/, etc.
   ```
2. Check skills directory:
   ```bash
   ls -la ~/.claude/plugins/workflow/skills/
   # Should have: bootstrap.md, implementer.md, etc.
   ```
3. Verify settings.json is correct:
   ```bash
   cat ~/.claude/settings.json | grep -A 3 "plugins"
   ```
4. Restart Claude Code

### Problem: "Permission denied" when cloning

**Solution:**
```bash
# Create plugins directory if it doesn't exist
mkdir -p ~/.claude/plugins
chmod 755 ~/.claude/plugins

# Retry clone with SSH key or HTTPS with credentials
git clone git@github.com:yourusername/workflow-plugin.git ~/.claude/plugins/workflow
# OR
git clone https://yourusername:token@github.com/yourusername/workflow-plugin.git ~/.claude/plugins/workflow
```

### Problem: "Git is not installed"

**Solution:**
```bash
# Linux (Debian/Ubuntu)
sudo apt-get install git

# Linux (RHEL/CentOS)
sudo yum install git

# macOS
brew install git

# Windows
choco install git
# OR download from https://git-scm.com/download/win
```

### Problem: ".workflow directory not tracked by git"

**Solution:**
```bash
# Add .workflow directory to git
cd /path/to/your/project
git add .workflow/
git commit -m "workflow: track workflow directory"

# If not showing up:
# Check .gitignore doesn't exclude it
grep -v "^\.workflow" .gitignore > .gitignore.tmp && mv .gitignore.tmp .gitignore

# Verify it's tracked
git ls-files | grep "\.workflow"
```

### Problem: "Settings.json not found"

**Solution:**
```bash
# Create settings.json if it doesn't exist
mkdir -p ~/.claude
cat > ~/.claude/settings.json << 'EOF'
{
  "plugins": {
    "paths": ["~/.claude/plugins"]
  }
}
EOF

# Restart Claude Code
```

## Uninstallation

### Method 1: Remove from ~/.claude/plugins

```bash
rm -rf ~/.claude/plugins/workflow
```

### Method 2: Remove from Settings

Edit `~/.claude/settings.json`, remove workflow path from plugins.paths

### Cleanup Workflows

Remove workflow tasks from your project:

```bash
# Backup first
cp -r .workflow .workflow.backup

# Remove
rm -rf .workflow
git commit -m "workflow: remove workflow tasks"
```

## Next Steps

After successful installation:

1. **Read [QUICKSTART.md](QUICKSTART.md)** — Get started with your first workflow
2. **Create first workflow** — Run `/workflow:bootstrap` in Claude Code
3. **Execute workflow** — Run `/workflow:execute`
4. **Read [MODE_GUIDE.md](MODE_GUIDE.md)** — Understand Mode 1 vs 2

## Support

- **Installation issues:** See Troubleshooting section above
- **How to use:** See [QUICKSTART.md](QUICKSTART.md)
- **Detailed reference:** See [workflow-format.md](workflow-format.md)
- **Mode selection:** See [MODE_GUIDE.md](MODE_GUIDE.md)
- **Common issues:** See [TROUBLESHOOTING.md](TROUBLESHOOTING.md)

---

**Version:** 1.0.0
**Last Updated:** 2026-04-08
