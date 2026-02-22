---
name: wgsd:setup-repo
description: Initialize WGSD structure in a repository
argument-hint: "[repo-path]"
allowed-tools:
  - Read
  - Write
  - Bash
---

<objective>
Initialize WGSD (We Get Shit Done) collaborative development structure in a repository.
Creates shared .planning/ directory, configures git branches, and prepares for focus groups.
</objective>

<process>

## Step 1: Validate Repository

```bash
cd {repo-path}
if [ ! -d .git ]; then
    echo "❌ Not a git repository"
    exit 1
fi

echo "📁 Repository: $(basename $(pwd))"
echo "🌿 Current branch: $(git branch --show-current)"
```

## Step 2: Create Planning Structure

```bash
mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations
mkdir -p concepts
mkdir -p implementations
```

Create core planning files:

**.planning/PROJECT.md:**
```markdown
# {Repo Name} - Collaborative Development

**Repository:** {repo-path}
**WGSD Initialized:** {date}
**Main Development Channel:** #{repo-name}-dev

## Objective

{Brief description of the project}

## Development Workflow

This repository uses **WGSD (We Get Shit Done)** collaborative development:

1. **Focus Groups**: Long-lived topic discussions in dedicated channels
2. **Concepts**: Feature ideas developed socially within focus groups  
3. **Implementations**: Short-lived execution of mature concepts (2-4 max)

See #{repo-name}-dev channel canvas for detailed workflow.

## Focus Groups

{Will be populated as focus groups are created}

## Active Implementations

{Will be populated as implementations are started}

---

*Powered by WGSD - We Get Shit Done together*
```

**.planning/WGSD-CONFIG.md:**
```markdown
# WGSD Configuration

**Repository:** {repo-path}
**Initialized:** {date}

## Channel Mappings

### Main Development
- **Channel:** #{repo-name}-dev
- **Purpose:** Community hub, master roadmap, team coordination

### Focus Groups
{Will be populated as focus groups are created}

### Active Implementations  
{Will be populated as implementations are started}

## Git Structure

**Branches:**
- `main/develop` - Primary development branch
- `focus-groups/{name}` - Planning and concept development
- `implementations/{name}` - Code implementation branches

**Worktrees:**
- `concepts/{focus-group}/` - Focus group worktree directories
- `implementations/{name}/` - Implementation worktree directories

## Limits & Rules

- **Max Concurrent Implementations:** 4
- **Focus Group Naming:** lowercase-with-dashes
- **Implementation Lifecycle:** 1-3 days maximum
- **Canvas Sync:** Bidirectional between Slack and git

## Version Strategy

**Focus Groups:** No versions, states only (Draft → Exploring → Mature → Ready)
**Implementations:** Semantic versioning determined at merge time
- Format: `v{major}.{minor}.{patch}-{description}`
- Independent of planning order, based on merge reality
```

**.planning/MASTER-ROADMAP.md:**
```markdown
# Master Roadmap - {Repo Name}

**Last Updated:** {date}
**Status:** Initialized

## Focus Groups Overview

*No focus groups created yet*

## Active Implementations

*No implementations running*

## Implementation Queue

*No concepts ready for implementation*

---

Use `/wgsd roadmap` to update this file automatically.
```

## Step 3: Initialize Git Configuration

```bash
# Add .planning/ files to git
git add .planning/
git commit -m "feat: initialize WGSD collaborative development structure"

echo "✅ WGSD structure initialized"
echo "📋 Files created:"
echo "   .planning/PROJECT.md"
echo "   .planning/WGSD-CONFIG.md" 
echo "   .planning/MASTER-ROADMAP.md"
```

## Step 4: Create Initial Branches

```bash
# Create and push focus-groups base branch (for future focus group branches)
git checkout -b focus-groups/base
echo "# Focus Groups Base Branch" > .focus-groups-readme.md
git add .focus-groups-readme.md
git commit -m "feat: create focus groups base branch"
git push -u origin focus-groups/base

# Create and push implementations base branch
git checkout -b implementations/base  
echo "# Implementations Base Branch" > .implementations-readme.md
git add .implementations-readme.md
git commit -m "feat: create implementations base branch"
git push -u origin implementations/base

# Return to main branch
git checkout develop 2>/dev/null || git checkout main
git push origin HEAD
```

## Step 5: Workspace Setup Complete

```bash
echo ""
echo "🎉 WGSD Repository Setup Complete!"
echo ""
echo "📁 Repository: $(basename $(pwd))"
echo "📋 Planning structure: .planning/"
echo "🌿 Base branches: focus-groups/base, implementations/base"
echo ""
echo "▶️  Next Steps:"
echo "   1. Create main development channel: #{repo-name}-dev"
echo "   2. Add team members and explain WGSD workflow"
echo "   3. Create first focus group: /wgsd create-focus-group [name]"
echo ""
echo "💡 The shared .planning/ directory enables cross-channel collaboration"
echo "   All focus groups and implementations reference the same planning files"
echo ""
```

## Step 6: Return Status

Update caller with initialization results and next steps.

</process>

<success_criteria>
- [ ] .planning/ directory structure created with all base files
- [ ] Git branches for focus-groups and implementations initialized
- [ ] Repository committed with WGSD structure
- [ ] Worktree directories prepared (concepts/ and implementations/)
- [ ] Configuration files ready for focus group creation
- [ ] Clear next steps provided for team onboarding
</success_criteria>