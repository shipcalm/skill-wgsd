# Plan 1.1: Workspace Management

**Requirements:** INTEGRATE-01, INTEGRATE-06
**Estimated Duration:** 1 day
**Dependencies:** None

---

## Objective

Implement workspace management for `wgsd/{project}` structure with integrated git operations.

---

## Requirements Addressed

### INTEGRATE-01: Workspace management for wgsd/{project} structure
- Create standardized workspace directory structure
- Support multiple WGSD projects in parallel
- Integrate with OpenClaw workspace conventions

### INTEGRATE-06: Git operations integration for workspace and branch management
- Reliable git clone, checkout, and worktree operations
- Clean develop branch detection and checkout
- Workspace status reporting (branch, changes, sync state)

---

## Deliverables

### 1. Workflow: `workflows/init.md`

**Purpose:** Initialize new WGSD workspace from repository

**Process:**
```
1. Validate inputs (repo URL or path)
2. Create workspace directory: ~/.openclaw/workspace/wgsd/{project}/
3. Clone or link repository to workspace
4. Detect primary branch (main/develop)
5. Checkout clean develop branch
6. Create WGSD directory structure
7. Register workspace in WGSD registry
8. Return workspace status
```

**Inputs:**
- Repository URL or local path
- Project name (auto-derived from repo name)
- Slack stub (prompt user, suggest from project name)

**Outputs:**
- Workspace path
- Git status (branch, clean/dirty)
- Next steps for user

### 2. Workflow: `workflows/workspace-status.md`

**Purpose:** Report health and status of WGSD workspace

**Process:**
```
1. Detect current workspace context
2. Query git status (branch, uncommitted changes)
3. Check remote sync state (ahead/behind)
4. List active focus groups with branch status
5. List active implementations with status
6. Report any blocking issues
```

**Outputs:**
- Current branch and clean status
- Sync state with remote
- Focus group summary
- Implementation summary
- Blocking issues (if any)

### 3. Library: `lib/git-ops.md`

**Purpose:** Reusable git operations with error handling

**Operations:**
```
git_clone(url, path)        → Clone repository to path
git_checkout(branch)        → Checkout branch, validate success
git_worktree_add(path, branch) → Create worktree for branch
git_worktree_remove(path)   → Clean up worktree
git_status()                → Return structured status object
git_sync_status()           → Check ahead/behind remote
git_ensure_clean()          → Validate no uncommitted changes
git_detect_primary()        → Detect main or develop branch
```

**Error Handling:**
- Each operation returns success/failure with message
- Rollback support for multi-step operations
- Clear error messages for common failures

### 4. Registry: `.planning/WORKSPACES.md`

**Purpose:** Track all WGSD workspaces

**Format:**
```markdown
# WGSD Workspaces

| Project | Path | Stub | Repository | Status |
|---------|------|------|------------|--------|
| marvin | ~/.openclaw/workspace/wgsd/marvin | mvn | github.com/... | Active |
```

---

## Implementation Steps

### Step 1: Create git-ops library (30 min)

Location: `workflows/lib/git-ops.md`

```markdown
---
name: wgsd:lib:git-ops
description: Git operations library with error handling
---

## git_clone

<operation>
<input>url, target_path</input>
<process>
git clone "$url" "$target_path" 2>&1
if [ $? -eq 0 ]; then
  echo "SUCCESS: Cloned to $target_path"
else
  echo "ERROR: Clone failed"
  exit 1
fi
</process>
</operation>

## git_detect_primary

<operation>
<process>
# Check for develop first, then main
if git rev-parse --verify develop >/dev/null 2>&1; then
  echo "develop"
elif git rev-parse --verify main >/dev/null 2>&1; then
  echo "main"
else
  echo "ERROR: No primary branch found"
  exit 1
fi
</process>
</operation>

## git_ensure_clean

<operation>
<process>
if [ -n "$(git status --porcelain)" ]; then
  echo "ERROR: Uncommitted changes present"
  git status --short
  exit 1
fi
echo "SUCCESS: Working directory clean"
</process>
</operation>

## git_status

<operation>
<process>
BRANCH=$(git branch --show-current)
DIRTY=$(git status --porcelain | wc -l)
if [ "$DIRTY" -gt 0 ]; then
  STATUS="dirty"
else
  STATUS="clean"
fi
echo "branch=$BRANCH status=$STATUS changes=$DIRTY"
</process>
</operation>

## git_worktree_add

<operation>
<input>path, branch</input>
<process>
git worktree add "$path" "$branch" 2>&1
if [ $? -eq 0 ]; then
  echo "SUCCESS: Worktree created at $path for $branch"
else
  echo "ERROR: Worktree creation failed"
  exit 1
fi
</process>
</operation>
```

### Step 2: Create init workflow (45 min)

Location: `workflows/init.md`

**Key additions to existing setup-repo.md:**
- Workspace directory creation outside repo
- Registry management
- Stub collection and validation
- Integration with git-ops library

### Step 3: Create workspace-status workflow (30 min)

Location: `workflows/workspace-status.md`

**Report format:**
```
🏠 WGSD Workspace Status: marvin

📁 Path: ~/.openclaw/workspace/wgsd/marvin
🌿 Branch: develop (clean)
🔄 Sync: Up to date with origin

📊 Focus Groups:
   • security (2 concepts)
   • onboarding (1 concept)

🚀 Implementations:
   • auth-v2 (active, 2 days)
   
⚠️ Issues: None
```

### Step 4: Update existing workflows (30 min)

Modify `setup-repo.md` to:
- Call new init workflow
- Use git-ops library
- Register workspace

### Step 5: Create WORKSPACES.md template (15 min)

Location: `.planning/WORKSPACES.md`

---

## Verification Checklist

### Functional Tests

- [ ] `wgsd init https://github.com/user/repo` creates workspace correctly
- [ ] `wgsd init /path/to/local/repo` works for local repositories
- [ ] `wgsd status` shows accurate workspace state
- [ ] Git operations handle errors gracefully
- [ ] Workspace registry tracks all projects

### Edge Cases

- [ ] Repository with no develop branch (uses main)
- [ ] Repository with uncommitted changes (blocks with warning)
- [ ] Repository already initialized (detects and offers resume)
- [ ] Network failure during clone (clean error message)
- [ ] Workspace directory already exists (offers overwrite/resume)

### Integration Tests

- [ ] Multiple workspaces can coexist
- [ ] Workspace status works from any subdirectory
- [ ] Git operations work across worktrees

---

## Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Workspace created | Directory exists with correct structure |
| Git state correct | Develop branch checked out, clean |
| Registry updated | WORKSPACES.md includes new project |
| Status accurate | Reports match actual git state |
| Errors handled | No uncaught failures, clear messages |

---

## Rollback Plan

If workspace creation fails mid-process:

1. Remove partially created workspace directory
2. Remove any created worktrees
3. Remove registry entry
4. Report clear error with failure point

---

*Plan created: 2026-02-22*
*Ready for execution*
