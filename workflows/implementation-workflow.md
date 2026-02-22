---
name: wgsd:implementation-workflow
description: Implementation track workflow for trunk-based code development
argument-hint: "[implementation-name] [action]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Manage the implementation track (Track 2) of WGSD two-track development:
- Code PRs go directly to develop branch (trunk-based)
- Traditional GSD workflow within implementation
- Phase-based execution with progress tracking
- Integration with status reporting and dashboard
- Clean completion with channel archival
</objective>

<libraries>
- workflows/lib/github-pr.md - PR creation and management
- workflows/lib/git-ops.md - Git operations
- workflows/lib/priority-queue.md - Queue management
- workflows/lib/canvas-api.md - Canvas updates
- workflows/lib/slack-api.md - Channel management
</libraries>

<process>

## Action: Start Implementation

When user starts working on an implementation:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
REPO_ROOT=$(git rev-parse --show-toplevel)
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "🚀 Starting implementation: ${IMPLEMENTATION_NAME}"
echo ""

# Verify implementation exists
IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
if [ ! -d "$IMPL_DIR" ]; then
    echo "❌ Implementation not found: ${IMPLEMENTATION_NAME}"
    echo ""
    echo "💡 Available implementations:"
    ls -1 .planning/active-implementations/ 2>/dev/null || echo "  (none)"
    exit 1
fi

# Get implementation details
STATE_FILE="${IMPL_DIR}/STATE.md"
OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: @*//')
STATUS=$(grep "^Status:" "$STATE_FILE" 2>/dev/null | sed 's/Status: //')
FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //')

echo "📍 Implementation: ${IMPLEMENTATION_NAME}"
echo "   Owner: @${OWNER}"
echo "   Focus Group: ${FOCUS_GROUP}"
echo "   Status: ${STATUS}"
echo ""

# Check if worktree exists
WORKTREE_PATH="${REPO_ROOT}/implementations/${IMPLEMENTATION_NAME}"
if [ ! -d "$WORKTREE_PATH" ]; then
    echo "📂 Creating worktree..."
    
    # Create implementation branch from develop
    git checkout develop 2>/dev/null
    git pull origin develop 2>/dev/null
    
    BRANCH_NAME="implementations/${IMPLEMENTATION_NAME}"
    git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
    
    # Create worktree
    mkdir -p "${REPO_ROOT}/implementations"
    git worktree add "$WORKTREE_PATH" "$BRANCH_NAME"
    
    echo "✅ Worktree created: $WORKTREE_PATH"
else
    echo "✅ Worktree exists: $WORKTREE_PATH"
fi

# Update status to implementing
if [ "$STATUS" = "planning" ] || [ "$STATUS" = "ready" ]; then
    sed -i "s/^Status:.*/Status: implementing/" "$STATE_FILE"
    sed -i "s/^Started:.*/Started: ${TIMESTAMP}/" "$STATE_FILE" || \
        echo "Started: ${TIMESTAMP}" >> "$STATE_FILE"
    echo ""
    echo "✅ Status updated to: implementing"
fi
```

**Output to user:**
```
✅ Implementation ${IMPLEMENTATION_NAME} is ready!

📁 Work in: implementations/${IMPLEMENTATION_NAME}/
📚 Requirements: .planning/active-implementations/${IMPLEMENTATION_NAME}/REQUIREMENTS.md
📍 Roadmap: .planning/active-implementations/${IMPLEMENTATION_NAME}/ROADMAP.md

