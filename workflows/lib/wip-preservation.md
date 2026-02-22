---
name: wgsd:lib:wip-preservation
description: Work-in-progress preservation library for GSD to WGSD migration
---

# WIP Preservation Library

Preserve work-in-progress during GSD to WGSD migration by converting active phases to implementations.

---

## Overview

When migrating from GSD to WGSD, any work-in-progress should not be lost:
- Uncommitted changes → Stashed and restored after migration
- Unpushed commits → Preserved as implementation branch
- Active phase work → Converted to active implementation

---

## preserve_uncommitted

Stash uncommitted changes for later restoration.

```bash
# Usage: preserve_uncommitted [repo_path]
# Returns: Stash reference if changes were stashed
preserve_uncommitted() {
  local repo_path="${1:-.}"
  cd "$repo_path" 2>/dev/null || return 1
  
  local dirty=$(git status --porcelain | wc -l | tr -d ' ')
  
  if [ "$dirty" -eq 0 ]; then
    echo "WIP_UNCOMMITTED:"
    echo "  has_changes: false"
    echo "  action: none"
    return 0
  fi
  
  # Create stash with descriptive message
  local stash_msg="WGSD-MIGRATION-$(date +%Y%m%d-%H%M%S)"
  git stash push -m "$stash_msg" --include-untracked 2>&1
  
  if [ $? -eq 0 ]; then
    local stash_ref=$(git stash list | grep "$stash_msg" | cut -d: -f1)
    echo "WIP_UNCOMMITTED:"
    echo "  has_changes: true"
    echo "  action: stashed"
    echo "  stash_ref: $stash_ref"
    echo "  stash_msg: $stash_msg"
    echo "  files_stashed: $dirty"
    return 0
  else
    echo "WIP_UNCOMMITTED:"
    echo "  has_changes: true"
    echo "  action: failed"
    echo "  error: Could not stash changes"
    return 1
  fi
}
```

---

## restore_uncommitted

Restore stashed changes after migration.

```bash
# Usage: restore_uncommitted <stash_ref> [repo_path]
# Returns: Success/failure status
restore_uncommitted() {
  local stash_ref="$1"
  local repo_path="${2:-.}"
  
  if [ -z "$stash_ref" ]; then
    echo "ERROR: Stash reference required"
    return 1
  fi
  
  cd "$repo_path" 2>/dev/null || return 1
  
  git stash pop "$stash_ref" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "WIP_RESTORE:"
    echo "  action: restored"
    echo "  stash_ref: $stash_ref"
    return 0
  else
    echo "WIP_RESTORE:"
    echo "  action: failed"
    echo "  error: Could not restore stash $stash_ref"
    echo "  hint: Use 'git stash list' to find your changes"
    return 1
  fi
}
```

---

## detect_current_phase

Detect the current active phase for preservation.

```bash
# Usage: detect_current_phase [planning_dir]
# Returns: Current phase info for implementation conversion
detect_current_phase() {
  local planning_dir="${1:-.planning}"
  
  local current_phase=""
  local phase_number=""
  local phase_name=""
  local phase_status=""
  
  # Check STATE.md for current phase
  if [ -f "$planning_dir/STATE.md" ]; then
    # Look for current phase markers
    local phase_line=$(grep -E "^##? (Current )?Phase|^\*\*Phase" "$planning_dir/STATE.md" | head -1)
    
    if [ -n "$phase_line" ]; then
      phase_number=$(echo "$phase_line" | grep -oE '[0-9]+' | head -1)
      phase_name=$(echo "$phase_line" | sed -E 's/.*Phase[^:]*[: -]+//' | sed 's/[*#]//g' | xargs)
    fi
    
    # Check status
    if grep -qE "IN PROGRESS|🟡|50%|75%" "$planning_dir/STATE.md"; then
      phase_status="in_progress"
    elif grep -qE "COMPLETE|✅|100%" "$planning_dir/STATE.md"; then
      phase_status="complete"
    else
      phase_status="unknown"
    fi
  fi
  
  # Check phases directory
  if [ -z "$phase_number" ] && [ -d "$planning_dir/phases" ]; then
    # Get most recently modified phase directory
    local latest=$(ls -dt "$planning_dir/phases/"*/ 2>/dev/null | head -1)
    if [ -n "$latest" ]; then
      local dirname=$(basename "$latest")
      phase_number=$(echo "$dirname" | grep -oE '[0-9]+' | head -1)
      phase_name=$(echo "$dirname" | sed -E 's/^phase-?[0-9]+-?//')
    fi
  fi
  
  cat << EOF
CURRENT_PHASE:
  phase_number: $phase_number
  phase_name: "$phase_name"
  phase_status: $phase_status
  preserve_as_impl: $([ "$phase_status" = "in_progress" ] && echo true || echo false)
EOF
  return 0
}
```

---

## create_wip_implementation

Convert active phase work to WGSD implementation.

