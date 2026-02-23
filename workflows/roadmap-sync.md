---
name: wgsd:roadmap-sync
description: Sync roadmap branch changes to develop
argument-hint: "[--dry-run] [--reverse]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
---

# Roadmap Sync

Synchronize approved concepts from roadmap branch to develop (or vice versa), keeping branches current with approved work.

---

## Objective

The roadmap branch contains approved concepts ready for implementation. This workflow:
1. Merges roadmap changes into develop (default)
2. Or syncs develop updates back to roadmap (--reverse)
3. Keeps both branches in sync without losing approved concept context

---

## When to Run

- **Before starting new feature work** - Ensure develop has latest approved concepts
- **After implementations complete** - Sync implementation results
- **Periodically** - Daily or on-demand to keep branches aligned
- **Before release** - Ensure all approved work is in develop

---

## Process

### Step 1: Check Branch State

```bash
DRY_RUN="${1:-}"
REVERSE="${2:-}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "🔄 Roadmap Sync"
echo "   Time: ${TIMESTAMP}"
echo ""

# Parse flags
if [ "$DRY_RUN" = "--reverse" ]; then
    REVERSE="--reverse"
    DRY_RUN=""
elif [ "$REVERSE" = "--dry-run" ]; then
    DRY_RUN="--dry-run"
    REVERSE=""
fi

# Detect primary branch
if git rev-parse --verify origin/develop >/dev/null 2>&1; then
    PRIMARY="develop"
elif git rev-parse --verify develop >/dev/null 2>&1; then
    PRIMARY="develop"
elif git rev-parse --verify origin/main >/dev/null 2>&1; then
    PRIMARY="main"
else
    PRIMARY="main"
fi

echo "   Primary Branch: ${PRIMARY}"

# Check roadmap exists
if ! git rev-parse --verify roadmap >/dev/null 2>&1; then
    if ! git rev-parse --verify origin/roadmap >/dev/null 2>&1; then
        echo "❌ Roadmap branch does not exist"
        echo "   Run: /wgsd init to create it"
        exit 1
    fi
fi

# Determine sync direction
if [ "$REVERSE" = "--reverse" ]; then
    SOURCE_BRANCH="$PRIMARY"
    TARGET_BRANCH="roadmap"
    echo "   Direction: ${PRIMARY} → roadmap (reverse sync)"
else
    SOURCE_BRANCH="roadmap"
    TARGET_BRANCH="$PRIMARY"
    echo "   Direction: roadmap → ${PRIMARY} (forward sync)"
fi

echo ""

# Fetch latest
echo "📥 Fetching latest from origin..."
git fetch origin 2>/dev/null || true
```

### Step 2: Analyze Differences

```bash
echo ""
echo "📊 Analyzing differences..."

# Get commit difference
SOURCE_AHEAD=$(git rev-list --count ${TARGET_BRANCH}..${SOURCE_BRANCH} 2>/dev/null || echo "0")
TARGET_AHEAD=$(git rev-list --count ${SOURCE_BRANCH}..${TARGET_BRANCH} 2>/dev/null || echo "0")

echo "   ${SOURCE_BRANCH} ahead of ${TARGET_BRANCH}: ${SOURCE_AHEAD} commits"
echo "   ${TARGET_BRANCH} ahead of ${SOURCE_BRANCH}: ${TARGET_AHEAD} commits"

if [ "$SOURCE_AHEAD" -eq 0 ]; then
    echo ""
    echo "✅ ${TARGET_BRANCH} is up-to-date with ${SOURCE_BRANCH}"
    echo "   No sync needed."
    exit 0
fi

# Show what will be merged
echo ""
echo "📋 Commits to merge (${SOURCE_BRANCH} → ${TARGET_BRANCH}):"
echo ""
git log --oneline ${TARGET_BRANCH}..${SOURCE_BRANCH} 2>/dev/null | head -15
if [ "$SOURCE_AHEAD" -gt 15 ]; then
    echo "   ... and $((SOURCE_AHEAD - 15)) more"
fi

# Show file changes summary
echo ""
echo "📁 Files changed:"
git diff --stat ${TARGET_BRANCH}..${SOURCE_BRANCH} 2>/dev/null | tail -10
```