💡 Development workflow:
1. Work on code in the worktree
2. Commit regularly with descriptive messages  
3. Push when ready: git push origin implementations/${IMPLEMENTATION_NAME}
4. Create PR when phase is complete: /wgsd implementation pr ${IMPLEMENTATION_NAME}
```

---

## Action: Create Code PR

When user is ready to merge a phase:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
PHASE_NUMBER="{phase-number:-current}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📋 Creating code PR for: ${IMPLEMENTATION_NAME}"

# Get implementation details
IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"
OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: @*//')
FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //')

# Detect current phase if not specified
if [ "$PHASE_NUMBER" = "current" ]; then
    ROADMAP_FILE="${IMPL_DIR}/ROADMAP.md"
    PHASE_NUMBER=$(grep -E "^\| \d+ \|.*In Progress" "$ROADMAP_FILE" 2>/dev/null | \
        head -1 | awk -F'|' '{print $2}' | tr -d ' ')
    PHASE_NUMBER="${PHASE_NUMBER:-1}"
fi

# Get phase name from roadmap
PHASE_NAME=$(grep -E "^## Phase ${PHASE_NUMBER}:" "${IMPL_DIR}/ROADMAP.md" 2>/dev/null | \
    sed "s/## Phase ${PHASE_NUMBER}: //")
PHASE_NAME="${PHASE_NAME:-Phase ${PHASE_NUMBER}}"

# Prepare PR details
BRANCH_NAME="implementations/${IMPLEMENTATION_NAME}"
BASE_BRANCH="develop"
PR_TITLE="feat(${FOCUS_GROUP}): ${IMPLEMENTATION_NAME} - ${PHASE_NAME}"
PR_BODY="## Implementation: ${IMPLEMENTATION_NAME}

**Phase ${PHASE_NUMBER}:** ${PHASE_NAME}
**Focus Group:** ${FOCUS_GROUP}
**Owner:** @${OWNER}

### Changes
<!-- Describe the changes in this phase -->

### Testing
- [ ] Unit tests pass
- [ ] Integration tests pass  
- [ ] Manual testing complete

### Checklist
- [ ] Code follows project conventions
- [ ] Documentation updated if needed
- [ ] No breaking changes (or documented)

---
*Created via WGSD implementation workflow*"

echo ""
echo "📝 PR Details:"
echo "   Branch: ${BRANCH_NAME} → ${BASE_BRANCH}"
echo "   Title: ${PR_TITLE}"
echo ""

# Push branch if not pushed
git push origin "$BRANCH_NAME" 2>/dev/null

# Create PR using GitHub CLI
if command -v gh &> /dev/null; then
    PR_URL=$(gh pr create \
        --base "$BASE_BRANCH" \
        --head "$BRANCH_NAME" \
        --title "$PR_TITLE" \
        --body "$PR_BODY" \
        --label "implementation,${FOCUS_GROUP}" \
        2>&1)
    
    if [ $? -eq 0 ]; then
        echo "✅ PR created: $PR_URL"
        
        # Record PR in state
        echo "PR: $PR_URL" >> "$STATE_FILE"
        echo "PR Created: ${TIMESTAMP}" >> "$STATE_FILE"
    else
        echo "⚠️ PR creation output: $PR_URL"
    fi
else
    echo "⚠️ GitHub CLI (gh) not installed"
    echo ""
    echo "Create PR manually:"
    echo "  Base: ${BASE_BRANCH}"
    echo "  Head: ${BRANCH_NAME}"
    echo "  Title: ${PR_TITLE}"
fi
```

---

## Action: Merge and Advance Phase

After PR is approved and merged:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
PHASE_NUMBER="{phase-number}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "✅ Advancing to next phase: ${IMPLEMENTATION_NAME}"

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"
ROADMAP_FILE="${IMPL_DIR}/ROADMAP.md"

# Update roadmap - mark current phase complete
sed -i "s/| ${PHASE_NUMBER} |.*In Progress.*/| ${PHASE_NUMBER} | ✅ Complete | ${TIMESTAMP} |/" "$ROADMAP_FILE"

# Determine next phase
NEXT_PHASE=$((PHASE_NUMBER + 1))
TOTAL_PHASES=$(grep -c "^## Phase" "$ROADMAP_FILE" 2>/dev/null || echo "3")

if [ "$NEXT_PHASE" -le "$TOTAL_PHASES" ]; then
    # Mark next phase as in progress
    sed -i "s/| ${NEXT_PHASE} |.*Not Started.*/| ${NEXT_PHASE} | 🔄 In Progress | ${TIMESTAMP} |/" "$ROADMAP_FILE"
    
    # Update state
    sed -i "s/^Current Phase:.*/Current Phase: ${NEXT_PHASE}/" "$STATE_FILE"
    
    echo ""
    echo "📍 Now in Phase ${NEXT_PHASE}"
    echo ""
    echo "💡 Next steps:"
    echo "   1. Review Phase ${NEXT_PHASE} requirements"
    echo "   2. Continue development in worktree"
    echo "   3. Create PR when ready"
