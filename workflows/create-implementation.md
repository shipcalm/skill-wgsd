---
name: wgsd:create-implementation
description: Create implementation from a promoted concept with scaling enforcement
argument-hint: "[concept-name]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Create active implementation from a promoted concept with:
- Dedicated implementation channel for focused execution
- Git branch and worktree for code development (trunk-based)
- Focus group scaling enforcement (~1 impl per focus group)
- Owner verification and commitment
- Priority queue management
- Traditional GSD workflow (requirements → roadmap → phases)
- Implementation tracking and status reporting
- Global 2-4 concurrent implementation limit
</objective>

<libraries>
- workflows/lib/priority-queue.md - Queue management
- workflows/lib/github-pr.md - PR integration
- workflows/lib/slack-api.md - Channel creation
- workflows/lib/canvas-api.md - Canvas creation
</libraries>

<process>

## Step 1: Validate Global Implementation Capacity

```bash
REPO_NAME=$(basename $(pwd))
CONCEPT_NAME="{concept-name}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📊 Checking implementation capacity..."

# Check current implementation count
CURRENT_IMPLEMENTATIONS=0
if [ -d .planning/active-implementations ]; then
    CURRENT_IMPLEMENTATIONS=$(find .planning/active-implementations -maxdepth 1 -type d \
        | grep -v "^\\.planning/active-implementations$" | wc -l)
fi

MAX_IMPLEMENTATIONS=4

if [ "$CURRENT_IMPLEMENTATIONS" -ge "$MAX_IMPLEMENTATIONS" ]; then
    echo ""
    echo "❌ **Implementation capacity reached** (${CURRENT_IMPLEMENTATIONS}/${MAX_IMPLEMENTATIONS})"
    echo ""
    echo "Active implementations:"
    for impl_dir in .planning/active-implementations/*/; do
        IMPL_NAME=$(basename "$impl_dir")
        OWNER=$(grep "Owner:" "${impl_dir}STATE.md" 2>/dev/null | head -1 | sed 's/.*: @*//')
        STATUS=$(grep "Status:" "${impl_dir}STATE.md" 2>/dev/null | head -1 | sed 's/.*: //')
        echo "  • ${IMPL_NAME} (@${OWNER:-unassigned}) - ${STATUS:-unknown}"
    done
    echo ""
    echo "💡 Complete or pause an implementation before starting a new one"
    exit 1
fi

echo "✅ Global capacity OK: ${CURRENT_IMPLEMENTATIONS}/${MAX_IMPLEMENTATIONS} slots used"
```

## Step 2: Find and Validate Concept

```bash
echo ""
echo "🔍 Locating concept: ${CONCEPT_NAME}"

# Search for concept across all focus groups
CONCEPT_FILE=""
FOCUS_GROUP=""
OWNER=""

for fg_dir in .planning/focus-groups/*/; do
    if [ -f "${fg_dir}concepts/${CONCEPT_NAME}.md" ]; then
        CONCEPT_FILE="${fg_dir}concepts/${CONCEPT_NAME}.md"
        FOCUS_GROUP=$(basename "$fg_dir")
        break
    fi
done

if [ -z "$CONCEPT_FILE" ]; then
    echo "❌ Concept '${CONCEPT_NAME}' not found"
    echo ""
    echo "💡 Available concepts ready for implementation:"
    find .planning/focus-groups -name "*.md" -path "*/concepts/*" \
        -exec grep -l "🚀 Ready" {} \; 2>/dev/null | \
        sed 's|.planning/focus-groups/||;s|/concepts/| → |;s|\.md||'
    exit 1
fi

echo "📍 Found: ${CONCEPT_NAME} in ${FOCUS_GROUP} focus group"

# Check status
CURRENT_STATUS=$(grep "^\*\*Status:\*\*" "$CONCEPT_FILE" | head -1 | sed 's/\*\*Status:\*\* //')

if [[ "$CURRENT_STATUS" != *"Ready"* ]]; then
    echo ""
    echo "❌ Concept is not ready for implementation"
    echo "   Current status: ${CURRENT_STATUS}"
    echo ""
    echo "💡 Promote concept first: /wgsd promote ${CONCEPT_NAME}"
    exit 1
fi

echo "✅ Concept status: Ready for Implementation"
```

## Step 3: Verify Owner Assignment