### Step 3: Dry Run Check

```bash
if [ "$DRY_RUN" = "--dry-run" ]; then
    echo ""
    echo "═══════════════════════════════════════════════════════════"
    echo "🔍 DRY RUN - No changes made"
    echo "═══════════════════════════════════════════════════════════"
    echo ""
    echo "Would merge ${SOURCE_AHEAD} commit(s) from ${SOURCE_BRANCH} to ${TARGET_BRANCH}"
    echo ""
    echo "To perform sync:"
    if [ "$REVERSE" = "--reverse" ]; then
        echo "   /wgsd roadmap-sync --reverse"
    else
        echo "   /wgsd roadmap-sync"
    fi
    exit 0
fi
```

### Step 4: Confirm Sync

```bash
echo ""
echo "⚠️ About to merge ${SOURCE_AHEAD} commit(s)"
echo ""
echo "   From: ${SOURCE_BRANCH}"
echo "   To:   ${TARGET_BRANCH}"
echo ""

# In automated mode, proceed. In interactive, could prompt.
echo "Proceeding with sync..."
```

### Step 5: Perform Merge

```bash
echo ""
echo "🔀 Merging ${SOURCE_BRANCH} → ${TARGET_BRANCH}..."

# Store current branch
CURRENT_BRANCH=$(git branch --show-current)

# Stash any local changes
git stash push -m "WGSD: roadmap-sync stash" 2>/dev/null || true

# Checkout target branch
git checkout ${TARGET_BRANCH}
git pull origin ${TARGET_BRANCH} 2>/dev/null || true

# Perform merge
MERGE_MESSAGE="chore: sync ${SOURCE_BRANCH} to ${TARGET_BRANCH}

Roadmap Sync: ${TIMESTAMP}
Direction: ${SOURCE_BRANCH} → ${TARGET_BRANCH}
Commits merged: ${SOURCE_AHEAD}

This brings approved concepts from the ${SOURCE_BRANCH} branch into ${TARGET_BRANCH}.

Sync performed by: /wgsd roadmap-sync"

git merge ${SOURCE_BRANCH} --no-ff -m "$MERGE_MESSAGE"

MERGE_RESULT=$?

if [ $MERGE_RESULT -ne 0 ]; then
    echo ""
    echo "═══════════════════════════════════════════════════════════"
    echo "⚠️ MERGE CONFLICT"
    echo "═══════════════════════════════════════════════════════════"
    echo ""
    echo "Conflicting files:"
    git status --short | grep "^UU"
    echo ""
    echo "Options:"
    echo ""
    echo "  1. **Resolve conflicts manually:**"
    echo "     - Edit conflicting files"
    echo "     - git add <resolved-files>"
    echo "     - git commit"
    echo "     - git push origin ${TARGET_BRANCH}"
    echo ""
    echo "  2. **Abort and return to previous state:**"
    echo "     git merge --abort"
    echo "     git checkout ${CURRENT_BRANCH}"
    echo ""
    echo "  3. **Accept all changes from ${SOURCE_BRANCH}:**"
    echo "     git checkout --theirs ."
    echo "     git add ."
    echo "     git commit -m 'Resolve conflicts: accept ${SOURCE_BRANCH} changes'"
    echo ""
    exit 1
fi

echo "✅ Merge successful"
```

### Step 6: Push Changes

```bash
echo ""
echo "📤 Pushing to origin..."

git push origin ${TARGET_BRANCH}

if [ $? -ne 0 ]; then
    echo ""
    echo "⚠️ Push failed"
    echo ""
    echo "This might be due to:"
    echo "  - Remote has new commits (pull first)"
    echo "  - Permission issues"
    echo ""
    echo "Try: git pull origin ${TARGET_BRANCH} --rebase"
    echo "Then: git push origin ${TARGET_BRANCH}"
fi
```