else
    echo ""
    echo "🎉 All phases complete!"
    echo ""
    echo "💡 Ready to complete implementation:"
    echo "   /wgsd complete implementation ${IMPLEMENTATION_NAME}"
fi

# Pull latest develop into branch
WORKTREE_PATH="implementations/${IMPLEMENTATION_NAME}"
if [ -d "$WORKTREE_PATH" ]; then
    cd "$WORKTREE_PATH"
    git fetch origin develop
    git merge origin/develop --no-edit
    echo ""
    echo "✅ Branch synced with develop"
fi
```

---

## Action: Complete Implementation

When all phases are done:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "🎉 Completing implementation: ${IMPLEMENTATION_NAME}"

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"
STUB=$(grep "^Stub:" "$STATE_FILE" 2>/dev/null | sed 's/Stub: //')
CONCEPT_NAME=$(grep "^Source Concept:" "$STATE_FILE" 2>/dev/null | sed 's/Source Concept: //')
FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //')

# Update implementation state
sed -i "s/^Status:.*/Status: ✅ Complete/" "$STATE_FILE"
echo "Completed: ${TIMESTAMP}" >> "$STATE_FILE"

# Move to completed implementations
COMPLETED_DIR=".planning/completed-implementations"
mkdir -p "$COMPLETED_DIR"
mv "$IMPL_DIR" "${COMPLETED_DIR}/${IMPLEMENTATION_NAME}"

echo "✅ Implementation archived to: ${COMPLETED_DIR}/${IMPLEMENTATION_NAME}"

# Update source concept
if [ -n "$CONCEPT_NAME" ] && [ -n "$FOCUS_GROUP" ]; then
    CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    if [ -f "$CONCEPT_FILE" ]; then
        sed -i "s/^Status:.*/Status: ✔️ Implemented/" "$CONCEPT_FILE"
        echo "Implementation Completed: ${TIMESTAMP}" >> "$CONCEPT_FILE"
        echo ""
        echo "✅ Source concept marked as implemented"
    fi
fi

# Clean up worktree
WORKTREE_PATH="implementations/${IMPLEMENTATION_NAME}"
if [ -d "$WORKTREE_PATH" ]; then
    git worktree remove "$WORKTREE_PATH" --force
    echo "✅ Worktree removed"
fi

# Delete implementation branch (already merged)
BRANCH_NAME="implementations/${IMPLEMENTATION_NAME}"
git branch -d "$BRANCH_NAME" 2>/dev/null
git push origin --delete "$BRANCH_NAME" 2>/dev/null
echo "✅ Branch cleaned up"

# Archive Slack channel
CHANNEL_NAME="${STUB}-impl-${IMPLEMENTATION_NAME}"
echo ""
echo "📢 Archive implementation channel:"
echo "   /wgsd archive channel ${CHANNEL_NAME}"

# Update priority queue
echo ""
echo "📋 Updating priority queue..."
# Remove from queue (already done since it was active)

# Trigger canvas update
echo ""
echo "🎨 Trigger canvas sync to update dashboards:"
echo "   /wgsd sync canvas all"

echo ""
echo "🎉 Implementation ${IMPLEMENTATION_NAME} completed!"
echo ""
echo "Summary:"
echo "  • All phases merged to develop"
echo "  • Concept marked as implemented"
echo "  • Implementation archived"
echo "  • Channel ready to archive"
```

---

## Action: Update Progress

For regular progress updates:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
PROGRESS="{progress-percent}"
MESSAGE="{status-message}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📊 Updating progress: ${IMPLEMENTATION_NAME}"

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"

# Update progress in state file
if grep -q "^Progress:" "$STATE_FILE"; then
    sed -i "s/^Progress:.*/Progress: ${PROGRESS}%/" "$STATE_FILE"
else
    echo "Progress: ${PROGRESS}%" >> "$STATE_FILE"
fi

# Update last activity
sed -i "s/^Last Update:.*/Last Update: ${TIMESTAMP}/" "$STATE_FILE" || \
    echo "Last Update: ${TIMESTAMP}" >> "$STATE_FILE"

# Add to progress log
PROGRESS_LOG="${IMPL_DIR}/PROGRESS.md"
if [ ! -f "$PROGRESS_LOG" ]; then
    echo "# Progress Log: ${IMPLEMENTATION_NAME}" > "$PROGRESS_LOG"
    echo "" >> "$PROGRESS_LOG"
