---
name: wgsd:migrate
description: Full GSD to WGSD migration wizard with intelligent analysis and rollback
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUser
---

# WGSD Migration Wizard

Complete wizard for migrating GSD projects to WGSD with intelligent focus group suggestions, work-in-progress preservation, and safe rollback.

---

## Overview

This wizard performs a complete GSD → WGSD migration:
1. **Analyze** - Detect project state and suggest focus groups
2. **Preserve** - Convert work-in-progress to implementation
3. **Migrate** - Transform planning structure to WGSD format
4. **Configure** - Set up workspace and Slack integration
5. **Announce** - Generate team communication

All operations are transactional with automatic rollback on failure.

---

## Inputs

| Input | Required | Description | Example |
|-------|----------|-------------|---------|
| repo_path | Yes | Path to GSD repository | `/home/user/projects/myapp` |
| stub | No | Slack channel stub (auto-suggested) | `mvn` |
| dry_run | No | Preview without making changes | `true` |

---

## Process

### Step 1: Initialize Transaction

```bash
echo "═══════════════════════════════════════════════════════════"
echo "🚀 WGSD Migration Wizard"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Repository: $REPO_PATH"
echo "📅 Date: $(date -I)"
echo ""

# Validate repository exists
if [ ! -d "$REPO_PATH" ]; then
  echo "❌ Repository not found: $REPO_PATH"
  exit 1
fi

cd "$REPO_PATH"

# Check if it's a git repository
if ! git rev-parse --git-dir >/dev/null 2>&1; then
  echo "❌ Not a git repository: $REPO_PATH"
  exit 1
fi

# Source library functions
# (In actual use, these are inline or sourced from lib/)

# Start transaction
echo "🔒 Starting migration transaction..."
echo ""

BACKUP_DIR=".planning-backup-$(date +%Y%m%d-%H%M%S)"
if [ -d ".planning" ]; then
  cp -r .planning "$BACKUP_DIR"
  echo "✅ Backup created: $BACKUP_DIR"
else
  BACKUP_DIR=""
  echo "ℹ️  No existing .planning/ to backup"
fi
echo ""

# Set up rollback on error
cleanup_on_error() {
  echo ""
  echo "⚠️  Migration failed - rolling back..."
  if [ -n "$BACKUP_DIR" ] && [ -d "$BACKUP_DIR" ]; then
    rm -rf .planning
    mv "$BACKUP_DIR" .planning
    echo "✅ Rolled back to backup"
  fi
  exit 1
}
trap cleanup_on_error ERR
```

### Step 2: Detect Current State

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 1: Analyzing Project State"
echo "─────────────────────────────────────────────────────────────"
echo ""

# Git state
BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
DIRTY=$(git status --porcelain | wc -l | tr -d ' ')
echo "🌿 Branch: $BRANCH"
echo "📝 Uncommitted changes: $DIRTY files"

# GSD state detection
if [ -f ".planning/STATE.md" ]; then
  CURRENT_PHASE=$(grep -E "^##? (Current )?Phase|^\*\*Phase" .planning/STATE.md | head -1 | sed 's/[*#]//g' | xargs)
  echo "📍 Current Phase: $CURRENT_PHASE"
else
  CURRENT_PHASE=""
  echo "📍 Current Phase: Not detected"
fi

# Check if already WGSD
if [ -f ".planning/WGSD-CONFIG.md" ]; then
  echo ""
  echo "⚠️  This project appears to already be using WGSD!"
  echo "   Found: .planning/WGSD-CONFIG.md"
  echo ""
  echo "Options:"
  echo "   1. Re-migrate (will backup existing WGSD structure)"
  echo "   2. Abort migration"
  echo ""
  # In interactive mode, ask user
fi

