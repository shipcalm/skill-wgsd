---
name: wgsd:merge-concepts
description: Merge multiple related concepts into a unified concept (WGSD v2.3)
triggers:
  - "/wgsd merge-concepts"
  - "merge concepts"
dependencies:
  - wgsd:lib:concept-merge-engine
  - wgsd:lib:uuid-system
  - wgsd:lib:validation-suite
---

# Merge Concepts Workflow - WGSD v2.3

Intelligently merge multiple related concepts into a unified concept with consolidated impact matrices and preserved approval history.

---

## Overview

**Purpose:** Combine duplicate or related concepts that have emerged across teams into a single authoritative concept

**Use Cases:**
- Merge "oauth-v1" + "oauth-v2" → "oauth-integration"
- Combine duplicate concepts discovered across teams
- Consolidate related security concepts into unified approach

**Key Features:**
- Intelligent content similarity detection
- Impact matrix consolidation
- Approval status preservation
- Git history maintenance
- Rollback capability

---

## Command Usage

### Basic Merge
```bash
/wgsd merge-concepts source-concept target-concept
```

### Advanced Merge with Options
```bash
/wgsd merge-concepts oauth-v1-a1b2 oauth-v2-c3d4 \
  --target oauth-integration \
  --strategy content-based \
  --preserve-approvals \
  --dry-run
```

### Interactive Merge
```bash
/wgsd merge-concepts --interactive
# System guides you through concept selection and merge strategy
```

---

## Workflow Steps

### Step 1: Concept Discovery and Validation

```bash
echo "🔍 WGSD v2.3 Concept Merge Workflow"
echo "==================================="

# Parse concepts
source_concept="${1:-}"
target_concept="${2:-}"

if [ -z "$source_concept" ] || [ -z "$target_concept" ]; then
  echo "Usage: /wgsd merge-concepts <source-concept> <target-concept>"
  echo ""
  echo "Available concepts:"
  ls -1 .planning/concepts/ | grep -E "^[^.].*" | head -10
  exit 1
fi

# Validate concepts exist
source_path=".planning/concepts/$source_concept"
target_path=".planning/concepts/$target_concept"

if [ ! -d "$source_path" ]; then
  echo "❌ Source concept not found: $source_concept"
  exit 1
fi

if [ ! -d "$target_path" ]; then
  echo "❌ Target concept not found: $target_concept"
  exit 1
fi

echo "✅ Found concepts:"
echo "   Source: $source_concept"
echo "   Target: $target_concept"
```

### Step 2: Similarity Analysis

```bash
echo ""
echo "📊 Analyzing Concept Similarity"
echo "==============================="

# Content similarity analysis
similarity_score=$(concept_similarity_analyze "$source_path/CONCEPT.md" "$target_path/CONCEPT.md")
echo "Content Similarity: $similarity_score%"

# Impact overlap analysis  
impact_overlap=$(impact_matrix_overlap "$source_path/impact-matrix.md" "$target_path/impact-matrix.md")
echo "Impact Overlap: $impact_overlap%"

# Approval status check
source_approvals=$(get_concept_approvals "$source_concept")
target_approvals=$(get_concept_approvals "$target_concept")
echo "Source Approvals: $source_approvals"
echo "Target Approvals: $target_approvals"

# Merge recommendation
if [ $similarity_score -gt 70 ]; then
  echo "✅ HIGH similarity - Merge recommended"
elif [ $similarity_score -gt 40 ]; then
  echo "⚠️  MEDIUM similarity - Merge with caution"
else
  echo "❌ LOW similarity - Merge not recommended"
  echo "Consider if these concepts should remain separate"
fi
```

### Step 3: Merge Strategy Selection

```bash
echo ""
echo "🛠️  Merge Strategy Selection"
echo "==========================="

echo "Available merge strategies:"
echo "  [1] Content-based merge (intelligent content combination)"
echo "  [2] Target-priority merge (keep target, merge metadata)"
echo "  [3] Source-priority merge (keep source, merge metadata)"
echo "  [4] Manual merge (guided interactive merge)"

read -p "Select strategy (1-4): " strategy_choice

case "$strategy_choice" in
  1) merge_strategy="content-based" ;;
  2) merge_strategy="target-priority" ;;
  3) merge_strategy="source-priority" ;;
  4) merge_strategy="manual" ;;
  *) merge_strategy="content-based"; echo "Defaulting to content-based merge" ;;
esac

echo "Selected strategy: $merge_strategy"
```

### Step 4: Pre-Merge Validation

```bash
echo ""
echo "🔍 Pre-Merge Validation"
echo "======================="

# Check for blocking conditions
blocking_issues=""

# Check if concepts are in active implementations
if concept_in_active_implementation "$source_concept"; then
  blocking_issues="$blocking_issues\n- Source concept in active implementation"
fi

if concept_in_active_implementation "$target_concept"; then
  blocking_issues="$blocking_issues\n- Target concept in active implementation"
fi

# Check for conflicting approvals
if has_conflicting_approvals "$source_concept" "$target_concept"; then
  blocking_issues="$blocking_issues\n- Conflicting approval statuses"
fi

if [ -n "$blocking_issues" ]; then
  echo "❌ Blocking issues detected:"
  echo -e "$blocking_issues"
  echo ""
  echo "Resolve these issues before merging:"
  echo "1. Complete active implementations"
  echo "2. Resolve approval conflicts"
  echo "3. Notify impacted focus groups"
  exit 1
fi

echo "✅ Pre-merge validation passed"
```

### Step 5: Backup and Safety