```bash
# Usage: create_wip_implementation <stub> <phase_name> [repo_path]
# Returns: Created implementation details
create_wip_implementation() {
  local stub="$1"
  local phase_name="$2"
  local repo_path="${3:-.}"
  
  if [ -z "$stub" ] || [ -z "$phase_name" ]; then
    echo "ERROR: Stub and phase_name required"
    return 1
  fi
  
  cd "$repo_path" 2>/dev/null || return 1
  
  # Generate implementation name
  local impl_slug=$(echo "$phase_name" | \
    tr '[:upper:]' '[:lower:]' | \
    tr ' ' '-' | \
    tr -cd 'a-z0-9-' | \
    sed 's/--*/-/g' | \
    sed 's/^-//' | \
    sed 's/-$//' | \
    cut -c1-20)
  
  # Add version suffix if name is too generic
  if [ ${#impl_slug} -lt 3 ]; then
    impl_slug="${impl_slug}-wip"
  fi
  
  local impl_name="${stub}-impl-${impl_slug}"
  local impl_branch="implementations/${impl_slug}"
  
  # Check if branch already exists
  if git rev-parse --verify "$impl_branch" >/dev/null 2>&1; then
    impl_slug="${impl_slug}-$(date +%m%d)"
    impl_name="${stub}-impl-${impl_slug}"
    impl_branch="implementations/${impl_slug}"
  fi
  
  # Get primary branch
  local primary="develop"
  if ! git rev-parse --verify "$primary" >/dev/null 2>&1; then
    primary="main"
  fi
  
  # Create implementation branch from current state
  git checkout -b "$impl_branch" 2>&1
  
  if [ $? -eq 0 ]; then
    cat << EOF
WIP_IMPLEMENTATION:
  created: true
  impl_name: $impl_name
  impl_branch: $impl_branch
  impl_slug: $impl_slug
  base_branch: $primary
  channel: #${impl_name}
  status: Ready for implementation work
EOF
    return 0
  else
    cat << EOF
WIP_IMPLEMENTATION:
  created: false
  error: Could not create branch $impl_branch
EOF
    return 1
  fi
}
```

---

## preserve_phase_as_implementation

Full workflow to preserve current phase as implementation.

```bash
# Usage: preserve_phase_as_implementation <stub> [repo_path]
# Returns: Complete preservation result
preserve_phase_as_implementation() {
  local stub="$1"
  local repo_path="${2:-.}"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "📦 Preserving Work-in-Progress"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  
  cd "$repo_path" 2>/dev/null || {
    echo "ERROR: Cannot access $repo_path"
    return 1
  }
  
  # Step 1: Detect current phase
  echo "🔍 Detecting current phase..."
  local phase_result=$(detect_current_phase ".planning")
  echo "$phase_result"
  echo ""
  
  local phase_name=$(echo "$phase_result" | grep "phase_name:" | cut -d'"' -f2)
  local should_preserve=$(echo "$phase_result" | grep "preserve_as_impl:" | cut -d: -f2 | xargs)
  
  if [ "$should_preserve" != "true" ]; then
    echo "ℹ️  No active work-in-progress to preserve"
    echo ""
    return 0
  fi
  
  # Step 2: Stash any uncommitted changes
  echo "📋 Checking for uncommitted changes..."
  local stash_result=$(preserve_uncommitted)
  echo "$stash_result"
  echo ""
  
  local stash_ref=$(echo "$stash_result" | grep "stash_ref:" | cut -d: -f2 | xargs)
  
  # Step 3: Create implementation branch
  echo "🔧 Creating implementation branch..."
  local impl_result=$(create_wip_implementation "$stub" "$phase_name")
  echo "$impl_result"
  echo ""
  
  local impl_created=$(echo "$impl_result" | grep "created:" | cut -d: -f2 | xargs)
  local impl_name=$(echo "$impl_result" | grep "impl_name:" | cut -d: -f2 | xargs)
  
  # Step 4: Restore stashed changes if any
  if [ -n "$stash_ref" ]; then
    echo "📥 Restoring uncommitted changes..."
    restore_uncommitted "$stash_ref"
    echo ""
  fi
  
  # Summary
  echo "═══════════════════════════════════════════════════════════"
  if [ "$impl_created" = "true" ]; then
    echo "✅ Work-in-Progress Preserved"
    echo ""
    echo "   Implementation: $impl_name"
    echo "   Your active work is now on the implementation branch."
    echo "   Continue working as normal!"
  else
    echo "⚠️  Preservation encountered issues"
    echo "   Your changes are still available in the stash."
  fi
  echo "═══════════════════════════════════════════════════════════"
  
  return 0
}
```

---

## generate_impl_name

Generate a valid implementation name from context.

```bash
# Usage: generate_impl_name <stub> [context_string]
# Returns: Valid implementation name
generate_impl_name() {
  local stub="$1"
  local context="${2:-wip}"
  
  # Clean and normalize context
  local slug=$(echo "$context" | \
    tr '[:upper:]' '[:lower:]' | \
    sed -E 's/^phase[-_ ]*[0-9]+[-_: ]*//' | \
    tr ' _' '--' | \
    tr -cd 'a-z0-9-' | \
    sed 's/--*/-/g' | \
    sed 's/^-//' | \
    sed 's/-$//' | \
    cut -c1-25)
  
  # Ensure minimum length
  if [ ${#slug} -lt 2 ]; then
    slug="wip-$(date +%m%d)"
  fi
  
  echo "${stub}-impl-${slug}"
}
```

---

## Usage Examples

```bash
# Preserve uncommitted changes
preserve_uncommitted /path/to/repo

# Detect current phase
detect_current_phase /path/to/.planning

# Create implementation from WIP
create_wip_implementation "mvn" "Security Auth" /path/to/repo

# Full preservation workflow
preserve_phase_as_implementation "mvn" /path/to/repo

# Generate implementation name
generate_impl_name "mvn" "Phase 2: Security Implementation"
# Returns: mvn-impl-security-implementation
```

---

*Library created for WGSD Phase 2 - MIGRATE-04, MIGRATE-05*