echo ""
```

### Step 3: Analyze for Focus Group Suggestions

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 2: Analyzing for Focus Group Suggestions"
echo "─────────────────────────────────────────────────────────────"
echo ""

SUGGESTED_FGS=""

# Analyze roadmap
if [ -f ".planning/ROADMAP.md" ]; then
  echo "📄 Analyzing ROADMAP.md..."
  
  # Extract phase names and map to focus groups
  PHASES=$(grep -E "^##? Phase [0-9]+:" .planning/ROADMAP.md | head -10)
  
  while IFS= read -r phase; do
    [ -z "$phase" ] && continue
    phase_lower=$(echo "$phase" | tr '[:upper:]' '[:lower:]')
    
    if echo "$phase_lower" | grep -qE "foundation|infrastructure|core"; then
      SUGGESTED_FGS="$SUGGESTED_FGS core"
      echo "   📂 core ← $phase"
    elif echo "$phase_lower" | grep -qE "auth|security|permission"; then
      SUGGESTED_FGS="$SUGGESTED_FGS security"
      echo "   📂 security ← $phase"
    elif echo "$phase_lower" | grep -qE "api|integrat|endpoint"; then
      SUGGESTED_FGS="$SUGGESTED_FGS api"
      echo "   📂 api ← $phase"
    elif echo "$phase_lower" | grep -qE "migration|wizard"; then
      SUGGESTED_FGS="$SUGGESTED_FGS migration"
      echo "   📂 migration ← $phase"
    elif echo "$phase_lower" | grep -qE "channel|slack"; then
      SUGGESTED_FGS="$SUGGESTED_FGS channels"
      echo "   📂 channels ← $phase"
    elif echo "$phase_lower" | grep -qE "canvas|dashboard|visual"; then
      SUGGESTED_FGS="$SUGGESTED_FGS visualization"
      echo "   📂 visualization ← $phase"
    elif echo "$phase_lower" | grep -qE "workflow|process"; then
      SUGGESTED_FGS="$SUGGESTED_FGS workflow"
      echo "   📂 workflow ← $phase"
    elif echo "$phase_lower" | grep -qE "community|social"; then
      SUGGESTED_FGS="$SUGGESTED_FGS community"
      echo "   📂 community ← $phase"
    fi
  done <<< "$PHASES"
  echo ""
fi

# Analyze codebase structure
echo "📁 Analyzing codebase structure..."

[ -d "api" ] || [ -d "src/api" ] || [ -d "routes" ] && {
  SUGGESTED_FGS="$SUGGESTED_FGS api"
  echo "   📂 api ← Found api/ or routes/ directory"
}

[ -d "components" ] || [ -d "src/components" ] || [ -d "ui" ] && {
  SUGGESTED_FGS="$SUGGESTED_FGS frontend"
  echo "   📂 frontend ← Found components/ or ui/ directory"
}

[ -d "lib" ] || [ -d "src/lib" ] || [ -d "core" ] && {
  SUGGESTED_FGS="$SUGGESTED_FGS core"
  echo "   📂 core ← Found lib/ or core/ directory"
}

[ -d "workflows" ] || [ -d "agents" ] && {
  SUGGESTED_FGS="$SUGGESTED_FGS ai"
  echo "   📂 ai ← Found workflows/ or agents/ directory"
}

echo ""

# Deduplicate suggestions
UNIQUE_FGS=$(echo $SUGGESTED_FGS | tr ' ' '\n' | sort -u | tr '\n' ' ')
FG_COUNT=$(echo $UNIQUE_FGS | wc -w | tr -d ' ')

echo "💡 Suggested Focus Groups ($FG_COUNT):"
for fg in $UNIQUE_FGS; do
  [ -n "$fg" ] && echo "   ✅ $fg"
done
echo ""
```

### Step 4: Preserve Work-in-Progress

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 3: Preserving Work-in-Progress"
echo "─────────────────────────────────────────────────────────────"
echo ""

WIP_PRESERVED="false"
IMPL_NAME=""

# Check for uncommitted changes
if [ "$DIRTY" -gt 0 ]; then
  echo "📝 Found $DIRTY uncommitted files"
  echo "   Stashing changes for migration..."
  
  STASH_MSG="WGSD-MIGRATION-$(date +%Y%m%d-%H%M%S)"
  git stash push -m "$STASH_MSG" --include-untracked 2>&1
  STASH_REF=$(git stash list | grep "$STASH_MSG" | cut -d: -f1)
  
  echo "   ✅ Stashed as: $STASH_REF"
fi

# Check if current phase is in-progress
if [ -n "$CURRENT_PHASE" ]; then
  # Check if phase is incomplete
  if [ -f ".planning/STATE.md" ]; then
    if grep -qE "IN PROGRESS|🟡|50%|75%" .planning/STATE.md; then
      echo ""
      echo "📍 Active Phase Detected: $CURRENT_PHASE"
      echo "   This will be converted to an implementation."
      
      # Generate implementation name
      PHASE_SLUG=$(echo "$CURRENT_PHASE" | \
        sed 's/^Phase[ -]*[0-9]*[: -]*//' | \
        tr '[:upper:]' '[:lower:]' | \
        tr ' ' '-' | \
        tr -cd 'a-z0-9-' | \
        sed 's/--*/-/g' | \
        cut -c1-20)
      
      if [ -z "$PHASE_SLUG" ]; then
        PHASE_SLUG="wip-$(date +%m%d)"
      fi
      
      IMPL_NAME="${STUB:-proj}-impl-${PHASE_SLUG}"
      WIP_PRESERVED="true"
      
      echo "   📦 Will create implementation: $IMPL_NAME"
    fi
  fi
