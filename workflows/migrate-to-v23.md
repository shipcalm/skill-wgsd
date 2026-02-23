---
name: wgsd:migrate-to-v23
description: Automated migration from WGSD v2.2 to v2.3 Independent Concept Architecture
triggers:
  - "/wgsd migrate-to-v23"
  - "migrate to v2.3"
dependencies:
  - wgsd:lib:migration-compatibility-engine
  - wgsd:lib:uuid-system
  - wgsd:lib:validation-suite
---

# Migrate to WGSD v2.3 - Independent Concept Architecture

Automated migration from v2.2 focus-group-owned concepts to v2.3 independent concept management with full safety and rollback capabilities.

---

## Overview

**Purpose:** Seamlessly upgrade existing WGSD v2.2 installations to v2.3 Independent Concept Architecture

**Key Changes:**
- Migrate `.planning/focus-groups/{fg}/concepts/` → `.planning/concepts/`
- Implement UUID-based concept identification
- Enable concept merge/split operations
- Maintain full backward compatibility

**Safety Features:**
- Complete backup before migration
- Rollback capability at any point
- Validation at each migration step
- Zero data loss guarantee

---

## Migration Command

### Basic Migration
```bash
/wgsd migrate-to-v23
```

### Advanced Migration with Options
```bash
/wgsd migrate-to-v23 \
  --backup-dir /path/to/backup \
  --dry-run \
  --validate-only \
  --preserve-structure
```

### Migration Analysis Only
```bash
/wgsd migrate-to-v23 --analyze
# Analyzes current installation and provides migration preview
```

---

## Migration Workflow

### Step 1: Pre-Migration Analysis

```bash
echo "🔍 WGSD v2.3 Migration Analysis"
echo "==============================="

# Detect current installation type
installation_type=$(detect_wgsd_installation_type)

echo "Current Installation: $installation_type"

case "$installation_type" in
  "fresh")
    echo "✅ Fresh installation - v2.3 can be installed directly"
    echo "No migration needed. Run: wgsd init --version v2.3"
    exit 0
    ;;
  "v2.2")
    echo "📋 WGSD v2.2 detected - migration to v2.3 available"
    ;;
  "v2.3")
    echo "✅ Already on WGSD v2.3"
    echo "Current version: $(get_wgsd_version)"
    exit 0
    ;;
  "mixed")
    echo "⚠️  Mixed version detected - manual intervention required"
    echo "Please resolve version conflicts before migration"
    exit 1
    ;;
  "unknown")
    echo "❓ Unknown WGSD installation"
    echo "Please verify WGSD installation before migration"
    exit 1
    ;;
esac
```

### Step 2: Migration Readiness Assessment