```bash
echo ""
echo "💾 Creating Safety Backup"
echo "========================="

backup_timestamp=$(date +%Y%m%d_%H%M%S)
backup_dir=".planning/.backups/merge_$backup_timestamp"

mkdir -p "$backup_dir"

# Backup both concepts
cp -r "$source_path" "$backup_dir/source_concept/"
cp -r "$target_path" "$backup_dir/target_concept/"

# Backup concept registry
cp .planning/concepts/.concept-ids.json "$backup_dir/concept_ids_backup.json"

echo "✅ Backup created: $backup_dir"
```

### Step 6: Execute Merge

```bash
echo ""
echo "🔄 Executing Concept Merge"
echo "=========================="

# Create merged concept using selected strategy
case "$merge_strategy" in
  "content-based")
    merged_content=$(merge_concepts_content_based "$source_path" "$target_path")
    ;;
  "target-priority")
    merged_content=$(merge_concepts_target_priority "$source_path" "$target_path")
    ;;
  "source-priority") 
    merged_content=$(merge_concepts_source_priority "$source_path" "$target_path")
    ;;
  "manual")
    merged_content=$(merge_concepts_interactive "$source_path" "$target_path")
    ;;
esac

# Apply merged content to target concept
echo "$merged_content" > "$target_path/CONCEPT.md"

# Merge impact matrices
merge_impact_matrices "$source_path/impact-matrix.md" "$target_path/impact-matrix.md"

# Update concept metadata
update_concept_metadata "$target_concept" "merged_from" "$source_concept"

echo "✅ Concept merge completed"
```

### Step 7: Cleanup and Notification

```bash
echo ""
echo "🧹 Cleanup and Notification"
echo "==========================="

# Archive source concept (don't delete, preserve for audit)
archive_dir=".planning/.archived/concepts"
mkdir -p "$archive_dir"
mv "$source_path" "$archive_dir/${source_concept}_archived_${backup_timestamp}"

# Update concept registry
remove_concept_from_registry "$source_concept"
update_concept_registry "$target_concept" "merged_from" "$source_concept"

# Update focus group references
update_focus_group_references "$source_concept" "$target_concept"

# Send notifications to impacted focus groups
notify_concept_merge "$source_concept" "$target_concept"

echo "✅ Cleanup completed"
echo "✅ Notifications sent to impacted focus groups"
```

### Step 8: Validation and Summary

```bash
echo ""
echo "🔍 Post-Merge Validation"
echo "========================"

# Validate merged concept
if validate_concept_integrity "$target_concept"; then
  echo "✅ Merged concept validation passed"
else
  echo "❌ Merged concept validation failed"
  echo "Initiating rollback..."
  restore_from_backup "$backup_dir"
  exit 1
fi

# Validation summary
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Merge Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Merged: $source_concept → $target_concept"
echo "📁 Backup: $backup_dir"
echo "📊 Similarity: $similarity_score%"
echo "🔄 Strategy: $merge_strategy"
echo "📤 Notifications sent to impacted focus groups"
echo ""
echo "Next steps:"
echo "1. Review merged concept: $target_concept"
echo "2. Update implementation plans if needed"
echo "3. Archive or delete backup after verification"
echo ""
echo "Rollback command (if needed):"
echo "wgsd restore-backup $backup_dir"
```

---

## Merge Strategies

### Content-Based Merge
- **Algorithm:** Intelligent content combination using similarity analysis
- **Sections:** Merges overlapping sections, preserves unique content
- **Impact Matrix:** Consolidates all impacts with priority resolution
- **Use Case:** Related concepts with overlapping functionality

### Target-Priority Merge  
- **Algorithm:** Keeps target concept content, merges source metadata
- **Sections:** Target concept content preserved, source impacts added
- **Impact Matrix:** Source impacts merged into target matrix
- **Use Case:** Authoritative target concept with supplementary source

### Source-Priority Merge
- **Algorithm:** Keeps source concept content, merges target metadata  
- **Sections:** Source concept content preserved, target impacts added
- **Impact Matrix:** Target impacts merged into source matrix
- **Use Case:** Comprehensive source concept with legacy target

### Manual Merge
- **Algorithm:** Interactive guided merge with user decisions
- **Sections:** User chooses content for each section
- **Impact Matrix:** User resolves impact conflicts interactively
- **Use Case:** Complex merges requiring human judgment

---

## Rollback Procedure

If merge fails or issues are discovered:

```bash
# Restore from backup
wgsd restore-backup .planning/.backups/merge_20260223_143022

# Or manual rollback
wgsd rollback-merge source-concept target-concept
```

---

## Integration Points

### Impact on Focus Groups
- Focus groups receive notifications about concept merges
- Impact matrices are consolidated across all impacted focus groups
- Approval statuses are preserved and inherited appropriately

### Git Integration
- Merge preserves git history from both concepts
- Branch merges handled automatically where applicable
- Commit history attribution maintained

### Canvas Integration
- Canvas references updated to point to merged concept
- Historical canvas data preserved in archived concept

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Concept not found | Invalid concept UUID | List available concepts, check spelling |
| Active implementation | Concept in current implementation | Complete implementation first |
| Conflicting approvals | Approval status conflicts | Resolve approval conflicts with focus groups |
| Similarity too low | Unrelated concepts | Consider keeping concepts separate |
| Validation failure | Merged concept invalid | Automatic rollback initiated |

---

## Examples

### Example 1: OAuth Concepts
```bash
# Merge OAuth v1 and v2 concepts
/wgsd merge-concepts oauth-v1-a1b2 oauth-v2-c3d4 --target oauth-integration

# Result: Single OAuth integration concept with combined impacts
```

### Example 2: Security Concepts  
```bash
# Merge related security concepts
/wgsd merge-concepts auth-refactor-e5f6 sso-integration-g7h8 --strategy content-based

# Result: Unified authentication concept with consolidated security impacts
```

---

*WGSD v2.3 Concept Merge Operations - Intelligent concept consolidation for better collaboration*