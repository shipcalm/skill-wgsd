---
name: wgsd:split-concept
description: Split complex concepts into focused sub-concepts (WGSD v2.3)
triggers:
  - "/wgsd split-concept"
  - "split concept"
dependencies:
  - wgsd:lib:concept-split-engine
  - wgsd:lib:uuid-system
  - wgsd:lib:validation-suite
---

# Split Concept Workflow - WGSD v2.3

Intelligently split complex concepts into focused sub-concepts with distributed impact matrices and relationship tracking.

---

## Overview

**Purpose:** Break complex concepts into smaller, more focused concepts for parallel development and clearer ownership

**Use Cases:**
- Split "user-management" → "user-auth" + "user-profile" + "user-permissions"
- Break complex concepts for parallel development
- Separate concerns for better approval matrix management

**Key Features:**
- Complexity analysis and split recommendations
- Intelligent content distribution
- Impact inheritance across sub-concepts
- Parent-child relationship tracking
- Approval status distribution

---

## Command Usage

### Basic Split
```bash
/wgsd split-concept parent-concept child1-concept child2-concept
```

### Advanced Split with Strategy
```bash
/wgsd split-concept user-management-a1b2 \
  --children user-auth user-profile user-permissions \
  --strategy section-based \
  --preserve-relationships \
  --dry-run
```

### Interactive Split Analysis
```bash
/wgsd split-concept user-management-a1b2 --analyze
# System analyzes complexity and suggests optimal split strategy
```

---

## Workflow Steps

### Step 1: Concept Analysis and Validation

```bash
echo "🔍 WGSD v2.3 Concept Split Workflow"
echo "===================================="

parent_concept="${1:-}"
shift
child_concepts=("$@")

if [ -z "$parent_concept" ] || [ ${#child_concepts[@]} -eq 0 ]; then
  echo "Usage: /wgsd split-concept <parent-concept> <child1> [child2] [child3]..."
  echo ""
  echo "Available concepts:"
  ls -1 .planning/concepts/ | grep -E "^[^.].*" | head -10
  exit 1
fi

# Validate parent concept exists
parent_path=".planning/concepts/$parent_concept"
if [ ! -d "$parent_path" ]; then
  echo "❌ Parent concept not found: $parent_concept"
  exit 1
fi

echo "✅ Parent concept found: $parent_concept"
echo "📋 Child concepts to create: ${child_concepts[*]}"
```

### Step 2: Complexity Analysis

```bash
echo ""
echo "📊 Concept Complexity Analysis"
echo "=============================="

# Analyze concept complexity (10-point scale)
complexity_score=0

# Content volume analysis
content_lines=$(wc -l < "$parent_path/CONCEPT.md")
if [ $content_lines -gt 200 ]; then
  complexity_score=$((complexity_score + 3))
  echo "📝 Large content volume: $content_lines lines (+3)"
elif [ $content_lines -gt 100 ]; then
  complexity_score=$((complexity_score + 2))
  echo "📝 Medium content volume: $content_lines lines (+2)"
else
  complexity_score=$((complexity_score + 1))
  echo "📝 Small content volume: $content_lines lines (+1)"
fi

# Impact breadth analysis
impact_count=$(grep -c "focus-group:" "$parent_path/impact-matrix.md" 2>/dev/null || echo 0)
if [ $impact_count -gt 4 ]; then
  complexity_score=$((complexity_score + 3))
  echo "🎯 High impact breadth: $impact_count focus groups (+3)"
elif [ $impact_count -gt 2 ]; then
  complexity_score=$((complexity_score + 2))
  echo "🎯 Medium impact breadth: $impact_count focus groups (+2)"
else
  complexity_score=$((complexity_score + 1))
  echo "🎯 Low impact breadth: $impact_count focus groups (+1)"
fi

# Implementation complexity analysis
has_api_spec=$([ -f "$parent_path/API-SPEC.md" ] && echo "yes" || echo "no")
has_acceptance_criteria=$([ -f "$parent_path/acceptance-criteria.md" ] && echo "yes" || echo "no")

if [ "$has_api_spec" = "yes" ]; then
  complexity_score=$((complexity_score + 2))
  echo "🔌 API specification present (+2)"
fi

if [ "$has_acceptance_criteria" = "yes" ]; then
  complexity_score=$((complexity_score + 1))
  echo "✅ Acceptance criteria defined (+1)"
fi

# Approval complexity
approval_count=$(get_concept_approval_count "$parent_concept")
if [ $approval_count -gt 3 ]; then
  complexity_score=$((complexity_score + 1))
  echo "📋 Complex approval matrix: $approval_count approvals (+1)"
fi

echo ""
echo "🏆 Total Complexity Score: $complexity_score/10"

# Split recommendation
if [ $complexity_score -ge 7 ]; then
  echo "✅ HIGH complexity - Split STRONGLY recommended"
elif [ $complexity_score -ge 4 ]; then
  echo "⚠️  MEDIUM complexity - Split recommended"
else
  echo "ℹ️  LOW complexity - Split optional, consider keeping unified"
fi
```