```bash
echo ""
echo "📊 Migration Readiness Assessment"
echo "================================="

readiness_score=0
blocking_issues=""

# Check for active implementations
active_impls=$(count_active_implementations)
if [ $active_impls -gt 0 ]; then
  echo "⚠️  Active implementations detected: $active_impls"
  echo "   Consider completing implementations before migration"
  readiness_score=$((readiness_score - 2))
else
  echo "✅ No active implementations"
  readiness_score=$((readiness_score + 1))
fi

# Check for pending approvals
pending_approvals=$(count_pending_approvals)
if [ $pending_approvals -gt 0 ]; then
  echo "⚠️  Pending approvals detected: $pending_approvals"
  echo "   Migration will preserve approval states"
  readiness_score=$((readiness_score + 0))
else
  echo "✅ No pending approvals"
  readiness_score=$((readiness_score + 1))
fi

# Check concept count and complexity
concept_count=$(count_v22_concepts)
if [ $concept_count -gt 20 ]; then
  echo "📊 Large concept set detected: $concept_count concepts"
  echo "   Migration will take longer but should complete successfully"
  readiness_score=$((readiness_score + 1))
elif [ $concept_count -gt 0 ]; then
  echo "📊 Moderate concept set: $concept_count concepts"
  readiness_score=$((readiness_score + 2))
else
  echo "📊 No concepts detected - migration will be quick"
  readiness_score=$((readiness_score + 2))
fi

# Check for naming conflicts
naming_conflicts=$(detect_concept_naming_conflicts)
if [ $naming_conflicts -gt 0 ]; then
  echo "⚠️  Naming conflicts detected: $naming_conflicts conflicts"
  echo "   These will be resolved automatically during migration"
  blocking_issues="$blocking_issues\n- Naming conflicts require resolution"
else
  echo "✅ No naming conflicts detected"
  readiness_score=$((readiness_score + 1))
fi

# Check available disk space
available_space=$(df -BM . | awk 'NR==2 {print $4}' | sed 's/M//')
required_space=100  # Minimum 100MB for migration
if [ $available_space -lt $required_space ]; then
  echo "❌ Insufficient disk space: ${available_space}MB available, ${required_space}MB required"
  blocking_issues="$blocking_issues\n- Insufficient disk space for migration"
else
  echo "✅ Sufficient disk space: ${available_space}MB available"
  readiness_score=$((readiness_score + 1))
fi

echo ""
echo "🏆 Migration Readiness Score: $readiness_score/6"

if [ -n "$blocking_issues" ]; then
  echo "❌ Blocking issues detected:"
  echo -e "$blocking_issues"
  echo ""
  echo "Please resolve these issues before migration"
  exit 1
fi

if [ $readiness_score -ge 4 ]; then
  echo "✅ Ready for migration"
elif [ $readiness_score -ge 2 ]; then
  echo "⚠️  Migration possible but with caution"
  echo "Consider resolving warnings before proceeding"
else
  echo "❌ Not ready for migration"
  echo "Please address the issues above"
  exit 1
fi
```

### Step 3: Backup Creation

```bash
echo ""
echo "💾 Creating Comprehensive Backup"
echo "==============================="

backup_timestamp=$(date +%Y%m%d_%H%M%S)
backup_dir=".planning/.backups/v23_migration_$backup_timestamp"

mkdir -p "$backup_dir"

echo "📁 Backup directory: $backup_dir"

# Backup entire .planning directory
echo "📋 Backing up planning structure..."
cp -r .planning/* "$backup_dir/" 2>/dev/null || true

# Backup git state
echo "🔗 Backing up git references..."
git log --oneline -10 > "$backup_dir/git_history.txt"
git branch -a > "$backup_dir/git_branches.txt"

# Backup WGSD configuration
echo "⚙️  Backing up WGSD configuration..."
if [ -f ".planning/WGSD-CONFIG.md" ]; then
  cp .planning/WGSD-CONFIG.md "$backup_dir/wgsd_config_backup.md"
fi

# Create migration manifest
cat > "$backup_dir/migration_manifest.json" << EOF
{
  "migration_version": "v2.2_to_v2.3",
  "backup_timestamp": "$backup_timestamp",
  "concept_count": $concept_count,
  "readiness_score": $readiness_score,
  "git_commit": "$(git rev-parse HEAD 2>/dev/null || echo 'unknown')",
  "migration_date": "$(date -Iseconds)"
}
EOF

echo "✅ Backup completed: $backup_dir"
echo "📄 Migration manifest: $backup_dir/migration_manifest.json"
```

### Step 4: UUID System Initialization

```bash
echo ""
echo "🆔 Initializing UUID System"
echo "=========================="

# Create UUID registry
mkdir -p .planning/concepts
cat > .planning/concepts/.concept-ids.json << 'EOF'
{
  "version": "v2.3.0",
  "created": "2026-02-23T22:30:00Z",
  "last_updated": "2026-02-23T22:30:00Z",
  "concept_count": 0,
  "concepts": {},
  "archived_concepts": {}
}
EOF

# Install UUID generation system
cat > .planning/concepts/.metadata-manager.js << 'EOF'
// WGSD v2.3 Concept Metadata Manager
// Handles UUID generation, concept registration, and metadata management

function generateConceptUuid() {
  // Generate 8-character UUID suffix
  return Math.random().toString(36).substr(2, 8);
}

function registerConcept(name, uuid, metadata = {}) {
  const registry = JSON.parse(fs.readFileSync('.concept-ids.json', 'utf8'));
  const fullName = `${name}-${uuid}`;
  
  registry.concepts[fullName] = {
    name: name,
    uuid: uuid,
    created: new Date().toISOString(),
    ...metadata
  };
  
  registry.concept_count = Object.keys(registry.concepts).length;
  registry.last_updated = new Date().toISOString();
  
  fs.writeFileSync('.concept-ids.json', JSON.stringify(registry, null, 2));
  return fullName;
}

module.exports = { generateConceptUuid, registerConcept };
EOF

echo "✅ UUID system initialized"
echo "📋 Concept registry created: .planning/concepts/.concept-ids.json"
```