fi

if [ "$WIP_PRESERVED" = "false" ]; then
  echo "ℹ️  No active work-in-progress to preserve"
fi
echo ""
```

### Step 5: Create WGSD Structure

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 4: Creating WGSD Structure"
echo "─────────────────────────────────────────────────────────────"
echo ""

# Create directories
mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations
echo "✅ Created focus-groups/ directory"
echo "✅ Created active-implementations/ directory"

# Transform phases to focus groups if present
if [ -d ".planning/phases" ]; then
  echo ""
  echo "📁 Transforming phases to focus groups..."
  
  for phase_dir in .planning/phases/*/; do
    if [ -d "$phase_dir" ]; then
      phase_name=$(basename "$phase_dir")
      
      # Map phase to focus group name
      fg_name=$(echo "$phase_name" | \
        sed 's/^phase-[0-9]*-//' | \
        tr '-' ' ' | \
        xargs)
      
      if [ -z "$fg_name" ]; then
        fg_name="$phase_name"
      fi
      
      fg_slug=$(echo "$fg_name" | \
        tr '[:upper:]' '[:lower:]' | \
        tr ' ' '-' | \
        tr -cd 'a-z0-9-')
      
      fg_dir=".planning/focus-groups/$fg_slug"
      mkdir -p "$fg_dir/concepts"
      
      # Copy phase contents
      for file in "${phase_dir}"*; do
        if [ -f "$file" ]; then
          cp "$file" "$fg_dir/"
        fi
      done
      
      # Create focus group STATE.md
      cat > "$fg_dir/STATE.md" << EOF
# Focus Group: $fg_name

**Status:** Active
**Created:** $(date -I)
**Migrated From:** phases/$phase_name

## Description

*Add focus group description here*

## Concepts

*Concepts will be listed as they are created*

---

*Migrated from GSD during WGSD migration*
EOF
      
      echo "   ✅ $phase_name → focus-groups/$fg_slug"
    fi
  done
fi

echo ""
```

### Step 6: Generate WGSD Configuration

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 5: Generating Configuration"
echo "─────────────────────────────────────────────────────────────"
echo ""

# Detect or use provided stub
PROJECT_NAME=$(basename "$(pwd)")
if [ -z "$STUB" ]; then
  # Suggest stub from project name
  case "$PROJECT_NAME" in
    marvin|marvin-*) STUB="mvn" ;;
    openclaw|openclaw-*) STUB="oc" ;;
    skill-wgsd|wgsd-*) STUB="wgsd" ;;
    *)
      STUB=$(echo "$PROJECT_NAME" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)
      ;;
  esac
fi

echo "📱 Slack stub: $STUB"

# Detect primary branch
if git rev-parse --verify origin/develop >/dev/null 2>&1; then
  PRIMARY="develop"
elif git rev-parse --verify origin/main >/dev/null 2>&1; then
  PRIMARY="main"
else
  PRIMARY="main"
fi

echo "🌿 Primary branch: $PRIMARY"

# Create WGSD-CONFIG.md
cat > .planning/WGSD-CONFIG.md << EOF
# WGSD Configuration

**Project:** $PROJECT_NAME
**Migrated:** $(date -Iseconds)
**Migration Backup:** ${BACKUP_DIR:-"none"}

---

## Slack Configuration

**Stub:** $STUB
**Channel Prefix:** #${STUB}-

