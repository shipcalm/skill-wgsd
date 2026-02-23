---
name: wgsd:lib:branch-ops
description: Branch strategy enforcement library for WGSD workspaces
---

# Branch Operations Library

Branch strategy enforcement and worktree management for WGSD workspaces.

---

## Branch Strategy

### Branch Hierarchy

```
main/develop (primary)
├── concepts/
│   ├── oauth-integration   ← Concept development (medium-lived)
│   ├── billing-v2          ← Concept development (medium-lived)
│   └── dark-mode           ← Concept development (medium-lived)
├── focus-groups/
│   ├── security            ← Planning work (long-lived)
│   ├── onboarding          ← Planning work (long-lived)
│   └── billing             ← Planning work (long-lived)
└── implementations/
    ├── auth-v2             ← Code execution (short-lived)
    └── payment-api         ← Code execution (short-lived)
```

### Branch Rules

| Type | Base | Merge Target | Lifetime | Clean Required |
|------|------|--------------|----------|----------------|
| Primary | - | - | Permanent | Yes |
| roadmap | Primary | None | Permanent | Yes |
| concepts/* | Primary | roadmap (via approval) | Medium-lived | No |
| focus-groups/* | Primary | None | Long-lived | No |
| implementations/* | **roadmap** | Primary | 1-3 days | Yes |

### Concept Branch Flow (v2.2)

```
concepts/{name}          ← Concept development happens here
    │
    ▼ (PR on approval)
focus-groups/{fg}        ← Focus group review
    │
    ▼ (full matrix approval)
roadmap                  ← Approved backlog (auto-merge on completion)
    │
    ▼ (create-implementation)
implementations/{name}   ← Code execution (branches FROM roadmap)
    │
    ▼ (merge)
develop/main            ← Production code
```

### Three-Tier Branching (Phase 13)

```
main/develop (primary)
├── roadmap                        # Approved concepts backlog (permanent)
├── concepts/                      # Concept branches
│   ├── oauth-integration          
│   └── billing-v2                
├── focus-groups/                  # Focus group branches
│   ├── security                   
│   └── billing                   
└── implementations/               # Branch FROM roadmap
    └── auth-v2-impl               
```

---

## detect_primary_branch

Detect the primary branch (develop preferred, falls back to main).

```bash
# Usage: detect_primary_branch
# Returns: "develop" or "main" (stdout), or error
detect_primary_branch() {
  # Check for develop first (preferred for WGSD)
  if git rev-parse --verify origin/develop >/dev/null 2>&1; then
    echo "develop"
    return 0
  elif git rev-parse --verify origin/main >/dev/null 2>&1; then
    echo "main"
    return 0
  elif git rev-parse --verify develop >/dev/null 2>&1; then
    echo "develop"
    return 0
  elif git rev-parse --verify main >/dev/null 2>&1; then
    echo "main"
    return 0
  else
    echo "ERROR: No primary branch (main or develop) found"
    return 1
  fi
}
```

---

## ensure_clean_checkout

Validate working directory is clean, optionally checkout specific branch.

```bash
# Usage: ensure_clean_checkout [branch]
# Returns: 0 if clean, 1 if dirty (with list of dirty files)
ensure_clean_checkout() {
  local branch="${1:-}"
  
  # Check for uncommitted changes
  local dirty_count=$(git status --porcelain 2>/dev/null | wc -l)
  
  if [ "$dirty_count" -gt 0 ]; then
    echo "ERROR: Uncommitted changes detected ($dirty_count files)"
    echo "DIRTY_FILES:"
    git status --porcelain
    echo ""
    echo "ACTION_REQUIRED: Commit or stash changes before proceeding"
    return 1
  fi
  
  # Switch branch if specified
  if [ -n "$branch" ]; then
    local current=$(git branch --show-current 2>/dev/null)
    if [ "$current" != "$branch" ]; then
      git checkout "$branch" 2>&1
      if [ $? -ne 0 ]; then
        echo "ERROR: Failed to checkout $branch"
        return 1
      fi
      echo "OK: Switched to $branch"
    fi
  fi
  
  echo "OK: Clean checkout on $(git branch --show-current)"
  return 0
}
```

---

## create_concept_branch

Create a concept branch from primary for isolated concept development.

```bash
# Usage: create_concept_branch <concept-slug>
# Returns: 0 on success, 1 on error
create_concept_branch() {
  local slug="$1"
  
  if [ -z "$slug" ]; then
    echo "ERROR: Concept slug required"
    return 1
  fi
  
  local primary=$(detect_primary_branch)
  if [ $? -ne 0 ]; then
    echo "$primary"
    return 1
  fi
  
  local branch="concepts/$slug"
  
  # Check if branch already exists locally
  if git rev-parse --verify "$branch" >/dev/null 2>&1; then
    echo "EXISTS: Branch $branch already exists locally"
    git checkout "$branch"
    return 0
  fi
  
  # Check if branch exists on remote
  if git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
    # Checkout from remote
    git checkout -b "$branch" "origin/$branch" 2>&1
    if [ $? -eq 0 ]; then
      echo "OK: Checked out existing remote branch $branch"
      return 0
    fi
  fi
  
  # Ensure primary is up to date
  git fetch origin "$primary" 2>/dev/null
  git checkout "$primary" 2>/dev/null
  git pull origin "$primary" 2>/dev/null
  
  # Create new branch from primary
  git checkout -b "$branch" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "OK: Created concept branch $branch from $primary"
    return 0
  else
    echo "ERROR: Failed to create concept branch $branch"
    return 1
  fi
}
```

---

## delete_concept_branch

Delete a concept branch after concept is promoted or abandoned.

```bash
# Usage: delete_concept_branch <concept-slug> [--force]
# Returns: 0 on success, 1 on error
delete_concept_branch() {
  local slug="$1"
  local force="${2:-}"
  
  if [ -z "$slug" ]; then
    echo "ERROR: Concept slug required"
    return 1
  fi
  
  local branch="concepts/$slug"
  
  # Check if branch exists
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    if ! git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
      echo "WARNING: Branch $branch does not exist"
      return 0
    fi
  fi
  
  # Switch away from branch if currently on it
  local current=$(git branch --show-current 2>/dev/null)
  if [ "$current" = "$branch" ]; then
    local primary=$(detect_primary_branch)
    git checkout "$primary" 2>/dev/null
  fi
  
  # Check for unmerged commits unless force flag
  if [ "$force" != "--force" ]; then
    local unmerged=$(git log "origin/$branch" --not --remotes="origin/*" --oneline 2>/dev/null | head -1)
    if [ -n "$unmerged" ]; then
      echo "ERROR: Branch has unmerged commits. Use --force to delete anyway."
      return 1
    fi
  fi
  
  # Delete local branch
  git branch -D "$branch" 2>/dev/null
  
  # Delete remote branch
  git push origin --delete "$branch" 2>/dev/null
  
  echo "OK: Deleted concept branch $branch"
  return 0
}
```

---

## list_concept_branches

List all concept branches.

```bash
# Usage: list_concept_branches
# Returns: List of concept names (stdout)
list_concept_branches() {
  git branch -a 2>/dev/null | \
    grep "concepts/" | \
    sed 's|.*concepts/||' | \
    sed 's|^remotes/origin/||' | \
    sort -u
}
```

---

## create_focus_branch

Create a focus group branch from primary.

```bash
# Usage: create_focus_branch <name>
# Returns: 0 on success, 1 on error
create_focus_branch() {
  local name="$1"
  
  if [ -z "$name" ]; then
    echo "ERROR: Focus branch name required"
    return 1
  fi
  
  local primary=$(detect_primary_branch)
  if [ $? -ne 0 ]; then
    echo "$primary"
    return 1
  fi
  
  local branch="focus-groups/$name"
  
  # Check if branch already exists
  if git rev-parse --verify "$branch" >/dev/null 2>&1; then
    echo "EXISTS: Branch $branch already exists"
    return 0
  fi
  
  # Check remote
  if git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
    # Checkout from remote
    git checkout -b "$branch" "origin/$branch" 2>&1
    if [ $? -eq 0 ]; then
      echo "OK: Checked out existing remote branch $branch"
      return 0
    fi
  fi
  
  # Create new branch from primary
  git checkout "$primary" 2>/dev/null
  git checkout -b "$branch" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "OK: Created branch $branch from $primary"
    return 0
  else
    echo "ERROR: Failed to create branch $branch"
    return 1
  fi
}
```

---

## ensure_roadmap_branch

Create roadmap branch if missing, returns branch name.

```bash
# Usage: ensure_roadmap_branch
# Returns: "roadmap" on stdout, 0 on success
ensure_roadmap_branch() {
  local primary=$(detect_primary_branch)
  
  # Check local
  if git rev-parse --verify roadmap >/dev/null 2>&1; then
    echo "roadmap"
    return 0
  fi
  
  # Check remote
  if git rev-parse --verify origin/roadmap >/dev/null 2>&1; then
    git checkout -b roadmap origin/roadmap 2>/dev/null
    git checkout "$primary" 2>/dev/null
    echo "roadmap"
    return 0
  fi
  
  # Create from primary
  git checkout "$primary" 2>/dev/null
  git pull origin "$primary" 2>/dev/null
  git checkout -b roadmap 2>/dev/null
  
  # Initialize manifest
  mkdir -p .planning
  cat > .planning/ROADMAP-MANIFEST.md << EOF
# Roadmap Manifest

**Branch:** roadmap
**Created:** $(date -u +"%Y-%m-%dT%H:%M:%SZ")

## Approved Concepts

| Concept | Approval Date | Impacts | Priority |
|---------|---------------|---------|----------|

## Sync History

| Date | Type | Details |
|------|------|---------|
EOF
  
  git add .planning/ROADMAP-MANIFEST.md
  git commit -m "docs: initialize roadmap branch"
  git push -u origin roadmap 2>/dev/null || true
  
  git checkout "$primary" 2>/dev/null
  echo "roadmap"
  return 0
}
```

---

## is_on_roadmap

Check if a concept has been merged to roadmap.

```bash
# Usage: is_on_roadmap <concept-slug>
# Returns: 0 if on roadmap, 1 if not
is_on_roadmap() {
  local concept_slug="$1"
  
  # Check if concept directory exists on roadmap branch
  local roadmap_exists=$(git ls-tree -d roadmap --name-only 2>/dev/null | grep -c "concepts/${concept_slug}$" || echo "0")
  
  # Also check manifest
  local in_manifest="0"
  if git show roadmap:.planning/ROADMAP-MANIFEST.md >/dev/null 2>&1; then
    in_manifest=$(git show roadmap:.planning/ROADMAP-MANIFEST.md 2>/dev/null | grep -c "| ${concept_slug} |" || echo "0")
  fi
  
  if [ "$roadmap_exists" -gt 0 ] || [ "$in_manifest" -gt 0 ]; then
    return 0  # On roadmap
  fi
  
  return 1  # Not on roadmap
}
```

---

## get_roadmap_concepts

List all concepts currently on the roadmap branch.

```bash
# Usage: get_roadmap_concepts
# Returns: List of concept names (one per line)
get_roadmap_concepts() {
  if ! git rev-parse --verify roadmap >/dev/null 2>&1; then
    echo ""
    return 1
  fi
  
  # Parse from manifest
  if git show roadmap:.planning/ROADMAP-MANIFEST.md >/dev/null 2>&1; then
    git show roadmap:.planning/ROADMAP-MANIFEST.md 2>/dev/null | \
      grep -E "^\| [a-z]" | \
      cut -d'|' -f2 | \
      xargs -n1 | \
      sort -u
  fi
  
  return 0
}
```

---

## create_impl_branch

Create an implementation branch from **roadmap** (Phase 13: requires clean state and concept on roadmap).

```bash
# Usage: create_impl_branch <name> [--from-develop]
# Arguments:
#   name           - Implementation branch name
#   --from-develop - Emergency mode: branch from develop instead of roadmap
# Returns: 0 on success, 1 on error
create_impl_branch() {
  local name="$1"
  local force_from_develop="${2:-}"
  
  if [ -z "$name" ]; then
    echo "ERROR: Implementation branch name required"
    return 1
  fi
  
  local branch="implementations/$name"
  
  # Check if branch already exists
  if git rev-parse --verify "$branch" >/dev/null 2>&1; then
    echo "ERROR: Implementation branch $branch already exists"
    echo "HINT: Implementation branches should be short-lived. Complete or remove the existing one."
    return 1
  fi
  
  # Determine base branch (Phase 13: default to roadmap)
  local base_branch="roadmap"
  
  if [ "$force_from_develop" = "--from-develop" ]; then
    # Emergency/hotfix mode: branch from develop
    base_branch=$(detect_primary_branch)
    echo "WARNING: EMERGENCY MODE - Branching from $base_branch instead of roadmap"
  else
    # Normal mode: ensure roadmap exists and use it
    ensure_roadmap_branch >/dev/null
    base_branch="roadmap"
    
    # Verify concept is on roadmap (Phase 13 requirement)
    if ! is_on_roadmap "$name"; then
      echo "WARNING: Concept '$name' is not on the roadmap branch"
      echo "HINT: Concepts must be fully approved to appear on roadmap"
      echo "      Use --from-develop for emergency bypass"
    fi
  fi
  
  # Ensure clean checkout
  local clean_result=$(ensure_clean_checkout "$base_branch")
  if [ $? -ne 0 ]; then
    echo "$clean_result"
    return 1
  fi
  
  # Update base branch
  git pull origin "$base_branch" 2>/dev/null || true
  
  # Create branch
  git checkout -b "$branch" "$base_branch" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "OK: Created implementation branch $branch from $base_branch"
    return 0
  else
    echo "ERROR: Failed to create branch $branch"
    return 1
  fi
}
```

---

## setup_worktree

Create a git worktree with validation.

```bash
# Usage: setup_worktree <path> <branch>
# Returns: 0 on success, 1 on error
setup_worktree() {
  local path="$1"
  local branch="$2"
  
  if [ -z "$path" ] || [ -z "$branch" ]; then
    echo "ERROR: setup_worktree requires path and branch"
    return 1
  fi
  
  # Check if worktree already exists at path
  if [ -d "$path" ]; then
    # Check if it's a valid worktree
    if [ -f "$path/.git" ]; then
      echo "EXISTS: Worktree already exists at $path"
      return 0
    else
      echo "ERROR: Directory exists but is not a worktree: $path"
      return 1
    fi
  fi
  
  # Validate branch exists
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    # Try to create the branch first if it doesn't exist
    if echo "$branch" | grep -qE '^focus-groups/'; then
      local fg_name=$(echo "$branch" | sed 's|focus-groups/||')
      create_focus_branch "$fg_name"
    elif echo "$branch" | grep -qE '^implementations/'; then
      local impl_name=$(echo "$branch" | sed 's|implementations/||')
      create_impl_branch "$impl_name"
    else
      echo "ERROR: Branch $branch does not exist"
      return 1
    fi
  fi
  
  # Create parent directories
  mkdir -p "$(dirname "$path")"
  
  # Create worktree
  git worktree add "$path" "$branch" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "OK: Created worktree at $path for $branch"
    return 0
  else
    echo "ERROR: Failed to create worktree"
    return 1
  fi
}
```

---

## remove_worktree

Remove a git worktree safely.

```bash
# Usage: remove_worktree <path>
# Returns: 0 on success, 1 on error
remove_worktree() {
  local path="$1"
  
  if [ -z "$path" ]; then
    echo "ERROR: Worktree path required"
    return 1
  fi
  
  if [ ! -d "$path" ]; then
    echo "WARNING: Worktree path does not exist: $path"
    return 0
  fi
  
  # Check for uncommitted changes in worktree
  if [ -d "$path/.git" ] || [ -f "$path/.git" ]; then
    cd "$path"
    local dirty=$(git status --porcelain 2>/dev/null | wc -l)
    cd - >/dev/null
    
    if [ "$dirty" -gt 0 ]; then
      echo "ERROR: Worktree has uncommitted changes"
      echo "HINT: Commit or stash changes before removing"
      return 1
    fi
  fi
  
  git worktree remove "$path" --force 2>&1
  
  if [ $? -eq 0 ]; then
    echo "OK: Removed worktree at $path"
    return 0
  else
    echo "ERROR: Failed to remove worktree"
    return 1
  fi
}
```

---

## list_focus_branches

List all focus group branches.

```bash
# Usage: list_focus_branches
# Returns: List of focus group names (stdout)
list_focus_branches() {
  git branch -a 2>/dev/null | \
    grep "focus-groups/" | \
    sed 's|.*focus-groups/||' | \
    sort -u
}
```

---

## list_impl_branches

List all implementation branches.

```bash
# Usage: list_impl_branches
# Returns: List of implementation names (stdout)
list_impl_branches() {
  git branch -a 2>/dev/null | \
    grep "implementations/" | \
    sed 's|.*implementations/||' | \
    sort -u
}
```

---

## validate_branch_state

Validate all branches are in good state.

```bash
# Usage: validate_branch_state
# Returns: Status report (stdout)
validate_branch_state() {
  echo "🔍 Branch State Validation"
  echo ""
  
  # Check primary branch
  local primary=$(detect_primary_branch)
  if [ $? -ne 0 ]; then
    echo "❌ No primary branch found"
    return 1
  fi
  echo "✅ Primary: $primary"
  
  # List concept branches (v2.2)
  local concept_count=$(list_concept_branches | wc -l)
  echo "💡 Concepts: $concept_count"
  list_concept_branches | while read concept; do
    echo "   - $concept"
  done
  
  # List focus branches
  local focus_count=$(list_focus_branches | wc -l)
  echo "📂 Focus Groups: $focus_count"
  list_focus_branches | while read fg; do
    echo "   - $fg"
  done
  
  # List implementation branches
  local impl_count=$(list_impl_branches | wc -l)
  echo "🚀 Implementations: $impl_count"
  list_impl_branches | while read impl; do
    echo "   - $impl"
  done
  
  # Check for stale implementations (branches older than 7 days)
  echo ""
  echo "⚠️  Warnings:"
  local warnings=0
  
  if [ "$impl_count" -gt 4 ]; then
    echo "   - Too many implementations ($impl_count > 4 recommended)"
    warnings=$((warnings + 1))
  fi
  
  if [ "$concept_count" -gt 10 ]; then
    echo "   - Many active concepts ($concept_count) - consider consolidating"
    warnings=$((warnings + 1))
  fi
  
  if [ "$warnings" -eq 0 ]; then
    echo "   None"
  fi
  
  return 0
}
```

---

## Usage Example

```bash
# In a WGSD workflow:

# Detect primary and ensure clean
PRIMARY=$(detect_primary_branch)
ensure_clean_checkout "$PRIMARY" || exit 1

# Create concept branch (v2.2)
create_concept_branch "oauth-integration"
# Concept branch: concepts/oauth-integration

# Create optional worktree for concept
setup_worktree "./worktrees/oauth-integration" "concepts/oauth-integration"

# Create focus group with worktree
create_focus_branch "security"
setup_worktree "./concepts/security" "focus-groups/security"

# Create implementation (requires clean state)
create_impl_branch "auth-v2"
setup_worktree "./implementations/auth-v2" "implementations/auth-v2"

# List all concept branches
list_concept_branches

# Clean up concept branch after promotion
delete_concept_branch "oauth-integration"

# Validate overall state
validate_branch_state
```

---

*Library updated for WGSD v2.2 - Concept Directory Architecture*