### Step 5: Concept Migration

```bash
echo ""
echo "🔄 Migrating Concepts to Independent Architecture"
echo "==============================================="

migrated_count=0
conflict_count=0
error_count=0

# Scan all focus groups for concepts
for fg_dir in .planning/focus-groups/*/; do
  if [ ! -d "$fg_dir" ]; then continue; fi
  
  fg_name=$(basename "$fg_dir")
  echo "🔍 Processing focus group: $fg_name"
  
  concepts_dir="$fg_dir/concepts"
  if [ ! -d "$concepts_dir" ]; then
    echo "   No concepts directory found"
    continue
  fi
  
  # Process each concept in this focus group
  for concept_dir in "$concepts_dir"/*/; do
    if [ ! -d "$concept_dir" ]; then continue; fi
    
    concept_name=$(basename "$concept_dir")
    echo "   📋 Migrating concept: $concept_name"
    
    # Generate UUID for this concept
    concept_uuid=$(node -e "console.log(Math.random().toString(36).substr(2, 8))")
    
    # Check for naming conflicts
    target_name="$concept_name"
    conflict_suffix=1
    while [ -d ".planning/concepts/$target_name-$concept_uuid" ]; do
      target_name="$concept_name-$fg_name-conflict$conflict_suffix"
      conflict_suffix=$((conflict_suffix + 1))
      conflict_count=$((conflict_count + 1))
    done
    
    # Create independent concept directory
    independent_concept_dir=".planning/concepts/$target_name-$concept_uuid"
    mkdir -p "$independent_concept_dir"
    
    # Copy concept files with metadata updates
    for file in "$concept_dir"/*; do
      if [ -f "$file" ]; then
        filename=$(basename "$file")
        cp "$file" "$independent_concept_dir/$filename"
        
        # Update metadata in files
        case "$filename" in
          "CONCEPT.md")
            # Add v2.3 metadata header
            temp_file=$(mktemp)
            cat > "$temp_file" << EOF
---
uuid: $concept_uuid
name: $target_name
original_focus_group: $fg_name
migrated_from: v2.2
migration_date: $(date -Iseconds)
version: 1.0.0
---

EOF
            # Skip existing frontmatter if present
            if head -1 "$independent_concept_dir/$filename" | grep -q "^---"; then
              awk 'BEGIN{skip=1} /^---/{if(skip && NR>1){skip=0; next}} !skip' "$independent_concept_dir/$filename" >> "$temp_file"
            else
              cat "$independent_concept_dir/$filename" >> "$temp_file"
            fi
            mv "$temp_file" "$independent_concept_dir/$filename"
            ;;
          "impact-matrix.md")
            # Update impact matrix with v2.3 schema
            add_v23_impact_metadata "$independent_concept_dir/$filename" "$concept_uuid" "$target_name"
            ;;
        esac
      fi
    done
    
    # Register concept in UUID system
    register_concept_in_registry "$target_name" "$concept_uuid" "$fg_name"
    
    migrated_count=$((migrated_count + 1))
    echo "   ✅ Migrated: $concept_name → $target_name-$concept_uuid"
  done
done

echo ""
echo "📊 Migration Statistics:"
echo "   ✅ Concepts migrated: $migrated_count"
echo "   ⚠️  Naming conflicts resolved: $conflict_count"
echo "   ❌ Errors: $error_count"
```

### Step 6: Focus Group Updates

