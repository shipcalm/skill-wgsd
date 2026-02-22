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
```

---

*Library created for WGSD Phase 1*