```bash
echo ""
echo "👤 Verifying owner assignment..."

# Extract owner from concept file
OWNER=$(grep "Assigned Owner:" "$CONCEPT_FILE" 2>/dev/null | sed 's/.*: @*//' | head -1)

if [ -z "$OWNER" ]; then
    echo ""
    echo "❌ **No owner assigned**"
    echo ""
    echo "An implementation requires an assigned owner who commits to:"
    echo "  • 1-3 day implementation timeline"
    echo "  • Daily status updates"
    echo "  • Driving the work to completion"
    echo ""
    echo "**To assign an owner:**"
    echo "  /wgsd assign ${CONCEPT_NAME} @user"
    echo ""
    echo "**To claim it yourself:**"
    echo "  /wgsd claim ${CONCEPT_NAME}"
    exit 1
fi

echo "✅ Owner: @${OWNER}"
```

## Step 4: Check Focus Group Scaling (WORKFLOW-05)

```bash
echo ""
echo "📊 Checking focus group scaling..."

# Count active implementations for this specific focus group
FG_IMPL_COUNT=0
FG_ACTIVE_IMPLS=""

if [ -d .planning/active-implementations ]; then
    for impl_dir in .planning/active-implementations/*/; do
        if [ -f "${impl_dir}concept-source.md" ]; then
            IMPL_FG=$(grep "Source Focus Group:" "${impl_dir}concept-source.md" 2>/dev/null | sed 's/.*: //')
            if [ "$IMPL_FG" = "$FOCUS_GROUP" ]; then
                FG_IMPL_COUNT=$((FG_IMPL_COUNT + 1))
                IMPL_NAME=$(basename "$impl_dir")
                FG_ACTIVE_IMPLS="${FG_ACTIVE_IMPLS}  • ${IMPL_NAME}\n"
            fi
        fi
    done
fi

echo "   ${FOCUS_GROUP} focus group: ${FG_IMPL_COUNT} active implementation(s)"

if [ "$FG_IMPL_COUNT" -ge 1 ]; then
    echo ""
    echo "⚠️ **Scaling Warning**"
    echo ""
    echo "The ${FOCUS_GROUP} focus group already has an active implementation:"
    echo -e "${FG_ACTIVE_IMPLS}"
    echo ""
    echo "WGSD recommends ~1 implementation per focus group to:"
    echo "  • Maintain domain expertise focus"
    echo "  • Prevent context switching within teams"
    echo "  • Enable natural parallel scaling across focus groups"
    echo ""
    
    # Check for override flag or prompt
    if [ -z "$OVERRIDE_REASON" ]; then
        echo "**Options:**"
        echo "1. Wait for current implementation to complete"
        echo "2. Override with justification:"
        echo "   /wgsd create-implementation ${CONCEPT_NAME} --override \"<reason>\""
        echo ""
        echo "Override with a valid reason to proceed."
        exit 1
    else
        echo "⚠️ Override accepted"
        echo "   Reason: ${OVERRIDE_REASON}"
        echo ""
        
        # Log the override decision
        cat >> ".planning/focus-groups/${FOCUS_GROUP}/STATE.md" << EOF

### Scaling Override: ${TIMESTAMP}
- **Implementation:** ${CONCEPT_NAME}
- **Reason:** ${OVERRIDE_REASON}
- **Existing Active:** $(echo -e "$FG_ACTIVE_IMPLS" | tr '\n' ', ')
EOF
        
        echo "📝 Override logged to focus group STATE.md"
    fi
else
    echo "✅ Scaling check passed"
fi
```

## Step 5: Create Implementation Structure

```bash
echo ""
echo "🏗️ Creating implementation structure..."

IMPL_NAME="${CONCEPT_NAME}-impl"
IMPL_DIR=".planning/active-implementations/${IMPL_NAME}"

mkdir -p "${IMPL_DIR}"

echo "   Directory: ${IMPL_DIR}"
```

## Step 6: Create Implementation Files