### Step 3: Split Strategy Selection

```bash
echo ""
echo "🛠️  Split Strategy Selection"  
echo "==========================="

echo "Available split strategies:"
echo "  [1] Section-based split (divide by major sections)"
echo "  [2] Theme-based split (divide by functional themes)"
echo "  [3] Impact-based split (divide by focus group impacts)"
echo "  [4] Size-based split (divide by content volume)"
echo "  [5] Custom split (manual content assignment)"

read -p "Select strategy (1-5): " strategy_choice

case "$strategy_choice" in
  1) split_strategy="section-based" ;;
  2) split_strategy="theme-based" ;;
  3) split_strategy="impact-based" ;;
  4) split_strategy="size-based" ;;
  5) split_strategy="custom" ;;
  *) split_strategy="theme-based"; echo "Defaulting to theme-based split" ;;
esac

echo "Selected strategy: $split_strategy"

# Analyze split feasibility
if [ ${#child_concepts[@]} -gt 5 ]; then
  echo "⚠️  Warning: Splitting into ${#child_concepts[@]} concepts may create excessive fragmentation"
  read -p "Continue with split? (y/n): " continue_split
  if [ "$continue_split" != "y" ]; then
    echo "Split cancelled"
    exit 0
  fi
fi
```

### Step 4: Pre-Split Validation

```bash
echo ""
echo "🔍 Pre-Split Validation"
echo "======================="

# Check for blocking conditions
blocking_issues=""

# Check if parent concept is in active implementation
if concept_in_active_implementation "$parent_concept"; then
  blocking_issues="$blocking_issues\n- Parent concept in active implementation"
fi

# Check for pending approvals that would be disrupted
if has_pending_approvals "$parent_concept"; then
  blocking_issues="$blocking_issues\n- Pending approvals on parent concept"
fi

# Check for child concept name conflicts
for child in "${child_concepts[@]}"; do
  if concept_exists "$child"; then
    blocking_issues="$blocking_issues\n- Child concept name conflict: $child"
  fi
done

if [ -n "$blocking_issues" ]; then
  echo "❌ Blocking issues detected:"
  echo -e "$blocking_issues"
  echo ""
  echo "Resolve these issues before splitting:"
  echo "1. Complete active implementations"
  echo "2. Resolve pending approvals"
  echo "3. Choose unique child concept names"
  exit 1
fi

echo "✅ Pre-split validation passed"
```

### Step 5: Content Distribution Analysis

```bash
echo ""
echo "📄 Content Distribution Analysis"
echo "==============================="

# Analyze parent concept content for distribution
case "$split_strategy" in
  "section-based")
    analyze_sections_for_split "$parent_path/CONCEPT.md" "${child_concepts[@]}"
    ;;
  "theme-based")
    analyze_themes_for_split "$parent_path/CONCEPT.md" "${child_concepts[@]}"
    ;;
  "impact-based")
    analyze_impacts_for_split "$parent_path/impact-matrix.md" "${child_concepts[@]}"
    ;;
  "size-based")
    analyze_size_for_split "$parent_path/CONCEPT.md" "${child_concepts[@]}"
    ;;
  "custom")
    interactive_content_assignment "$parent_path" "${child_concepts[@]}"
    ;;
esac

echo "✅ Content distribution plan created"
```

### Step 6: Safety Backup