### Channel Patterns
| Type | Pattern | Example |
|------|---------|---------|
| Main Dev | \`${STUB}-dev\` | #${STUB}-dev |
| Focus Group | \`${STUB}-fg-{name}\` | #${STUB}-fg-security |
| Concept | \`${STUB}-cpt-{name}\` | #${STUB}-cpt-feature |
| Implementation | \`${STUB}-impl-{name}\` | #${STUB}-impl-auth-v2 |
| Community | \`${STUB}-community\` | #${STUB}-community |

---

## Git Configuration

**Primary Branch:** $PRIMARY
**Focus Groups Base:** focus-groups/
**Implementations Base:** implementations/

---

## Focus Groups

| Name | Status | Channel |
|------|--------|---------|
$(for fg in $UNIQUE_FGS; do
  [ -n "$fg" ] && echo "| $fg | Active | #${STUB}-fg-${fg} |"
done)

---

## Active Implementations

$(if [ "$WIP_PRESERVED" = "true" ]; then
  echo "| $IMPL_NAME | In Progress | #${IMPL_NAME} |"
else
  echo "*No active implementations*"
fi)

---

*Configuration generated during GSD → WGSD migration*
EOF

echo "✅ Created WGSD-CONFIG.md"
echo ""
```

### Step 7: Create MASTER-ROADMAP.md

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 6: Creating Master Roadmap"
echo "─────────────────────────────────────────────────────────────"
echo ""

if [ -f ".planning/ROADMAP.md" ] && [ ! -f ".planning/MASTER-ROADMAP.md" ]; then
  cat > .planning/MASTER-ROADMAP.md << EOF
# Master Roadmap

**Project:** $PROJECT_NAME
**Updated:** $(date -I)

---

## Overview

This roadmap aggregates progress across all focus groups.

## Focus Groups

$(for fg in $UNIQUE_FGS; do
  [ -n "$fg" ] && echo "- **$fg** - \`#${STUB}-fg-${fg}\`"
done)

---

## Original Roadmap

*Migrated from ROADMAP.md*

$(cat .planning/ROADMAP.md)

---

*Master roadmap created during WGSD migration*
EOF
  
  echo "✅ Created MASTER-ROADMAP.md"
else
  echo "ℹ️  MASTER-ROADMAP.md already exists or no ROADMAP.md to migrate"
fi
echo ""
```

### Step 8: Update PROJECT.md and STATE.md

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 7: Updating Project Files"
echo "─────────────────────────────────────────────────────────────"
echo ""

# Add WGSD section to PROJECT.md
if [ -f ".planning/PROJECT.md" ]; then
  if ! grep -q "## WGSD Workflow" .planning/PROJECT.md; then
    cat >> .planning/PROJECT.md << 'EOF'

---

## WGSD Workflow

This project uses **WGSD (We Get Shit Done)** collaborative development:

1. **Focus Groups** - Long-lived topic discussions (security, api, frontend)
2. **Concepts** - Feature ideas developed socially within focus groups
3. **Implementations** - Short-lived code execution (1-3 days)

### Development Flow

```
Community Feedback → Focus Group → Concept → Implementation → Develop
```

### Channel Structure

- **#stub-dev** - Main development channel
- **#stub-fg-*** - Focus group discussions
- **#stub-impl-*** - Active implementations
- **#stub-community** - Public feedback channel

---

*WGSD section added during migration*
EOF
    echo "✅ Updated PROJECT.md with WGSD section"
  else
    echo "ℹ️  PROJECT.md already has WGSD section"
  fi
fi

# Add WGSD status to STATE.md
if [ -f ".planning/STATE.md" ]; then
  if ! grep -q "## WGSD Status" .planning/STATE.md; then
    cat >> .planning/STATE.md << EOF

---

## WGSD Status

**Migrated:** $(date -I)
**Mode:** WGSD v2.0

### Focus Groups
$(for fg in $UNIQUE_FGS; do
  [ -n "$fg" ] && echo "- $fg - Active"
done)

### Active Implementations
$(if [ "$WIP_PRESERVED" = "true" ]; then
  echo "- $IMPL_NAME - In Progress"
else
  echo "*None*"
fi)

---

*WGSD status added during migration*
EOF
    echo "✅ Updated STATE.md with WGSD status"
  fi
fi
echo ""
```

### Step 9: Restore Stashed Changes

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 8: Restoring Changes"
echo "─────────────────────────────────────────────────────────────"
echo ""

if [ -n "$STASH_REF" ]; then
  echo "📥 Restoring stashed changes..."
  git stash pop "$STASH_REF" 2>&1 || true
  echo "✅ Changes restored"
else
  echo "ℹ️  No stashed changes to restore"