**.planning/active-implementations/{impl-name}/concept-source.md:**
```bash
# Extract concept details
PROBLEM=$(grep -A5 "## Problem Statement" "$CONCEPT_FILE" | grep -v "^##" | head -3)
PRIORITY=$(grep "Priority:" "$CONCEPT_FILE" | head -1 | sed 's/.*: //')

cat > "${IMPL_DIR}/concept-source.md" << EOF
# Implementation Source

**Implementation:** ${IMPL_NAME}
**Source Concept:** ${CONCEPT_NAME}
**Source Focus Group:** ${FOCUS_GROUP}
**Promoted:** ${TIMESTAMP}
**Owner:** @${OWNER}

## Original Concept

**File:** \`.planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md\`
**Priority:** ${PRIORITY:-medium}

## Problem Statement

${PROBLEM}

## Implementation Commitment

**Owner:** @${OWNER}
**Target Timeline:** 1-3 days maximum
**Started:** ${TIMESTAMP}

## Links

- **Source Concept:** \`.planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md\`
- **Focus Group Channel:** #${REPO_NAME}-${FOCUS_GROUP}
- **Implementation Channel:** #${REPO_NAME}-impl-${CONCEPT_NAME}

---

*This implementation tracks back to the ${FOCUS_GROUP} focus group discussion*
EOF

echo "   Created: concept-source.md"
```

**.planning/active-implementations/{impl-name}/STATE.md:**
```bash
cat > "${IMPL_DIR}/STATE.md" << EOF
# ${CONCEPT_NAME} Implementation - State

**Created:** ${TIMESTAMP}
**Source Concept:** ${CONCEPT_NAME} from ${FOCUS_GROUP}
**Owner:** @${OWNER}
**Status:** 🟡 Initializing
**Progress:** 0%
**Days Active:** 0
**Target Version:** TBD (determined at merge)

## Current Phase

**Phase:** 0 - Requirements Definition
**Progress:** Starting
**Next:** Define implementation requirements from concept

## Daily Log

### Day 0 (${TIMESTAMP:0:10})
- Implementation created from concept
- Channel and infrastructure setup
- Ready to begin requirements definition

## Blockers

*None currently*

## Milestones

- [ ] Requirements defined
- [ ] Roadmap created
- [ ] Phase 1 complete
- [ ] Code review passed
- [ ] Merged to develop

---

**Daily Updates Required:** Update this file daily with progress
**Canvas Sync:** Changes sync to implementation channel canvas
EOF

echo "   Created: STATE.md"
```

## Step 7: Verify Concept is on Roadmap (Phase 13)

```bash
echo ""
echo "📋 Verifying concept is on roadmap..."

# Phase 13: Implementations must branch from roadmap
# First, ensure roadmap branch exists
if ! git rev-parse --verify roadmap >/dev/null 2>&1; then
    if git rev-parse --verify origin/roadmap >/dev/null 2>&1; then
        git checkout -b roadmap origin/roadmap 2>/dev/null
        git checkout "$(git branch --show-current)" 2>/dev/null
        echo "   Fetched roadmap branch from remote"
    else
        echo "⚠️ Roadmap branch does not exist"
        echo "   Run '/wgsd init' to create it, or use --from-develop for emergency bypass"
        
        if [ -z "$BYPASS_ROADMAP" ]; then
            echo ""
            echo "❌ Cannot create implementation without roadmap branch"
            echo "   Concepts must be fully approved and merged to roadmap first."
            exit 1
        fi
    fi
fi

# Check if concept is on roadmap
ON_ROADMAP=false
if git show "roadmap:.planning/ROADMAP-MANIFEST.md" 2>/dev/null | grep -q "| ${CONCEPT_NAME} |"; then
    ON_ROADMAP=true
    echo "✅ Concept found on roadmap"
fi

if [ "$ON_ROADMAP" = false ]; then
    echo ""
    echo "⚠️ Concept '${CONCEPT_NAME}' is not on the roadmap branch"
    echo ""
    echo "This means either:"
    echo "  1. The concept hasn't been fully approved yet"
    echo "  2. The merge to roadmap hasn't completed"
    echo ""
    echo "Check approval status:"
    echo "  /wgsd status ${CONCEPT_NAME}"
    echo ""
    echo "If the concept is ready, merge it to roadmap:"
    echo "  /wgsd merge-to-roadmap ${CONCEPT_NAME}"
    echo ""
    
    if [ -z "$BYPASS_ROADMAP" ]; then
        echo "To bypass roadmap requirement (emergency only):"
        echo "  Set BYPASS_ROADMAP=1 or use --from-develop flag"
        exit 1
    else
        echo "⚠️ BYPASS: Proceeding without roadmap verification"
    fi
fi
```

## Step 8: Create Git Branch and Worktree (Phase 13: from roadmap)

