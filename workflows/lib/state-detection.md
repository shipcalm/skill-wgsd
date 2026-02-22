---
name: wgsd:lib:state-detection
description: GSD project state detection library for migration analysis
---

# State Detection Library

Detect the current state of a GSD project for intelligent migration.

---

## Overview

This library provides functions to detect:
- Git repository state (branch, uncommitted changes, sync status)
- GSD planning state (current phase, completion status)
- Work-in-progress indicators (draft documents, incomplete phases)
- Migration readiness assessment

---

## detect_git_state

Get comprehensive git repository state.

```bash
# Usage: detect_git_state [repo_path]
# Output: JSON-like structured output
detect_git_state() {
  local repo_path="${1:-.}"
  cd "$repo_path" 2>/dev/null || {
    echo "ERROR: Cannot access repository: $repo_path"
    return 1
  }
  
  # Ensure we're in a git repository
  if ! git rev-parse --git-dir >/dev/null 2>&1; then
    echo "ERROR: Not a git repository: $repo_path"
    return 1
  fi
  
  # Gather state
  local branch=$(git branch --show-current 2>/dev/null || echo "detached")
  local dirty_count=$(git status --porcelain 2>/dev/null | wc -l | tr -d ' ')
  local staged_count=$(git diff --cached --numstat 2>/dev/null | wc -l | tr -d ' ')
  local untracked_count=$(git status --porcelain 2>/dev/null | grep '^??' | wc -l | tr -d ' ')
  
  # Check ahead/behind
  local ahead=0
  local behind=0
  if git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
    git fetch origin --quiet 2>/dev/null
    ahead=$(git rev-list --count "origin/$branch..$branch" 2>/dev/null || echo "0")
    behind=$(git rev-list --count "$branch..origin/$branch" 2>/dev/null || echo "0")
  fi
  
  # Get last commit info
  local last_commit=$(git log -1 --format="%H" 2>/dev/null)
  local last_commit_date=$(git log -1 --format="%ci" 2>/dev/null)
  local last_commit_msg=$(git log -1 --format="%s" 2>/dev/null)
  
  # Determine clean status
  local clean="true"
  [ "$dirty_count" -gt 0 ] && clean="false"
  
  # Determine sync status
  local sync_status="synced"
  [ "$ahead" -gt 0 ] && [ "$behind" -gt 0 ] && sync_status="diverged"
  [ "$ahead" -gt 0 ] && [ "$behind" -eq 0 ] && sync_status="ahead"
  [ "$ahead" -eq 0 ] && [ "$behind" -gt 0 ] && sync_status="behind"
  
  # Output structured result
  cat << EOF
GIT_STATE:
  branch: $branch
  clean: $clean
  dirty_files: $dirty_count
  staged_files: $staged_count
  untracked_files: $untracked_count
  ahead: $ahead
  behind: $behind
  sync_status: $sync_status
  last_commit: $last_commit
  last_commit_date: $last_commit_date
  last_commit_msg: $last_commit_msg
EOF
  return 0
}
```

---

## detect_gsd_state

Analyze GSD planning structure and current phase.