fi

echo "## ${TIMESTAMP}" >> "$PROGRESS_LOG"
echo "" >> "$PROGRESS_LOG"
echo "**Progress:** ${PROGRESS}%" >> "$PROGRESS_LOG"
echo "" >> "$PROGRESS_LOG"
echo "${MESSAGE}" >> "$PROGRESS_LOG"
echo "" >> "$PROGRESS_LOG"
echo "---" >> "$PROGRESS_LOG"
echo "" >> "$PROGRESS_LOG"

echo "✅ Progress updated to ${PROGRESS}%"
echo ""
echo "💡 This will be reflected in the next canvas sync"
```

---

## Action: Report Blocker

When implementation is blocked:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
BLOCKER_DESC="{blocker-description}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "⚠️ Recording blocker: ${IMPLEMENTATION_NAME}"

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"
STUB=$(grep "^Stub:" "$STATE_FILE" 2>/dev/null | sed 's/Stub: //')
CHANNEL="${STUB}-impl-${IMPLEMENTATION_NAME}"

# Update status to blocked
sed -i "s/^Status:.*/Status: 🚫 Blocked/" "$STATE_FILE"

# Add blocker to state
echo "" >> "$STATE_FILE"
echo "## Blocker (${TIMESTAMP})" >> "$STATE_FILE"
echo "" >> "$STATE_FILE"
echo "${BLOCKER_DESC}" >> "$STATE_FILE"
echo "" >> "$STATE_FILE"

# Add to blockers file
BLOCKERS_FILE="${IMPL_DIR}/BLOCKERS.md"
if [ ! -f "$BLOCKERS_FILE" ]; then
    echo "# Blockers: ${IMPLEMENTATION_NAME}" > "$BLOCKERS_FILE"
    echo "" >> "$BLOCKERS_FILE"
fi

echo "## ${TIMESTAMP}" >> "$BLOCKERS_FILE"
echo "" >> "$BLOCKERS_FILE"
echo "**Status:** 🔴 Active" >> "$BLOCKERS_FILE"
echo "" >> "$BLOCKERS_FILE"
echo "${BLOCKER_DESC}" >> "$BLOCKERS_FILE"
echo "" >> "$BLOCKERS_FILE"
echo "---" >> "$BLOCKERS_FILE"
echo "" >> "$BLOCKERS_FILE"

echo "✅ Blocker recorded"
echo ""
echo "📢 Notifying team in channel #${CHANNEL}..."
```

**Notify team:**
Post to implementation channel with blocker details and request for help.

---

## Action: Resolve Blocker

When blocker is resolved:

```bash
IMPLEMENTATION_NAME="{implementation-name}"
RESOLUTION="{resolution-description}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "✅ Resolving blocker: ${IMPLEMENTATION_NAME}"

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
STATE_FILE="${IMPL_DIR}/STATE.md"
BLOCKERS_FILE="${IMPL_DIR}/BLOCKERS.md"

# Update status back to implementing
sed -i "s/^Status:.*/Status: implementing/" "$STATE_FILE"

# Mark latest blocker as resolved
if [ -f "$BLOCKERS_FILE" ]; then
    # Add resolution to most recent blocker
    sed -i "0,/Status: 🔴 Active/s//Status: ✅ Resolved (${TIMESTAMP})/" "$BLOCKERS_FILE"
    echo "" >> "$BLOCKERS_FILE"
    echo "**Resolution:** ${RESOLUTION}" >> "$BLOCKERS_FILE"
    echo "" >> "$BLOCKERS_FILE"
fi

echo "✅ Blocker resolved"
echo ""
echo "💡 Implementation is unblocked and can continue"
```

</process>

<routing>
| Action | Description |
|--------|-------------|
| start | Begin working on implementation, create worktree |
| pr, create-pr | Create code PR for current phase |
| merge, advance | Record phase completion, advance to next |
| complete | Finish implementation, archive and cleanup |
| update, progress | Update progress percentage |
| blocker, block | Report a blocking issue |
| unblock, resolve | Resolve a blocker |
</routing>

<success_criteria>
- Code PRs created correctly targeting develop branch
- Phase completion properly recorded in roadmap
- Blockers tracked and escalated appropriately
- Implementation cleanup removes worktree and branch
- Canvas dashboards updated after state changes
</success_criteria>