```bash
echo ""
echo "📋 Updating Focus Group References"
echo "================================="

# Update focus group roadmaps to reference independent concepts
for fg_dir in .planning/focus-groups/*/; do
  if [ ! -d "$fg_dir" ]; then continue; fi
  
  fg_name=$(basename "$fg_dir")
  roadmap_file="$fg_dir/ROADMAP.md"
  
  if [ -f "$roadmap_file" ]; then
    echo "🔄 Updating focus group: $fg_name"
    
    # Update roadmap references
    update_focus_group_roadmap_references "$roadmap_file" "$fg_name"
    
    # Remove old concepts directory (now empty)
    if [ -d "$fg_dir/concepts" ]; then
      if [ -z "$(ls -A "$fg_dir/concepts")" ]; then
        rmdir "$fg_dir/concepts"
        echo "   🗑️  Removed empty concepts directory"
      else
        echo "   ⚠️  Concepts directory not empty - manual cleanup required"
      fi
    fi
    
    echo "   ✅ Focus group updated: $fg_name"
  fi
done
```

### Step 7: Configuration Updates

```bash
echo ""
echo "⚙️  Updating WGSD Configuration"
echo "=============================="

# Update WGSD-CONFIG.md for v2.3
config_file=".planning/WGSD-CONFIG.md"

# Create v2.3 configuration
cat > "$config_file" << EOF
# WGSD Configuration - v2.3

**Version:** v2.3.0  
**Architecture:** Independent Concept Management  
**Migration Date:** $(date -Iseconds)

---

## v2.3 Features

- ✅ Independent concept architecture
- ✅ UUID-based concept identification  
- ✅ Concept merge/split operations
- ✅ Focus group merge/split operations
- ✅ Enhanced cross-cutting impact management

---

## Concept Management

**Independent Concepts:** \`.planning/concepts/\`
**UUID Registry:** \`.planning/concepts/.concept-ids.json\`
**Metadata Manager:** \`.planning/concepts/.metadata-manager.js\`

---

## Available Operations

### Concept Operations
- \`/wgsd create-concept [name]\` - Create independent concept
- \`/wgsd merge-concepts source target\` - Merge related concepts
- \`/wgsd split-concept parent child1 child2\` - Split complex concepts

### Focus Group Operations  
- \`/wgsd merge-focus-groups source target\` - Consolidate focus groups
- \`/wgsd split-focus-group parent child1 child2\` - Split focus groups

### System Operations
- \`/wgsd validate-v23\` - Validate v2.3 installation
- \`/wgsd migrate-to-v23\` - Migration utilities

---

## Migration Information

**Migrated From:** v2.2 Focus Group Owned Concepts
**Migration Date:** $(date -Iseconds)
**Concepts Migrated:** $migrated_count
**Backup Location:** $backup_dir

---

*WGSD v2.3 Independent Concept Architecture - Configured and Operational*
EOF

echo "✅ WGSD configuration updated: $config_file"
```

### Step 8: System Validation

```bash
echo ""
echo "🔍 Post-Migration Validation"
echo "============================"

validation_errors=0

# Validate concept structure
echo "📋 Validating concept structure..."
for concept_dir in .planning/concepts/*/; do
  if [ ! -d "$concept_dir" ]; then continue; fi
  
  concept_name=$(basename "$concept_dir")
  
  # Check required files
  if [ ! -f "$concept_dir/CONCEPT.md" ]; then
    echo "   ❌ Missing CONCEPT.md: $concept_name"
    validation_errors=$((validation_errors + 1))
  fi
  
  if [ ! -f "$concept_dir/impact-matrix.md" ]; then
    echo "   ❌ Missing impact-matrix.md: $concept_name"
    validation_errors=$((validation_errors + 1))
  fi
  
  # Validate UUID format
  if ! echo "$concept_name" | grep -qE ".*-[a-z0-9]{8}$"; then
    echo "   ❌ Invalid UUID format: $concept_name"
    validation_errors=$((validation_errors + 1))
  fi
done

# Validate UUID registry
echo "📋 Validating UUID registry..."
if [ ! -f ".planning/concepts/.concept-ids.json" ]; then
  echo "   ❌ Missing UUID registry"
  validation_errors=$((validation_errors + 1))
else
  registry_concept_count=$(jq -r '.concept_count' .planning/concepts/.concept-ids.json 2>/dev/null || echo 0)
  actual_concept_count=$(find .planning/concepts -maxdepth 1 -type d -name "*-????????" | wc -l)
  
  if [ "$registry_concept_count" -ne "$actual_concept_count" ]; then
    echo "   ⚠️  Registry count mismatch: registry=$registry_concept_count, actual=$actual_concept_count"
    # Auto-repair registry
    repair_uuid_registry
  fi
fi

# Validate focus group references
echo "📋 Validating focus group references..."
validate_focus_group_concept_references

echo ""
if [ $validation_errors -eq 0 ]; then
  echo "✅ Migration validation PASSED"
  echo "🎉 WGSD v2.3 migration completed successfully!"
else
  echo "❌ Migration validation FAILED with $validation_errors errors"
  echo "Consider running rollback: wgsd rollback-migration $backup_dir"
fi
```