```bash
# Usage: detect_gsd_state [repo_path]
# Output: Structured GSD state information
detect_gsd_state() {
  local repo_path="${1:-.}"
  local planning_dir="$repo_path/.planning"
  
  # Check if .planning exists
  if [ ! -d "$planning_dir" ]; then
    echo "GSD_STATE:"
    echo "  exists: false"
    echo "  migration_type: fresh"
    return 0
  fi
  
  # Detect GSD files
  local has_project="false"
  local has_roadmap="false"
  local has_requirements="false"
  local has_state="false"
  local has_phases="false"
  local phase_count=0
  
  [ -f "$planning_dir/PROJECT.md" ] && has_project="true"
  [ -f "$planning_dir/ROADMAP.md" ] && has_roadmap="true"
  [ -f "$planning_dir/REQUIREMENTS.md" ] && has_requirements="true"
  [ -f "$planning_dir/STATE.md" ] && has_state="true"
  
  if [ -d "$planning_dir/phases" ]; then
    has_phases="true"
    phase_count=$(ls -d "$planning_dir/phases/"*/ 2>/dev/null | wc -l | tr -d ' ')
  fi
  
  # Detect current phase from STATE.md
  local current_phase=""
  local phase_status=""
  if [ -f "$planning_dir/STATE.md" ]; then
    # Look for current phase marker
    current_phase=$(grep -E "^\*\*Phase|^## Current Phase|^#.*Phase [0-9]" "$planning_dir/STATE.md" | head -1 | sed 's/[*#]//g' | xargs)
    
    # Look for completion status
    if grep -qE "✅ COMPLETE|COMPLETE|100%" "$planning_dir/STATE.md"; then
      phase_status="complete"
    elif grep -qE "🟡|IN PROGRESS|50%|75%" "$planning_dir/STATE.md"; then
      phase_status="in_progress"
    else
      phase_status="planned"
    fi
  fi
  
  # Check for WGSD markers (already migrated)
  local already_migrated="false"
  if [ -f "$planning_dir/WGSD-CONFIG.md" ] || [ -d "$planning_dir/focus-groups" ]; then
    already_migrated="true"
  fi
  
  # Determine migration type
  local migration_type="gsd_to_wgsd"
  [ "$already_migrated" = "true" ] && migration_type="already_wgsd"
  [ "$phase_count" -eq 0 ] && [ "$has_roadmap" = "false" ] && migration_type="minimal"
  
  # Output structured result
  cat << EOF
GSD_STATE:
  exists: true
  already_migrated: $already_migrated
  migration_type: $migration_type
  files:
    project: $has_project
    roadmap: $has_roadmap
    requirements: $has_requirements
    state: $has_state
    phases: $has_phases
  phase_count: $phase_count
  current_phase: "$current_phase"
  phase_status: $phase_status
EOF
  return 0
}
```

---

## detect_wip_state

Detect work-in-progress indicators for preservation.

```bash
# Usage: detect_wip_state [repo_path]
# Output: WIP indicators for migration preservation
detect_wip_state() {
  local repo_path="${1:-.}"
  local planning_dir="$repo_path/.planning"
  
  local has_wip="false"
  local wip_type=""
  local wip_files=""
  local wip_branch=""
  
  # Check for uncommitted planning changes
  cd "$repo_path" 2>/dev/null || return 1
  local planning_dirty=$(git status --porcelain .planning/ 2>/dev/null | wc -l | tr -d ' ')
  
  if [ "$planning_dirty" -gt 0 ]; then
    has_wip="true"
    wip_type="uncommitted_changes"
    wip_files=$(git status --porcelain .planning/ 2>/dev/null | head -5)
  fi
  
  # Check for unpushed commits
  local branch=$(git branch --show-current 2>/dev/null)
  if [ -n "$branch" ]; then
    local unpushed=$(git rev-list --count "origin/$branch..$branch" 2>/dev/null || echo "0")
    if [ "$unpushed" -gt 0 ]; then
      has_wip="true"
      [ -n "$wip_type" ] && wip_type="$wip_type,unpushed_commits" || wip_type="unpushed_commits"
      wip_branch="$branch ($unpushed unpushed commits)"
    fi
  fi
  
  # Check for draft markers in current phase
  if [ -d "$planning_dir/phases" ]; then
    local latest_phase=$(ls -d "$planning_dir/phases/"*/ 2>/dev/null | sort | tail -1)
    if [ -n "$latest_phase" ]; then
      # Look for draft markers
      if grep -rqE "DRAFT|TODO|WIP|INCOMPLETE|⏳" "$latest_phase" 2>/dev/null; then
        has_wip="true"
        [ -n "$wip_type" ] && wip_type="$wip_type,draft_documents" || wip_type="draft_documents"
      fi
    fi
  fi
  
  # Determine WIP recommendation
  local recommendation="none"
  if [ "$has_wip" = "true" ]; then
    if echo "$wip_type" | grep -q "uncommitted"; then
      recommendation="stash_or_commit"
    elif echo "$wip_type" | grep -q "unpushed"; then
      recommendation="create_implementation"
    else
      recommendation="preserve_as_concept"
    fi
  fi
  
  cat << EOF
WIP_STATE:
  has_wip: $has_wip
  wip_type: "$wip_type"
  wip_branch: "$wip_branch"
  wip_files: |
$(echo "$wip_files" | sed 's/^/    /')
  recommendation: $recommendation
EOF
  return 0
}
```