```bash
echo ""
echo "🌿 Setting up git branch and worktree..."

# Ensure we're on develop and up to date
CURRENT_BRANCH=$(git branch --show-current)
git stash push -m "WGSD: temp stash for implementation creation" 2>/dev/null || true

# Phase 13: Branch from roadmap, not develop
if [ "$ON_ROADMAP" = true ] && [ -z "$BYPASS_ROADMAP" ]; then
    BASE_BRANCH="roadmap"
    echo "   Base branch: roadmap (Phase 13 three-tier branching)"
else
    BASE_BRANCH=$(git rev-parse --verify develop >/dev/null 2>&1 && echo "develop" || echo "main")
    echo "   Base branch: ${BASE_BRANCH} (emergency bypass)"
fi

# Checkout and update base branch
git checkout "$BASE_BRANCH" 2>/dev/null
git pull origin "$BASE_BRANCH" 2>/dev/null || true

# Create implementation branch
IMPL_BRANCH="implementations/${CONCEPT_NAME}"

if git show-ref --verify --quiet "refs/heads/${IMPL_BRANCH}"; then
    echo "   Branch exists, checking out..."
    git checkout "${IMPL_BRANCH}"
else
    echo "   Creating branch: ${IMPL_BRANCH} (from ${BASE_BRANCH})"
    git checkout -b "${IMPL_BRANCH}"
    
    # Add placeholder file
    cat > ".impl-${CONCEPT_NAME}.md" << EOF
# Implementation: ${CONCEPT_NAME}

**Base Branch:** ${BASE_BRANCH}
**Created:** $(date -u +"%Y-%m-%dT%H:%M:%SZ")
**Concept Source:** ${FOCUS_GROUP}

---

This implementation branch was created from the ${BASE_BRANCH} branch,
which contains the fully approved concept.

## Merge Target

When complete, this branch will merge to:
- **develop/main** (production code)

The roadmap branch will be synced via \`/wgsd roadmap-sync\`.
EOF
    git add ".impl-${CONCEPT_NAME}.md"
    git commit -m "feat: start ${CONCEPT_NAME} implementation (from ${BASE_BRANCH})"
    git push -u origin "${IMPL_BRANCH}"
fi

# Create worktree for isolated work
WORKTREE_PATH="implementations/${CONCEPT_NAME}"

if [ -d "$WORKTREE_PATH" ]; then
    echo "   Worktree already exists: ${WORKTREE_PATH}"
else
    mkdir -p implementations
    git worktree add "${WORKTREE_PATH}" "${IMPL_BRANCH}"
    echo "   Created worktree: ${WORKTREE_PATH}"
fi

# Return to original branch
git checkout "$CURRENT_BRANCH" 2>/dev/null || git checkout develop
git stash pop 2>/dev/null || true

echo "✅ Git structure ready"
echo "   Branch: ${IMPL_BRANCH}"
echo "   Base: ${BASE_BRANCH}"
echo "   Worktree: ${WORKTREE_PATH}/"
```

## Step 8: Create Implementation Channel

```bash
echo ""
echo "📱 Creating implementation channel..."

CHANNEL_NAME="${REPO_NAME}-impl-${CONCEPT_NAME}"
CHANNEL_TOPIC="Implementation of ${CONCEPT_NAME} from #${REPO_NAME}-${FOCUS_GROUP}. Owner: @${OWNER}. Target: 1-3 days."

# Create private channel via Slack API
# Uses lib/slack-api.md functions
echo "   Channel: #${CHANNEL_NAME}"
echo "   Topic: ${CHANNEL_TOPIC}"

# Would invoke create_private_channel from slack-api.md
```

## Step 9: Create Implementation Canvas

```bash
echo ""
echo "🖼️ Creating implementation canvas..."

CANVAS_CONTENT="# ${CONCEPT_NAME} Implementation

**Source:** ${CONCEPT_NAME} from ${FOCUS_GROUP}
**Owner:** @${OWNER}
**Status:** 🟡 Initializing
**Progress:** 0%
**Timeline:** 1-3 days

## Current Phase
Phase 0: Requirements Definition

## Daily Progress
*Updated automatically from STATE.md*

### Day 0
- Implementation created
- Ready to begin

## Blockers
*None*

---
*Canvas syncs from \`.planning/active-implementations/${IMPL_NAME}/STATE.md\`*"

# Would invoke create_channel_canvas from canvas-api.md
echo "   Canvas created and attached to channel"
```

## Step 10: Update Queue and Master Dashboard

```bash
echo ""
echo "📋 Updating implementation queue..."

QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"

# Remove from queue (it was waiting)
# Add to active section
# Uses lib/priority-queue.md functions

# Update master dashboard
echo "   Syncing master dashboard..."
# Would invoke canvas sync
```

