---
name: wgsd:workspace-status
description: Report health and status of WGSD workspace
allowed-tools:
  - Read
  - Bash
---

# WGSD Workspace Status

Display comprehensive status of a WGSD workspace including git state, focus groups, and implementations.

---

## Objective

Provide a clear overview of:
- Current git branch and clean/dirty state
- Sync state with remote (ahead/behind)
- Active focus groups and their status
- Active implementations and their status
- Any blocking issues

---

## Inputs

| Input | Required | Description | Example |
|-------|----------|-------------|---------|
| workspace | No | Workspace name or path | `marvin` or auto-detect |

---

## Process

### Step 1: Detect Workspace

```bash
WGSD_ROOT="$HOME/.openclaw/workspace/wgsd"

# If workspace specified, use it
if [ -n "$WORKSPACE" ]; then
  if [ -d "$WGSD_ROOT/$WORKSPACE" ]; then
    WORKSPACE_PATH="$WGSD_ROOT/$WORKSPACE"
  elif [ -d "$WORKSPACE" ]; then
    WORKSPACE_PATH="$WORKSPACE"
  else
    echo "❌ Workspace not found: $WORKSPACE"
    exit 1
  fi
else
  # Auto-detect from current directory
  CURRENT_DIR=$(pwd)
  
  # Check if we're in a WGSD workspace
  if [[ "$CURRENT_DIR" == *"/wgsd/"* ]]; then
    # Extract workspace path
    WORKSPACE_PATH=$(echo "$CURRENT_DIR" | sed 's|\(/wgsd/[^/]*\).*|\1|')
  elif [ -d ".planning/WGSD-CONFIG.md" ] || [ -f ".planning/WGSD-CONFIG.md" ]; then
    # We're in a repo that's part of a workspace
    WORKSPACE_PATH=$(grep "^Workspace:" .planning/WGSD-CONFIG.md | awk '{print $2}')
  else
    echo "❌ Not in a WGSD workspace"
    echo "   Run from a workspace or specify: wgsd status <workspace>"
    exit 1
  fi
fi

PROJECT_NAME=$(basename "$WORKSPACE_PATH")
REPO_PATH="$WORKSPACE_PATH/repo"

if [ ! -d "$REPO_PATH" ]; then
  # Try if workspace IS the repo
  if [ -d "$WORKSPACE_PATH/.git" ]; then
    REPO_PATH="$WORKSPACE_PATH"
  else
    echo "❌ Repository not found in workspace"
    exit 1
  fi
fi

cd "$REPO_PATH"
```

### Step 2: Load Configuration

```bash
CONFIG_FILE="$REPO_PATH/.planning/WGSD-CONFIG.md"

if [ -f "$CONFIG_FILE" ]; then
  SLACK_STUB=$(grep "^\*\*Stub:\*\*" "$CONFIG_FILE" | awk '{print $2}')
else
  SLACK_STUB="(not configured)"
fi
```

### Step 3: Get Git Status

```bash
# Current branch
BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")

# Dirty state
DIRTY_COUNT=$(git status --porcelain 2>/dev/null | wc -l)
if [ "$DIRTY_COUNT" -gt 0 ]; then
  GIT_STATUS="dirty ($DIRTY_COUNT files)"
  GIT_ICON="⚠️"
else
  GIT_STATUS="clean"
  GIT_ICON="✅"
fi

# Sync status with remote
git fetch origin --quiet 2>/dev/null

REMOTE_BRANCH="origin/$BRANCH"
if git rev-parse --verify "$REMOTE_BRANCH" >/dev/null 2>&1; then
  AHEAD=$(git rev-list --count "$REMOTE_BRANCH..$BRANCH" 2>/dev/null || echo "0")
  BEHIND=$(git rev-list --count "$BRANCH..$REMOTE_BRANCH" 2>/dev/null || echo "0")
  
  if [ "$AHEAD" -gt 0 ] && [ "$BEHIND" -gt 0 ]; then
    SYNC_STATUS="diverged (+$AHEAD/-$BEHIND)"
    SYNC_ICON="🔀"
  elif [ "$AHEAD" -gt 0 ]; then
    SYNC_STATUS="ahead by $AHEAD commits"
    SYNC_ICON="📤"
  elif [ "$BEHIND" -gt 0 ]; then
    SYNC_STATUS="behind by $BEHIND commits"
    SYNC_ICON="📥"
  else
    SYNC_STATUS="up to date"
    SYNC_ICON="✅"
  fi
else
  SYNC_STATUS="no remote tracking"
  SYNC_ICON="❓"
fi
```

### Step 4: List Focus Groups

```bash
# Find focus group branches
echo "FOCUS_GROUPS:" > /tmp/wgsd_status_fg.txt
git branch -a 2>/dev/null | grep "focus-groups/" | sed 's|.*focus-groups/||' | sort -u | while read fg; do
  # Check if there's an active worktree
  FG_PATH="$WORKSPACE_PATH/concepts/$fg"
  if [ -d "$FG_PATH" ]; then
    FG_STATUS="active"
  else
    FG_STATUS="branch-only"
  fi
  echo "  - $fg ($FG_STATUS)" >> /tmp/wgsd_status_fg.txt
done

FOCUS_GROUP_COUNT=$(git branch -a 2>/dev/null | grep "focus-groups/" | wc -l)
```

### Step 5: List Implementations

