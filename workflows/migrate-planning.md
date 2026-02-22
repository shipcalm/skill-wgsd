---
name: wgsd:migrate-planning
description: Migrate GSD planning structure to WGSD format
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUser
---

# GSD to WGSD Planning Migration

Transform existing GSD `.planning/` structure to WGSD format while preserving all content.

---

## Objective

Migrate a GSD-structured project to WGSD by:
- Detecting existing GSD planning files
- Creating backup before modifications
- Transforming content to WGSD structure
- Generating WGSD configuration files
- Preserving all original content

---

## GSD to WGSD Structure Mapping

| GSD Source | WGSD Target | Transformation |
|------------|-------------|----------------|
| PROJECT.md | PROJECT.md | Add WGSD workflow section |
| ROADMAP.md | MASTER-ROADMAP.md | Rename, add focus group refs |
| REQUIREMENTS.md | focus-groups/*/concepts/ | Split by domain |
| STATE.md | STATE.md | Add focus group status section |
| phases/* | focus-groups/* | Rename, restructure |
| codebase/* | codebase/* | Preserve unchanged |

---

## Process

### Step 1: Detect GSD Structure

```bash
echo "🔍 Detecting GSD planning structure..."

if [ ! -d ".planning" ]; then
  echo "⚠️  No .planning/ directory found"
  echo "   Creating fresh WGSD structure..."
  GSD_MODE="fresh"
else
  GSD_MODE="migrate"
  
  # Check for GSD files
  GSD_FILES=""
  [ -f ".planning/PROJECT.md" ] && GSD_FILES="$GSD_FILES PROJECT.md"
  [ -f ".planning/ROADMAP.md" ] && GSD_FILES="$GSD_FILES ROADMAP.md"
  [ -f ".planning/REQUIREMENTS.md" ] && GSD_FILES="$GSD_FILES REQUIREMENTS.md"
  [ -f ".planning/STATE.md" ] && GSD_FILES="$GSD_FILES STATE.md"
  [ -d ".planning/phases" ] && GSD_FILES="$GSD_FILES phases/"
  [ -d ".planning/codebase" ] && GSD_FILES="$GSD_FILES codebase/"
  
  if [ -z "$GSD_FILES" ]; then
    echo "⚠️  .planning/ exists but contains no GSD files"
    GSD_MODE="partial"
  else
    echo "✅ Found GSD files:$GSD_FILES"
  fi
fi
```

### Step 2: Create Backup

```bash
if [ "$GSD_MODE" != "fresh" ] && [ -d ".planning" ]; then
  BACKUP_DIR=".planning-backup-$(date +%Y%m%d-%H%M%S)"
  echo "📦 Creating backup: $BACKUP_DIR"
  cp -r .planning "$BACKUP_DIR"
  echo "✅ Backup created"
fi
```

### Step 3: Create WGSD Directory Structure

```bash
echo "📁 Creating WGSD directory structure..."

mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations

echo "✅ Directory structure created"
```

### Step 4: Transform PROJECT.md

```bash
if [ -f ".planning/PROJECT.md" ]; then
  echo "📝 Enhancing PROJECT.md with WGSD section..."
  
  # Check if WGSD section already exists
  if ! grep -q "## WGSD Workflow" .planning/PROJECT.md; then
    cat >> .planning/PROJECT.md << 'WGSD_SECTION'

---

## WGSD Workflow

This project uses **WGSD (We Get Shit Done)** collaborative development:

1. **Focus Groups** - Long-lived topic discussions (security, onboarding, billing)
2. **Concepts** - Feature ideas developed socially within focus groups
3. **Implementations** - Short-lived code execution (1-3 days)

### Active Focus Groups

*List of focus groups will be populated as they are created.*

### Current Implementations

*List of active implementations (max 2-4 concurrent).*

### Slack Integration

Channels follow the pattern: `#{stub}-{type}[-{name}]`
- Main: `#{stub}-dev`
- Focus Groups: `#{stub}-fg-{name}`
- Implementations: `#{stub}-impl-{name}`

---

*WGSD section added during migration*
WGSD_SECTION
    
    echo "✅ PROJECT.md enhanced"
  else
    echo "ℹ️  PROJECT.md already has WGSD section"
  fi
fi
```

### Step 5: Transform ROADMAP.md to MASTER-ROADMAP.md

```bash
if [ -f ".planning/ROADMAP.md" ] && [ ! -f ".planning/MASTER-ROADMAP.md" ]; then
  echo "📝 Converting ROADMAP.md to MASTER-ROADMAP.md..."
  
  # Add header and rename
  {
    echo "# Master Roadmap"
    echo ""
    echo "**WGSD Project Roadmap** - Aggregated from all focus groups"
    echo ""
    echo "---"
    echo ""
    echo "*Migrated from GSD ROADMAP.md on $(date -I)*"
    echo ""
    cat .planning/ROADMAP.md
  } > .planning/MASTER-ROADMAP.md
  
  # Keep original as reference (don't delete)
  echo "✅ MASTER-ROADMAP.md created (original ROADMAP.md preserved)"
fi
```

### Step 6: Transform Phases to Focus Groups

```bash
if [ -d ".planning/phases" ]; then
  echo "📝 Transforming phases to focus groups..."
  
  for phase_dir in .planning/phases/*/; do
    if [ -d "$phase_dir" ]; then
      phase_name=$(basename "$phase_dir")
      
      # Create focus group directory
      fg_dir=".planning/focus-groups/$phase_name"
      mkdir -p "$fg_dir"
      mkdir -p "$fg_dir/concepts"
      
      # Copy phase contents
      if [ -f "${phase_dir}PLAN.md" ]; then
        cp "${phase_dir}PLAN.md" "$fg_dir/ROADMAP.md"
      fi
      
      # Copy any other files
      for file in "${phase_dir}"*; do
        if [ -f "$file" ]; then
          filename=$(basename "$file")
          if [ "$filename" != "PLAN.md" ]; then
            cp "$file" "$fg_dir/"
          fi
        fi
      done
      
      # Create focus group STATE.md
      cat > "$fg_dir/STATE.md" << EOF