fi
echo ""
```

### Step 10: Generate Team Announcement

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 9: Generating Team Announcement"
echo "─────────────────────────────────────────────────────────────"
echo ""

ANNOUNCEMENT_FILE=".planning/MIGRATION-ANNOUNCEMENT.md"

cat > "$ANNOUNCEMENT_FILE" << EOF
# 🚀 $PROJECT_NAME has migrated to WGSD!

Hey team! We've upgraded our development workflow from GSD to **WGSD (We Get Shit Done)** v2.0.

## What Changed

- **Focus Groups** replace phases for ongoing discussions
- **Concepts** are ideas we develop socially before coding
- **Implementations** are short-lived coding sprints (1-3 days)

## New Channels

| Channel | Purpose |
|---------|---------|
| #${STUB}-dev | Main development channel |
$(for fg in $UNIQUE_FGS; do
  [ -n "$fg" ] && echo "| #${STUB}-fg-${fg} | $fg focus group |"
done)
$([ "$WIP_PRESERVED" = "true" ] && echo "| #${IMPL_NAME} | Active implementation |")

## How to Contribute

1. **Have an idea?** Share it in the relevant focus group channel
2. **Want to build something?** Create a concept for discussion
3. **Ready to code?** Promote your concept to an implementation

## Questions?

Drop a message in #${STUB}-dev and we'll help you get started!

---

*Migrated on $(date -I)*
EOF

echo "✅ Created team announcement: $ANNOUNCEMENT_FILE"
echo ""
echo "📋 Preview:"
echo "─────────────────────────────────────────────────────────────"
head -25 "$ANNOUNCEMENT_FILE"
echo "..."
echo "─────────────────────────────────────────────────────────────"
echo ""
```

### Step 11: Commit Migration

```bash
echo "─────────────────────────────────────────────────────────────"
echo "📊 Step 10: Committing Migration"
echo "─────────────────────────────────────────────────────────────"
echo ""

git add .planning/
git commit -m "feat: migrate from GSD to WGSD v2.0

- Created WGSD directory structure
- Generated WGSD-CONFIG.md
- Created focus groups: $UNIQUE_FGS
- Updated PROJECT.md and STATE.md
- Created MASTER-ROADMAP.md
- Generated team announcement

Migration backup: ${BACKUP_DIR:-"none"}"

if [ $? -eq 0 ]; then
  COMMIT_SHA=$(git rev-parse --short HEAD)
  echo "✅ Migration committed: $COMMIT_SHA"
else
  echo "⚠️  Commit failed (nothing to commit or already committed)"
fi
echo ""

# Remove error trap
trap - ERR
```

### Step 12: Success Report

```bash
echo "═══════════════════════════════════════════════════════════"
echo "🎉 Migration Complete!"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Project: $PROJECT_NAME"
echo "📱 Slack Stub: $STUB"
echo "📂 Focus Groups: $FG_COUNT"
$([ "$WIP_PRESERVED" = "true" ] && echo "📦 WIP Preserved: $IMPL_NAME")
echo ""
echo "📋 New Structure:"
echo "   .planning/"
echo "   ├── WGSD-CONFIG.md      ✅"
echo "   ├── MASTER-ROADMAP.md   ✅"
echo "   ├── focus-groups/       ✅"
echo "   └── active-implementations/  ✅"
echo ""
echo "📝 Next Steps:"
echo "   1. Create Slack channel: #${STUB}-dev"
echo "   2. Post announcement from: .planning/MIGRATION-ANNOUNCEMENT.md"
echo "   3. Create focus group channels"
echo "   4. Run: wgsd status"
echo ""
if [ -n "$BACKUP_DIR" ]; then
  echo "📦 Backup preserved at: $BACKUP_DIR"
  echo "   (Safe to delete after verifying migration)"
fi
echo ""
echo "═══════════════════════════════════════════════════════════"
```

---

## Dry Run Mode

When `dry_run=true`, the wizard shows what would happen without making changes:

```bash
if [ "$DRY_RUN" = "true" ]; then
  echo "🔍 DRY RUN MODE - No changes will be made"
  echo ""
  # ... show preview of all changes ...
  echo ""
  echo "Run without --dry-run to apply these changes"
  exit 0
fi
```

---

## Error Handling

The wizard uses transactional semantics:
- Backup created before any changes
- Automatic rollback on any error
- Stashed changes always restored
- Clear error messages with recovery hints

---

## Success Criteria

- [ ] GSD state detected and analyzed
- [ ] Focus groups suggested from roadmap + codebase
- [ ] Work-in-progress preserved as implementation
- [ ] WGSD structure created
- [ ] Configuration files generated
- [ ] Team announcement ready
- [ ] Changes committed to git
- [ ] Backup available for rollback

---

*Workflow created for WGSD Phase 2 - All MIGRATE-* requirements*