## Step 11: Update Source Concept

```bash
echo ""
echo "📝 Updating source concept..."

# Update concept status
sed -i "s/\*\*Status:\*\* .*/\*\*Status:\*\* 🔧 In Implementation/" "$CONCEPT_FILE"

# Add implementation reference
cat >> "$CONCEPT_FILE" << EOF

## Implementation Started

**Started:** ${TIMESTAMP}
**Implementation:** ${IMPL_NAME}
**Channel:** #${REPO_NAME}-impl-${CONCEPT_NAME}
**Owner:** @${OWNER}
**Branch:** implementations/${CONCEPT_NAME}
EOF

# Commit changes
git add "$CONCEPT_FILE"
git add "${IMPL_DIR}"
git commit -m "feat(${FOCUS_GROUP}): start ${CONCEPT_NAME} implementation [@${OWNER}]"

echo "✅ Concept updated"
```

## Step 12: Send Notifications

```bash
echo ""
echo "📢 Sending notifications..."
```

**Focus Group Message:**
```markdown
🚀 **Implementation Started: {concept-name}**

The concept has moved to active implementation!

**Owner:** @{owner}
**Channel:** #{repo-name}-impl-{concept-name}
**Timeline:** 1-3 days

**Track progress:**
- Join #{repo-name}-impl-{concept-name} for updates
- View `.planning/active-implementations/{impl-name}/STATE.md`

*Great teamwork maturing this concept!*
```

**Implementation Channel Welcome:**
```markdown
⚡ **Implementation: {Concept Name}**

**Owner:** @{owner}
**Source:** {concept-name} from #{repo-name}-{focus-group}
**Timeline:** 1-3 days
**Branch:** implementations/{concept-name}
**Worktree:** implementations/{concept-name}/

---

**This is a focused execution channel.** Different from concept discussion:
- Fast-paced coding using GSD workflow
- Requirements → Phases → Task commits
- **Daily status updates required**
- Blockers escalated immediately

---

**Next Steps:**
1. Switch to worktree: `cd implementations/{concept-name}`
2. Define requirements from concept context
3. Create GSD roadmap with phases
4. Begin implementation

**To update status:**
`/wgsd impl-status {concept-name} "Phase 1: 50% complete"`

*Jarvis guides the GSD workflow - no @ needed*
```

## Step 13: Return Success

```
---

## ✅ Implementation Created: {concept-name}

**Owner:** @{owner}
**Focus Group:** {focus-group}
**Channel:** #{repo-name}-impl-{concept-name}
**Branch:** implementations/{concept-name}
**Worktree:** implementations/{concept-name}/

### Capacity
- **Global:** {current+1}/4 implementations active
- **{focus-group}:** {fg-count+1} implementation(s) active

### Next Steps

1. **Switch to worktree:**
   ```
   cd implementations/{concept-name}
   ```

2. **Define requirements** from concept context

3. **Create GSD roadmap** with 2-3 phases

4. **Daily updates** required:
   ```
   /wgsd impl-status {concept-name} "Phase 1: 25%"
   ```

5. **When ready, create PR:**
   ```
   /wgsd impl-pr {concept-name}
   ```

---

**Timeline commitment:** Complete within 1-3 days
**Owner responsibility:** Daily updates, flag blockers

---
```

</process>

<error_handling>

## Error Recovery

| Error | Recovery |
|-------|----------|
| Branch already exists | Checkout existing branch |
| Worktree conflict | Clean up orphan worktrees |
| Channel exists | Use existing channel |
| Owner invalid | Prompt for valid owner |

```bash
# Clean orphan worktrees
git worktree prune

# Check for orphan branches
git branch -vv | grep ": gone]"
```

</error_handling>

<success_criteria>
- [ ] Global capacity validated (< 4 concurrent)
- [ ] Focus group scaling checked (warning or override for > 1)
- [ ] Concept found and in "Ready" status
- [ ] Owner verified and assigned
- [ ] Implementation directory created
- [ ] concept-source.md and STATE.md created
- [ ] Git branch created off develop
- [ ] Worktree created for isolated work
- [ ] Implementation channel created (private)
- [ ] Canvas created with status template
- [ ] Queue updated (moved from queued to active)
- [ ] Source concept status updated to "In Implementation"
- [ ] Notifications sent to focus group and impl channel
- [ ] Clear next steps provided for owner
</success_criteria>