# Focus Group: $phase_name

**Status:** Active
**Migrated From:** phases/$phase_name
**Migration Date:** $(date -I)

## Concepts

*Concepts will be listed here as they are created.*

## Notes

This focus group was created from GSD phase: $phase_name
EOF
      
      echo "   ✅ $phase_name → focus-groups/$phase_name"
    fi
  done
  
  echo "✅ Phases transformed to focus groups"
fi
```

### Step 7: Analyze REQUIREMENTS.md for Focus Group Suggestions

```bash
if [ -f ".planning/REQUIREMENTS.md" ]; then
  echo "🔍 Analyzing REQUIREMENTS.md for focus group suggestions..."
  
  # Look for common domain keywords
  DOMAINS=""
  
  if grep -qi "security\|auth\|permission\|access" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS security"
  fi
  if grep -qi "onboard\|setup\|wizard\|getting.started" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS onboarding"
  fi
  if grep -qi "billing\|payment\|subscription\|pricing" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS billing"
  fi
  if grep -qi "api\|integrat\|webhook\|endpoint" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS api"
  fi
  if grep -qi "ui\|frontend\|component\|design" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS frontend"
  fi
  if grep -qi "infra\|devops\|deploy\|ci.cd\|pipeline" .planning/REQUIREMENTS.md; then
    DOMAINS="$DOMAINS infrastructure"
  fi
  
  if [ -n "$DOMAINS" ]; then
    echo "💡 Suggested focus groups based on REQUIREMENTS.md:"
    for domain in $DOMAINS; do
      echo "   - $domain"
    done
    echo ""
    echo "   Create with: wgsd create-focus-group <name>"
  else
    echo "ℹ️  No obvious domain groupings detected"
  fi
fi
```

### Step 8: Generate WGSD-CONFIG.md

```bash
if [ ! -f ".planning/WGSD-CONFIG.md" ]; then
  echo "📝 Creating WGSD-CONFIG.md..."
  
  # Try to detect stub from existing config or suggest
  PROJECT_NAME=$(basename "$(pwd)")
  SUGGESTED_STUB=$(echo "$PROJECT_NAME" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)
  
  cat > .planning/WGSD-CONFIG.md << EOF
# WGSD Configuration

**Project:** $PROJECT_NAME
**Migrated From:** GSD
**Migration Date:** $(date -I)

---

## Slack Configuration

**Stub:** $SUGGESTED_STUB
**Channel Prefix:** #${SUGGESTED_STUB}-