```bash
# Find implementation branches
echo "IMPLEMENTATIONS:" > /tmp/wgsd_status_impl.txt
git branch -a 2>/dev/null | grep "implementations/" | sed 's|.*implementations/||' | sort -u | while read impl; do
  # Check if there's an active worktree
  IMPL_PATH="$WORKSPACE_PATH/implementations/$impl"
  if [ -d "$IMPL_PATH" ]; then
    IMPL_STATUS="active"
    # Check age
    if [ -d "$IMPL_PATH/.git" ]; then
      IMPL_CREATED=$(stat -c %Y "$IMPL_PATH/.git" 2>/dev/null || stat -f %m "$IMPL_PATH/.git" 2>/dev/null)
      NOW=$(date +%s)
      AGE_DAYS=$(( (NOW - IMPL_CREATED) / 86400 ))
      if [ "$AGE_DAYS" -gt 3 ]; then
        IMPL_STATUS="active ⚠️ $AGE_DAYS days old"
      else
        IMPL_STATUS="active ($AGE_DAYS days)"
      fi
    fi
  else
    IMPL_STATUS="branch-only"
  fi
  echo "  - $impl ($IMPL_STATUS)" >> /tmp/wgsd_status_impl.txt
done

IMPL_COUNT=$(git branch -a 2>/dev/null | grep "implementations/" | wc -l)
```

### Step 6: Check for Issues

```bash
ISSUES=""

# Check for stale implementations (>3 days)
STALE_IMPL=$(find "$WORKSPACE_PATH/implementations" -maxdepth 1 -type d -mtime +3 2>/dev/null | wc -l)
if [ "$STALE_IMPL" -gt 0 ]; then
  ISSUES="$ISSUES\n⚠️  $STALE_IMPL implementation(s) older than 3 days"
fi

# Check for dirty state
if [ "$DIRTY_COUNT" -gt 0 ]; then
  ISSUES="$ISSUES\n⚠️  Uncommitted changes in working directory"
fi

# Check for diverged branches
if [[ "$SYNC_STATUS" == *"diverged"* ]]; then
  ISSUES="$ISSUES\n⚠️  Branch has diverged from remote"
fi

# Check for too many implementations
if [ "$IMPL_COUNT" -gt 4 ]; then
  ISSUES="$ISSUES\n⚠️  Too many active implementations ($IMPL_COUNT > 4 recommended)"
fi
```

### Step 7: Display Status

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🏠 WGSD Workspace Status: $PROJECT_NAME"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Path: $WORKSPACE_PATH"
echo "📱 Slack Stub: $SLACK_STUB"
echo ""
echo "───────────────────────────────────────────────────────────"
echo "📊 Git Status"
echo "───────────────────────────────────────────────────────────"
echo "🌿 Branch: $BRANCH ($GIT_ICON $GIT_STATUS)"
echo "🔄 Sync: $SYNC_ICON $SYNC_STATUS"
echo ""

if [ "$FOCUS_GROUP_COUNT" -gt 0 ]; then
  echo "───────────────────────────────────────────────────────────"
  echo "📂 Focus Groups ($FOCUS_GROUP_COUNT)"
  echo "───────────────────────────────────────────────────────────"
  cat /tmp/wgsd_status_fg.txt | tail -n +2
  echo ""
fi

if [ "$IMPL_COUNT" -gt 0 ]; then
  echo "───────────────────────────────────────────────────────────"
  echo "🚀 Implementations ($IMPL_COUNT)"
  echo "───────────────────────────────────────────────────────────"
  cat /tmp/wgsd_status_impl.txt | tail -n +2
  echo ""
fi

if [ -n "$ISSUES" ]; then
  echo "───────────────────────────────────────────────────────────"
  echo "⚠️  Issues"
  echo "───────────────────────────────────────────────────────────"
  echo -e "$ISSUES"
  echo ""
else
  echo "───────────────────────────────────────────────────────────"
  echo "✅ No Issues"
  echo "───────────────────────────────────────────────────────────"
  echo ""
fi

# Cleanup temp files
rm -f /tmp/wgsd_status_fg.txt /tmp/wgsd_status_impl.txt
```

---

## Output Format

```
═══════════════════════════════════════════════════════════
🏠 WGSD Workspace Status: marvin
═══════════════════════════════════════════════════════════

📁 Path: ~/.openclaw/workspace/wgsd/marvin
📱 Slack Stub: mvn

───────────────────────────────────────────────────────────
📊 Git Status
───────────────────────────────────────────────────────────
🌿 Branch: develop (✅ clean)
🔄 Sync: ✅ up to date

───────────────────────────────────────────────────────────
📂 Focus Groups (2)
───────────────────────────────────────────────────────────
  - security (active)
  - onboarding (branch-only)

───────────────────────────────────────────────────────────
🚀 Implementations (1)
───────────────────────────────────────────────────────────
  - auth-v2 (active, 2 days)

───────────────────────────────────────────────────────────
✅ No Issues
───────────────────────────────────────────────────────────
```

---

## Success Criteria

- [ ] Workspace detected from path or auto-detected
- [ ] Git branch and clean state displayed
- [ ] Remote sync status shown
- [ ] Focus groups listed with worktree status
- [ ] Implementations listed with age
- [ ] Blocking issues highlighted

---

*Workflow created for WGSD Phase 1*