```bash
echo ""
echo "💾 Creating Safety Backup"
echo "========================="

backup_timestamp=$(date +%Y%m%d_%H%M%S)
backup_dir=".planning/.backups/split_$backup_timestamp"

mkdir -p "$backup_dir"

# Backup parent concept
cp -r "$parent_path" "$backup_dir/parent_concept/"

# Backup concept registry
cp .planning/concepts/.concept-ids.json "$backup_dir/concept_ids_backup.json"

# Backup focus group references
backup_focus_group_references "$parent_concept" "$backup_dir"

echo "✅ Backup created: $backup_dir"
```

### Step 7: Execute Split

```bash
echo ""
echo "🔄 Executing Concept Split"
echo "=========================="

# Generate UUIDs for child concepts
declare -A child_uuids
for child in "${child_concepts[@]}"; do
  child_uuid=$(generate_concept_uuid)
  child_uuids[$child]=$child_uuid
  echo "📋 Generated UUID for $child: $child_uuid"
done

# Create child concept directories
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_path=".planning/concepts/${child}-${child_uuid}"
  mkdir -p "$child_path"
  
  echo "📁 Created child concept: $child_path"
done

# Distribute content based on selected strategy
case "$split_strategy" in
  "section-based")
    distribute_content_by_sections "$parent_path" child_concepts child_uuids
    ;;
  "theme-based")
    distribute_content_by_themes "$parent_path" child_concepts child_uuids
    ;;
  "impact-based")
    distribute_content_by_impacts "$parent_path" child_concepts child_uuids
    ;;
  "size-based")
    distribute_content_by_size "$parent_path" child_concepts child_uuids
    ;;
  "custom")
    apply_custom_content_distribution "$parent_path" child_concepts child_uuids
    ;;
esac

echo "✅ Content distribution completed"
```

### Step 8: Impact Matrix Distribution

```bash
echo ""
echo "🎯 Distributing Impact Matrix"
echo "============================="

# Distribute parent impacts across child concepts
parent_impacts=$(parse_impact_matrix "$parent_path/impact-matrix.md")

for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_path=".planning/concepts/${child}-${child_uuid}"
  
  # Determine which impacts apply to this child concept
  relevant_impacts=$(determine_relevant_impacts "$child" "$parent_impacts" "$split_strategy")
  
  # Create child impact matrix
  create_child_impact_matrix "$child_path/impact-matrix.md" "$child" "$child_uuid" "$relevant_impacts"
  
  echo "🎯 Created impact matrix for: $child"
done

# Preserve parent-child relationships
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  add_parent_relationship "$parent_concept" "${child}-${child_uuid}"
done

echo "✅ Impact distribution completed"
```

### Step 9: Approval Status Distribution

```bash
echo ""
echo "📋 Distributing Approval Status"
echo "==============================="

# Get parent approval status
parent_approvals=$(get_concept_approvals "$parent_concept")

# Distribute approvals to child concepts based on impact relevance
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_full_name="${child}-${child_uuid}"
  
  # Determine which approvals are relevant to this child
  relevant_approvals=$(filter_approvals_by_child_impacts "$parent_approvals" "$child_full_name")
  
  # Set child concept approval status
  if [ -n "$relevant_approvals" ]; then
    set_concept_approvals "$child_full_name" "$relevant_approvals"
    echo "📋 Inherited approvals for: $child"
  else
    echo "📋 No inherited approvals for: $child (will need fresh approval)"
  fi
done

echo "✅ Approval distribution completed"
```

### Step 10: Archive Parent and Update References

```bash
echo ""
echo "🗃️  Archiving Parent and Updating References"
echo "==========================================="

# Archive parent concept (preserve for audit trail)
archive_dir=".planning/.archived/concepts"
mkdir -p "$archive_dir"
mv "$parent_path" "$archive_dir/${parent_concept}_split_${backup_timestamp}"

# Update concept registry
remove_concept_from_registry "$parent_concept"

# Register child concepts
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_full_name="${child}-${child_uuid}"
  
  add_concept_to_registry "$child_full_name" "split_from" "$parent_concept"
done

# Update focus group references
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_full_name="${child}-${child_uuid}"
  
  update_focus_group_references_for_child "$parent_concept" "$child_full_name"
done

# Send notifications
notify_concept_split "$parent_concept" "${child_concepts[@]}"

echo "✅ Parent archived and references updated"
echo "✅ Notifications sent to impacted focus groups"
```