### Channel Patterns
| Type | Pattern | Example |
|------|---------|---------|
| Main Dev | \`${SUGGESTED_STUB}-dev\` | #${SUGGESTED_STUB}-dev |
| Focus Group | \`${SUGGESTED_STUB}-fg-{name}\` | #${SUGGESTED_STUB}-fg-security |
| Concept | \`${SUGGESTED_STUB}-cpt-{name}\` | #${SUGGESTED_STUB}-cpt-byof |
| Implementation | \`${SUGGESTED_STUB}-impl-{name}\` | #${SUGGESTED_STUB}-impl-auth-v2 |

> **Note:** Update the stub value if needed. Run \`wgsd init\` to change.

---

## Git Configuration

**Primary Branch:** (auto-detected)
**Focus Groups Base:** focus-groups/
**Implementations Base:** implementations/

---

## Migration Notes

- Original GSD structure backed up to: ${BACKUP_DIR:-"(no backup needed)"}
- Requirements analyzed for focus group suggestions
- Phase structure converted to focus groups (if present)

---

*Configuration generated during GSD → WGSD migration*
EOF
  
  echo "✅ WGSD-CONFIG.md created"
fi
```

### Step 9: Update STATE.md

```bash
if [ -f ".planning/STATE.md" ]; then
  echo "📝 Adding WGSD section to STATE.md..."
  
  if ! grep -q "## WGSD Status" .planning/STATE.md; then
    cat >> .planning/STATE.md << 'WGSD_STATE'

---

## WGSD Status

### Migration
- **Status:** ✅ Complete
- **Date:** Migrated on $(date -I)
- **Mode:** GSD → WGSD

### Focus Groups
*Focus groups will be listed as they become active.*

### Implementations  
*Active implementations listed here (max 2-4).*

---

*WGSD status section added during migration*
WGSD_STATE
    
    echo "✅ STATE.md updated"
  fi
fi
```

### Step 10: Validate Migration

```bash
echo ""
echo "🔍 Validating migration..."

ERRORS=""

# Check required files/directories
[ ! -d ".planning/focus-groups" ] && ERRORS="$ERRORS\n❌ Missing focus-groups/"
[ ! -d ".planning/active-implementations" ] && ERRORS="$ERRORS\n❌ Missing active-implementations/"
[ ! -f ".planning/WGSD-CONFIG.md" ] && ERRORS="$ERRORS\n❌ Missing WGSD-CONFIG.md"

if [ -n "$ERRORS" ]; then
  echo "Migration validation failed:"
  echo -e "$ERRORS"
  exit 1
fi

echo "✅ Migration validated successfully"
```

### Step 11: Commit Migration

```bash
echo ""
echo "📦 Committing migration..."

git add .planning/
git commit -m "feat: migrate planning structure from GSD to WGSD

- Added WGSD directory structure (focus-groups/, active-implementations/)
- Created WGSD-CONFIG.md with project configuration
- Enhanced PROJECT.md with WGSD workflow section
- Transformed phases to focus groups (if present)
- Created MASTER-ROADMAP.md from existing roadmap
- Updated STATE.md with WGSD status section

Migration backup: ${BACKUP_DIR:-"(no backup needed)"}"

if [ $? -eq 0 ]; then
  echo "✅ Migration committed"
else
  echo "⚠️  Commit failed (may already be committed or nothing to commit)"
fi
```

### Step 12: Report Success

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🎉 GSD → WGSD Migration Complete"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Structure:"
echo "   .planning/"
echo "   ├── focus-groups/     ✅"
echo "   ├── active-implementations/  ✅"
echo "   ├── WGSD-CONFIG.md    ✅"
echo "   └── MASTER-ROADMAP.md ✅"
echo ""
if [ -n "${BACKUP_DIR:-}" ]; then
  echo "📦 Backup: $BACKUP_DIR"
  echo ""
fi
echo "Next Steps:"
echo "  1. Review WGSD-CONFIG.md and update stub if needed"
echo "  2. Create Slack channel: #<stub>-dev"
echo "  3. Create first focus group: wgsd create-focus-group <name>"
echo ""
```

---

## Rollback

If migration fails and you need to restore:

```bash
# Find backup directory
BACKUP=$(ls -d .planning-backup-* | tail -1)

# Restore
rm -rf .planning
mv "$BACKUP" .planning

# Reset git
git checkout -- .planning/

echo "Rollback complete"
```

---

## Success Criteria

- [ ] GSD structure detected and analyzed
- [ ] Backup created before modifications
- [ ] WGSD directory structure created
- [ ] WGSD-CONFIG.md generated
- [ ] Content transformed and preserved
- [ ] Migration committed to git

---

*Workflow created for WGSD Phase 1*
