---
name: wgsd:lib:git-ops
description: Git operations library with error handling for WGSD workspace management
---

# Git Operations Library

Reusable git operations with consistent error handling for WGSD workflows.

---

## git_clone

Clone a repository to specified path.

```bash
# Usage: git_clone <url> <target_path>
git_clone() {
  local url="$1"
  local target_path="$2"
  
  if [ -z "$url" ] || [ -z "$target_path" ]; then
    echo "ERROR: git_clone requires url and target_path"
    return 1
  fi
  
  if [ -d "$target_path" ]; then
    echo "ERROR: Target path already exists: $target_path"
    return 1
  fi
  
  git clone "$url" "$target_path" 2>&1
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Cloned to $target_path"
    return 0
  else
    echo "ERROR: Clone failed"
    return 1
  fi
}
```

---

## git_detect_primary

Detect the primary branch (develop preferred, falls back to main).

```bash
# Usage: git_detect_primary
git_detect_primary() {
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

## git_checkout

Checkout a branch with validation.

```bash
# Usage: git_checkout <branch>
git_checkout() {
  local branch="$1"
  
  if [ -z "$branch" ]; then
    echo "ERROR: git_checkout requires branch name"
    return 1
  fi
  
  # Verify branch exists
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    # Try with origin/ prefix
    if ! git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
      echo "ERROR: Branch $branch does not exist"
      return 1
    fi
  fi
  
  git checkout "$branch" 2>&1
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Checked out $branch"
    return 0
  else
    echo "ERROR: Checkout failed"
    return 1
  fi
}
```

---

## git_ensure_clean

Validate working directory has no uncommitted changes.

```bash
# Usage: git_ensure_clean
git_ensure_clean() {
  local dirty_count=$(git status --porcelain 2>/dev/null | wc -l)
  
  if [ "$dirty_count" -gt 0 ]; then
    echo "ERROR: Uncommitted changes present"
    echo "DIRTY_FILES:"
    git status --porcelain
    echo ""
    echo "ACTION_REQUIRED: Commit or stash changes before proceeding"
    return 1
  fi
  
  echo "SUCCESS: Working directory clean"
  return 0
}
```

---

## git_status

Return structured status object.

```bash
# Usage: git_status
# Output: branch=<name> status=<clean|dirty> changes=<count>
git_status() {
  local branch=$(git branch --show-current 2>/dev/null)
  local dirty_count=$(git status --porcelain 2>/dev/null | wc -l)
  local status="clean"
  
  if [ "$dirty_count" -gt 0 ]; then
    status="dirty"
  fi
  
  echo "branch=$branch status=$status changes=$dirty_count"
}
```

---

## git_sync_status

Check ahead/behind status with remote.

```bash
# Usage: git_sync_status
# Output: ahead=<n> behind=<n> sync=<synced|ahead|behind|diverged>
git_sync_status() {
  local branch=$(git branch --show-current 2>/dev/null)
  local remote="origin/$branch"
  
  # Fetch latest (quiet)
  git fetch origin --quiet 2>/dev/null
  
  # Get ahead/behind counts
  local ahead=$(git rev-list --count "$remote..$branch" 2>/dev/null || echo "0")
  local behind=$(git rev-list --count "$branch..$remote" 2>/dev/null || echo "0")
  
  local sync_status="synced"
  if [ "$ahead" -gt 0 ] && [ "$behind" -gt 0 ]; then
    sync_status="diverged"
  elif [ "$ahead" -gt 0 ]; then
    sync_status="ahead"
  elif [ "$behind" -gt 0 ]; then
    sync_status="behind"
  fi
  
  echo "ahead=$ahead behind=$behind sync=$sync_status"
}
```

---

## git_worktree_add

Create a git worktree for a branch.

```bash
# Usage: git_worktree_add <path> <branch>
git_worktree_add() {
  local path="$1"
  local branch="$2"
  
  if [ -z "$path" ] || [ -z "$branch" ]; then
    echo "ERROR: git_worktree_add requires path and branch"
    return 1
  fi
  
  # Check if worktree already exists
  if [ -d "$path" ]; then
    echo "EXISTS: Worktree at $path already exists"
    return 0
  fi
  
  # Validate branch exists
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    echo "ERROR: Branch $branch does not exist"
    return 1
  fi
  
  git worktree add "$path" "$branch" 2>&1
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Created worktree at $path for $branch"
    return 0
  else
    echo "ERROR: Worktree creation failed"
    return 1
  fi
}
```

---

## git_worktree_remove

Remove a git worktree.

```bash
# Usage: git_worktree_remove <path>
git_worktree_remove() {
  local path="$1"
  
  if [ -z "$path" ]; then
    echo "ERROR: git_worktree_remove requires path"
    return 1
  fi
  
  if [ ! -d "$path" ]; then
    echo "WARNING: Worktree path does not exist: $path"
    return 0
  fi
  
  git worktree remove "$path" --force 2>&1
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Removed worktree at $path"
    return 0
  else
    echo "ERROR: Failed to remove worktree"
    return 1
  fi
}
```

---

## git_worktree_list

List all worktrees for the repository.

```bash
# Usage: git_worktree_list
git_worktree_list() {
  git worktree list --porcelain 2>/dev/null | grep "^worktree" | cut -d' ' -f2
}
```

---

## git_concept_worktree_add

Create a worktree specifically for concept development.

```bash
# Usage: git_concept_worktree_add <concept-slug>
# Creates: worktrees/{concept-slug}/ pointing to concepts/{concept-slug} branch
git_concept_worktree_add() {
  local slug="$1"
  
  if [ -z "$slug" ]; then
    echo "ERROR: Concept slug required"
    return 1
  fi
  
  local path="worktrees/$slug"
  local branch="concepts/$slug"
  
  # Check if worktree already exists
  if [ -d "$path" ]; then
    if [ -f "$path/.git" ]; then
      echo "EXISTS: Concept worktree at $path already exists"
      return 0
    else
      echo "ERROR: Directory exists but is not a worktree: $path"
      return 1
    fi
  fi
  
  # Validate branch exists
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    # Check remote
    if git rev-parse --verify "origin/$branch" >/dev/null 2>&1; then
      # Fetch and create local tracking branch
      git fetch origin "$branch"
      git branch "$branch" "origin/$branch" 2>/dev/null
    else
      echo "ERROR: Concept branch $branch does not exist"
      echo "HINT: Create concept first with /wgsd create-concept"
      return 1
    fi
  fi
  
  # Create worktrees directory if needed
  mkdir -p "worktrees"
  
  # Add .gitignore for worktrees directory
  if [ ! -f "worktrees/.gitignore" ]; then
    echo "*" > "worktrees/.gitignore"
    echo "!.gitignore" >> "worktrees/.gitignore"
  fi
  
  git worktree add "$path" "$branch" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Created concept worktree at $path"
    echo "BRANCH: $branch"
    echo ""
    echo "To work on this concept:"
    echo "  cd $path"
    return 0
  else
    echo "ERROR: Failed to create concept worktree"
    return 1
  fi
}
```

---

## git_concept_worktree_remove

Remove a concept worktree safely.

```bash
# Usage: git_concept_worktree_remove <concept-slug>
git_concept_worktree_remove() {
  local slug="$1"
  
  if [ -z "$slug" ]; then
    echo "ERROR: Concept slug required"
    return 1
  fi
  
  local path="worktrees/$slug"
  
  if [ ! -d "$path" ]; then
    echo "WARNING: Concept worktree does not exist: $path"
    return 0
  fi
  
  # Check for uncommitted changes
  if [ -f "$path/.git" ]; then
    local dirty=$(cd "$path" && git status --porcelain 2>/dev/null | wc -l)
    
    if [ "$dirty" -gt 0 ]; then
      echo "ERROR: Concept worktree has uncommitted changes"
      echo ""
      echo "Uncommitted files in $path:"
      (cd "$path" && git status --porcelain)
      echo ""
      echo "HINT: Commit or stash changes before removing"
      return 1
    fi
  fi
  
  git worktree remove "$path" --force 2>&1
  
  if [ $? -eq 0 ]; then
    echo "SUCCESS: Removed concept worktree at $path"
    return 0
  else
    echo "ERROR: Failed to remove concept worktree"
    return 1
  fi
}
```

---

## git_list_concept_worktrees

List all concept worktrees.

```bash
# Usage: git_list_concept_worktrees
# Returns: List of concept slugs that have worktrees
git_list_concept_worktrees() {
  if [ -d "worktrees" ]; then
    git worktree list --porcelain 2>/dev/null | \
      grep "^worktree.*worktrees/" | \
      sed 's|.*worktrees/||' | \
      sort -u
  fi
}
```

---

## Usage Example

```bash
# In a WGSD workflow:
cd "$WORKSPACE_PATH"

# Detect and checkout primary branch
PRIMARY=$(git_detect_primary)
git_ensure_clean || exit 1
git_checkout "$PRIMARY"

# Check sync status
SYNC=$(git_sync_status)
echo "Repository: $SYNC"

# Create worktree for focus group
git_worktree_add "./concepts/security" "focus-groups/security"

# Create concept worktree (v2.2)
git_concept_worktree_add "oauth-integration"
# Creates: worktrees/oauth-integration/ → concepts/oauth-integration

# Work in concept worktree
cd worktrees/oauth-integration
# Make changes...
git add . && git commit -m "Update concept"
cd ../..

# List concept worktrees
git_list_concept_worktrees

# Clean up after concept is promoted
git_concept_worktree_remove "oauth-integration"
```

---

*Library updated for WGSD v2.2 - Concept Directory Architecture*