### Step 11: Validation and Summary

```bash
echo ""
echo "🔍 Post-Split Validation"
echo "========================"

# Validate all child concepts
validation_passed=true
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  child_full_name="${child}-${child_uuid}"
  
  if validate_concept_integrity "$child_full_name"; then
    echo "✅ $child validation passed"
  else
    echo "❌ $child validation failed"
    validation_passed=false
  fi
done

if [ "$validation_passed" = false ]; then
  echo "❌ Split validation failed - initiating rollback"
  restore_from_backup "$backup_dir"
  exit 1
fi

# Summary report
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Split Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Split: $parent_concept → ${child_concepts[*]}"
echo "📁 Backup: $backup_dir"
echo "📊 Complexity: $complexity_score/10"
echo "🔄 Strategy: $split_strategy"
echo "📤 Notifications sent to impacted focus groups"
echo ""
echo "Child concepts created:"
for child in "${child_concepts[@]}"; do
  child_uuid="${child_uuids[$child]}"
  echo "  • ${child}-${child_uuid}"
done
echo ""
echo "Next steps:"
echo "1. Review child concepts and refine content"
echo "2. Update implementation plans for parallel development"
echo "3. Get approval from impacted focus groups"
echo ""
echo "Rollback command (if needed):"
echo "wgsd restore-backup $backup_dir"
```

---

## Split Strategies

### Section-Based Split
- **Method:** Divide concept by major content sections
- **Algorithm:** Map sections to child concepts based on naming/content
- **Best For:** Well-structured concepts with clear sectional divisions
- **Example:** User management → auth (auth sections), profile (profile sections), permissions (permission sections)

### Theme-Based Split  
- **Method:** Analyze content themes and functional domains
- **Algorithm:** Semantic analysis to group related functionality
- **Best For:** Complex concepts with multiple functional domains
- **Example:** E-commerce system → payments, inventory, orders based on thematic analysis

### Impact-Based Split
- **Method:** Split based on focus group impact patterns
- **Algorithm:** Child concepts inherit impacts most relevant to their domain
- **Best For:** Cross-cutting concepts with clear focus group boundaries
- **Example:** Security concept → auth (security + api), compliance (security + legal), infrastructure (security + ops)

### Size-Based Split
- **Method:** Divide content evenly by volume
- **Algorithm:** Distribute content sections to balance child concept sizes
- **Best For:** Large concepts without clear functional boundaries
- **Example:** Split large API documentation into manageable chunks

### Custom Split
- **Method:** Manual assignment of content to child concepts
- **Algorithm:** Interactive workflow for precise content control
- **Best For:** Complex concepts requiring human judgment
- **Example:** Platform redesign with custom component assignment

---

## Parent-Child Relationships

After split, relationships are maintained:

```yaml
# Parent concept (archived)
original_concept: user-management-a1b2
split_date: 2026-02-23
split_into:
  - user-auth-c3d4
  - user-profile-e5f6  
  - user-permissions-g7h8

# Child concept metadata
parent_concept: user-management-a1b2
split_strategy: theme-based
inherited_impacts: security, api
sibling_concepts: [user-profile-e5f6, user-permissions-g7h8]
```

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Parent not found | Invalid concept UUID | Check concept name and UUID |
| Active implementation | Concept in current use | Complete implementation first |
| Name conflicts | Child concept names exist | Choose unique names |
| Low complexity | Concept not complex enough | Consider keeping unified |
| Validation failure | Child concept invalid | Automatic rollback initiated |

---

## Examples

### Example 1: User Management Split
```bash
# Split complex user management concept
/wgsd split-concept user-management-a1b2 user-auth user-profile user-permissions

# Result: Three focused concepts for parallel development
```

### Example 2: API Redesign Split
```bash
# Split large API concept by themes
/wgsd split-concept api-redesign-c3d4 --children api-auth api-data api-integration --strategy theme-based

# Result: Three API-focused concepts with distributed impacts
```

---

*WGSD v2.3 Concept Split Operations - Break complex concepts into focused, manageable sub-concepts for better collaboration*