### Step 7: Update Sync History (if syncing to develop)

```bash
if [ "$SOURCE_BRANCH" = "roadmap" ]; then
    echo ""
    echo "📝 Recording sync in roadmap manifest..."
    
    # Update manifest on roadmap branch
    git checkout roadmap 2>/dev/null
    
    if [ -f ".planning/ROADMAP-MANIFEST.md" ]; then
        # Add sync history entry
        echo "| ${TIMESTAMP%%T*} | Sync to ${TARGET_BRANCH} | ${SOURCE_AHEAD} commits |" >> .planning/ROADMAP-MANIFEST.md
        git add .planning/ROADMAP-MANIFEST.md
        git commit -m "docs: record sync to ${TARGET_BRANCH}" 2>/dev/null
        git push origin roadmap 2>/dev/null || true
    fi
fi
```

### Step 8: Return to Original Branch

```bash
# Return to original branch
git checkout "$CURRENT_BRANCH" 2>/dev/null || git checkout ${PRIMARY}
git stash pop 2>/dev/null || true
```

### Step 9: Report Success

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "✅ Roadmap Sync Complete"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📊 Summary:"
echo "   Direction: ${SOURCE_BRANCH} → ${TARGET_BRANCH}"
echo "   Commits merged: ${SOURCE_AHEAD}"
echo "   Time: ${TIMESTAMP}"
echo ""
echo "💡 What happened:"
if [ "$SOURCE_BRANCH" = "roadmap" ]; then
    echo "   - Approved concepts from roadmap are now in ${TARGET_BRANCH}"
    echo "   - Developers on ${TARGET_BRANCH} have access to all approved work"
    echo "   - Implementation branches can now include this content"
else
    echo "   - ${TARGET_BRANCH} branch now has latest develop updates"
    echo "   - Roadmap stays current with production changes"
    echo "   - New implementations will include these updates"
fi
echo ""
echo "🎯 Next steps:"
echo "   - Continue development on ${PRIMARY}"
echo "   - Create implementations from roadmap:"
echo "     /wgsd create-implementation <concept>"
echo "   - Run sync periodically to stay current"
```

---

## Variations

### Automatic Sync on Implementation Complete

When an implementation merges to develop, trigger roadmap sync:

```bash
# In implementation-complete workflow:
echo "🔄 Triggering reverse roadmap sync..."
./workflows/roadmap-sync.md --reverse
```

### Scheduled Sync

Can be added to HEARTBEAT.md or cron:

```bash
# Daily roadmap sync (check both directions)
wgsd roadmap-sync --dry-run       # Check forward sync
wgsd roadmap-sync                 # Execute if needed

wgsd roadmap-sync --reverse --dry-run  # Check reverse sync
wgsd roadmap-sync --reverse            # Execute if needed
```

### CI/CD Integration

```yaml
# GitHub Actions example
- name: Sync roadmap to develop
  run: |
    git fetch origin
    /wgsd roadmap-sync || echo "Sync may need manual intervention"
```

---

## Success Criteria

- [ ] Source branch changes merged to target branch
- [ ] Conflicts detected and reported clearly
- [ ] Sync history recorded in manifest
- [ ] No data loss on either branch
- [ ] Teams can continue development seamlessly
- [ ] Both branches remain in consistent state

---

## Error Recovery

| Error | Recovery |
|-------|----------|
| Merge conflict | Manual resolution or --theirs strategy |
| Push failed | Pull --rebase then push |
| Branch not found | Run /wgsd init |
| Permission denied | Check git credentials |

```bash
# If sync was interrupted
git merge --abort         # Abort incomplete merge
git checkout <branch>     # Return to safe branch
git stash pop            # Recover stashed changes
```

---

## Related Workflows

- `merge-to-roadmap.md` - Merge approved concepts to roadmap
- `create-implementation.md` - Create implementation from roadmap
- `init.md` - Initialize roadmap branch

---

*Workflow created for WGSD Phase 13 - Roadmap Branch Architecture*