---

## detect_phase_branch

Detect if current branch indicates a GSD phase.

```bash
# Usage: detect_phase_branch [branch_name]
# Output: Phase info if branch follows GSD naming
detect_phase_branch() {
  local branch="${1:-$(git branch --show-current 2>/dev/null)}"
  
  # Common GSD branch patterns
  # phase-01, phase-01-foundation, phases/phase-01
  if echo "$branch" | grep -qE '^phase-?[0-9]+|^phases/'; then
    local phase_num=$(echo "$branch" | grep -oE '[0-9]+' | head -1)
    local phase_name=$(echo "$branch" | sed -E 's/^(phases?\/)?phase-?[0-9]+[-_]?//' | tr '-_' ' ')
    
    echo "PHASE_BRANCH:"
    echo "  is_phase_branch: true"
    echo "  phase_number: $phase_num"
    echo "  phase_name: \"$phase_name\""
    echo "  original_branch: $branch"
    return 0
  fi
  
  echo "PHASE_BRANCH:"
  echo "  is_phase_branch: false"
  echo "  branch: $branch"
  return 0
}
```

---

## full_state_detection

Comprehensive state detection combining all checks.

```bash
# Usage: full_state_detection [repo_path]
# Output: Complete state report for migration wizard
full_state_detection() {
  local repo_path="${1:-.}"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "📊 GSD Project State Analysis"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  echo "📁 Repository: $(realpath "$repo_path")"
  echo "📅 Analysis Date: $(date -Iseconds)"
  echo ""
  
  # Git state
  echo "─────────────────────────────────────────────────────────────"
  detect_git_state "$repo_path"
  echo ""
  
  # GSD state
  echo "─────────────────────────────────────────────────────────────"
  detect_gsd_state "$repo_path"
  echo ""
  
  # Phase branch detection
  echo "─────────────────────────────────────────────────────────────"
  detect_phase_branch
  echo ""
  
  # WIP state
  echo "─────────────────────────────────────────────────────────────"
  detect_wip_state "$repo_path"
  echo ""
  
  # Migration readiness assessment
  echo "─────────────────────────────────────────────────────────────"
  echo "MIGRATION_READINESS:"
  
  cd "$repo_path" 2>/dev/null
  local ready="true"
  local blockers=""
  
  # Check for dirty state
  if [ "$(git status --porcelain | wc -l)" -gt 0 ]; then
    ready="false"
    blockers="$blockers uncommitted_changes"
  fi
  
  # Check if already WGSD
  if [ -f ".planning/WGSD-CONFIG.md" ]; then
    ready="false"
    blockers="$blockers already_migrated"
  fi
  
  echo "  ready: $ready"
  echo "  blockers: [$blockers ]"
  
  if [ "$ready" = "true" ]; then
    echo "  status: ✅ Ready for migration"
  else
    echo "  status: ⚠️  Address blockers before migration"
  fi
  echo ""
  echo "═══════════════════════════════════════════════════════════"
}
```

---

## Usage Examples

```bash
# Full state analysis
full_state_detection /path/to/project

# Just git state
detect_git_state /path/to/project

# Just GSD planning state  
detect_gsd_state /path/to/project

# Check for work-in-progress
detect_wip_state /path/to/project
```

---

*Library created for WGSD Phase 2 - MIGRATE-01*