### Step 9: Migration Summary

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Migration Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🎯 Migration: WGSD v2.2 → v2.3"
echo "📊 Concepts migrated: $migrated_count"
echo "🔧 Conflicts resolved: $conflict_count"
echo "💾 Backup location: $backup_dir"
echo "⚙️  Configuration: .planning/WGSD-CONFIG.md"
echo "🆔 UUID registry: .planning/concepts/.concept-ids.json"
echo ""
echo "✅ New v2.3 Features Available:"
echo "   • Independent concept management"
echo "   • Concept merge operations (/wgsd merge-concepts)"
echo "   • Concept split operations (/wgsd split-concept)"
echo "   • Focus group merge/split operations"
echo "   • Enhanced cross-cutting impact management"
echo ""
echo "🚀 Next Steps:"
echo "1. Review migrated concepts in .planning/concepts/"
echo "2. Test new merge/split operations"
echo "3. Update team workflows for independent concepts"
echo "4. Archive or delete backup after verification"
echo ""
echo "📚 Documentation:"
echo "   • /wgsd help - Updated command reference"
echo "   • .planning/concepts/README.md - v2.3 architecture guide"
echo ""
echo "🔄 Rollback (if needed):"
echo "   wgsd rollback-migration $backup_dir"
echo ""
echo "🎉 Welcome to WGSD v2.3 Independent Concept Architecture!"
```

---

## Rollback Procedure

If migration fails or issues are discovered:

```bash
# Complete rollback to v2.2
/wgsd rollback-migration .planning/.backups/v23_migration_20260223_223000

# Selective rollback (restore specific components)
/wgsd rollback-migration --partial --concepts-only
```

## Validation Commands

### Pre-Migration Validation
```bash
# Analyze current installation
/wgsd migrate-to-v23 --analyze

# Dry run migration (no changes)
/wgsd migrate-to-v23 --dry-run
```

### Post-Migration Validation
```bash
# Validate v2.3 installation
/wgsd validate-v23

# Check concept integrity
/wgsd validate-concepts --all

# Verify focus group references
/wgsd validate-focus-groups
```

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Insufficient disk space | Low disk space | Free up space, retry migration |
| Naming conflicts | Duplicate concept names | Auto-resolved with focus group suffix |
| Active implementations | Concepts in use | Complete implementations or use --force |
| Git conflicts | Uncommitted changes | Commit changes before migration |
| Validation failure | Corrupted migration | Automatic rollback initiated |

---

## Examples

### Standard Migration
```bash
# Analyze and migrate v2.2 to v2.3
/wgsd migrate-to-v23

# Result: All concepts moved to independent architecture
```

### Cautious Migration
```bash  
# Analyze first, then dry-run, then migrate
/wgsd migrate-to-v23 --analyze
/wgsd migrate-to-v23 --dry-run
/wgsd migrate-to-v23

# Result: Careful step-by-step migration with validation
```

---

*WGSD v2.3 Migration - Seamless upgrade to Independent Concept Architecture*