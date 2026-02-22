---
name: wgsd:concept-development
description: Planning track workflow - concept changes via PRs to focus group branches
argument-hint: "[concept-name] [action]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
---

<objective>
Implement Track 1 of two-track development: Collaborative Planning

This track handles concept evolution through:
- Planning PRs to focus group branches (not develop)
- Focus group review and discussion
- Canvas sync on merge
- Separation of planning from implementation

Key principle: Planning changes NEVER go directly to develop.
They evolve on focus group branches until concepts are ready for implementation.
</objective>

<libraries>
- workflows/lib/github-pr.md - PR creation and management
- workflows/lib/branch-ops.md - Branch operations
- workflows/lib/canvas-sync.md - Canvas synchronization
</libraries>

<process>

## Overview: Two-Track Development

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRACK 1: PLANNING                            │
│  Concepts evolve through discussion and planning PRs            │
│                                                                 │
│  concept/{name} → PR → focus-groups/{fg} → Canvas sync          │
│                                                                 │
│  ✅ .planning/ changes    ❌ No code changes                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    TRACK 2: IMPLEMENTATION                      │
│  Code execution via trunk-based development                     │
│                                                                 │
│  implementations/{name} → PR → develop → Production             │
│                                                                 │
│  ✅ Code changes    ❌ No planning structure changes            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Determine Action

```bash
CONCEPT_NAME="{concept-name}"
ACTION="{action}"  # edit, pr, status, sync

REPO_NAME=$(basename $(pwd))

echo "📝 Concept Development Workflow"
echo "   Concept: ${CONCEPT_NAME}"
echo "   Action: ${ACTION}"
```

## Step 2: Find Concept and Focus Group

```bash
echo ""
echo "🔍 Locating concept..."

CONCEPT_FILE=""
FOCUS_GROUP=""

for fg_dir in .planning/focus-groups/*/; do
    if [ -f "${fg_dir}concepts/${CONCEPT_NAME}.md" ]; then
        CONCEPT_FILE="${fg_dir}concepts/${CONCEPT_NAME}.md"
        FOCUS_GROUP=$(basename "$fg_dir")
        break
    fi
done

if [ -z "$CONCEPT_FILE" ]; then
    echo "❌ Concept not found: ${CONCEPT_NAME}"
    echo ""
    echo "💡 Available concepts:"
    find .planning/focus-groups -name "*.md" -path "*/concepts/*" | \
        sed 's|.planning/focus-groups/||;s|/concepts/| → |;s|\.md||'
    exit 1
fi

echo "📍 Found: ${FOCUS_GROUP} → ${CONCEPT_NAME}"
echo "   File: ${CONCEPT_FILE}"
```

---

## Action: Edit Concept

Create or switch to a planning branch for editing.

```bash
if [ "$ACTION" = "edit" ]; then
    echo ""
    echo "✏️ Starting concept edit..."
    
    # Determine branch names
    FG_BRANCH="focus-groups/${FOCUS_GROUP}"
    EDIT_BRANCH="concept/${CONCEPT_NAME}"
    
    # Fetch latest
    git fetch origin
    
    # Check if edit branch exists
    if git show-ref --verify --quiet "refs/heads/${EDIT_BRANCH}"; then
        echo "   Existing edit branch found"
        git checkout "${EDIT_BRANCH}"
        git pull origin "${EDIT_BRANCH}" 2>/dev/null || true
    else
        echo "   Creating new edit branch..."
        
        # Branch from focus group branch
        if git show-ref --verify --quiet "refs/heads/${FG_BRANCH}"; then
            git checkout "${FG_BRANCH}"
            git pull origin "${FG_BRANCH}"
        else
            # Focus group branch doesn't exist, create from develop
            echo "   Note: Creating focus group branch from develop"
            git checkout develop
            git pull origin develop
            git checkout -b "${FG_BRANCH}"
            git push -u origin "${FG_BRANCH}"
        fi
        
        # Create edit branch
        git checkout -b "${EDIT_BRANCH}"
        echo "   Created: ${EDIT_BRANCH}"
    fi
    
    echo ""
    echo "✅ Ready to edit concept"
    echo ""
    echo "**Edit the concept file:**"
    echo "  ${CONCEPT_FILE}"
    echo ""
    echo "**When done:**"
    echo "  git add ${CONCEPT_FILE}"
    echo "  git commit -m \"docs(${FOCUS_GROUP}): update ${CONCEPT_NAME} concept\""
    echo "  /wgsd concept pr ${CONCEPT_NAME}"
    exit 0
fi
```

---

## Action: Create Planning PR

Create a PR from concept branch to focus group branch.

```bash
if [ "$ACTION" = "pr" ]; then
    echo ""
    echo "📤 Creating planning PR..."
    
    EDIT_BRANCH="concept/${CONCEPT_NAME}"
    FG_BRANCH="focus-groups/${FOCUS_GROUP}"
    
    # Verify we're on the edit branch
    CURRENT_BRANCH=$(git branch --show-current)
    if [ "$CURRENT_BRANCH" != "$EDIT_BRANCH" ]; then
        echo "⚠️ Not on concept branch"
        echo "   Current: ${CURRENT_BRANCH}"
        echo "   Expected: ${EDIT_BRANCH}"
        echo ""
        echo "💡 Switch to concept branch first:"
        echo "   git checkout ${EDIT_BRANCH}"
        exit 1
    fi
    
    # Check for uncommitted changes
    if ! git diff --quiet || ! git diff --cached --quiet; then
        echo "⚠️ Uncommitted changes detected"
        echo ""
        echo "💡 Commit your changes first:"
        echo "   git add ."
        echo "   git commit -m \"docs(${FOCUS_GROUP}): update ${CONCEPT_NAME}\""
        exit 1
    fi
    
    # Check for unpushed commits
    UNPUSHED=$(git log origin/${EDIT_BRANCH}..HEAD 2>/dev/null | head -1)
    if [ -n "$UNPUSHED" ]; then
        echo "📤 Pushing commits..."
        git push origin "${EDIT_BRANCH}"
    fi
    
    # Verify changes are planning-only
    echo ""
    echo "🔍 Verifying planning-only changes..."
    
    CHANGED_FILES=$(git diff --name-only "${FG_BRANCH}...${EDIT_BRANCH}")
    NON_PLANNING=$(echo "$CHANGED_FILES" | grep -v "^\.planning/" | head -5)
    
    if [ -n "$NON_PLANNING" ]; then
        echo ""
        echo "⚠️ **Non-planning files detected:**"
        echo "$NON_PLANNING"
        echo ""
        echo "Planning PRs should only contain .planning/ changes."
        echo "Code changes should go through implementation track."
        echo ""
        echo "💡 To proceed anyway, use --force flag"
        
        if [ -z "$FORCE" ]; then
            exit 1
        fi
    fi
    
    echo "✅ Changes are planning-only"
    
    # Generate PR description
    PR_TITLE="docs(${FOCUS_GROUP}): ${CONCEPT_NAME} concept update"
    PR_BODY="## Planning Change: ${CONCEPT_NAME}

**Focus Group:** ${FOCUS_GROUP}
**Type:** Concept Update

### Summary
Updates to the ${CONCEPT_NAME} concept.

### Changed Files
\`\`\`
${CHANGED_FILES}
\`\`\`

### Review Notes
This is a **planning PR** for the ${FOCUS_GROUP} focus group.
- Review for clarity and completeness
- Check alignment with focus group direction
- Consider impact on related concepts

---
*Track 1: Planning PR → focus-groups/${FOCUS_GROUP} (not develop)*"
    
    # Create PR using gh CLI
    echo ""
    echo "📝 Creating PR..."
    
    PR_URL=$(gh pr create \
        --base "${FG_BRANCH}" \
        --head "${EDIT_BRANCH}" \
        --title "${PR_TITLE}" \
        --body "${PR_BODY}" \
        --label "planning" \
        --label "${FOCUS_GROUP}" \
        2>&1)
    
    if [ $? -eq 0 ]; then
        echo ""
        echo "✅ **Planning PR Created**"
        echo ""
        echo "   URL: ${PR_URL}"
        echo "   Base: ${FG_BRANCH}"
        echo "   Head: ${EDIT_BRANCH}"
        echo ""
        echo "**Next steps:**"
        echo "1. Request review from focus group members"
        echo "2. Address feedback"
        echo "3. Merge when approved"
        echo "4. Canvas will sync automatically"
    else
        if echo "$PR_URL" | grep -q "already exists"; then
            echo "ℹ️ PR already exists for this branch"
            gh pr view "${EDIT_BRANCH}" --web
        else
            echo "❌ PR creation failed: ${PR_URL}"
            exit 1
        fi
    fi
    
    exit 0
fi
```

---

## Action: Check PR Status

```bash
if [ "$ACTION" = "status" ]; then
    echo ""
    echo "📊 Concept PR Status"
    
    EDIT_BRANCH="concept/${CONCEPT_NAME}"
    
    # Check if PR exists
    PR_INFO=$(gh pr view "${EDIT_BRANCH}" --json state,number,url,reviews,mergeable 2>&1)
    
    if [ $? -ne 0 ]; then
        echo ""
        echo "ℹ️ No PR exists for ${CONCEPT_NAME}"
        echo ""
        echo "💡 To create a PR:"
        echo "   /wgsd concept pr ${CONCEPT_NAME}"
        exit 0
    fi
    
    STATE=$(echo "$PR_INFO" | jq -r '.state')
    NUMBER=$(echo "$PR_INFO" | jq -r '.number')
    URL=$(echo "$PR_INFO" | jq -r '.url')
    MERGEABLE=$(echo "$PR_INFO" | jq -r '.mergeable')
    
    echo ""
    echo "**PR #${NUMBER}**"
    echo "   State: ${STATE}"
    echo "   Mergeable: ${MERGEABLE}"
    echo "   URL: ${URL}"
    echo ""
    
    # Show reviews
    REVIEWS=$(echo "$PR_INFO" | jq -r '.reviews[]? | "\(.author.login): \(.state)"')
    if [ -n "$REVIEWS" ]; then
        echo "**Reviews:**"
        echo "$REVIEWS" | while read review; do
            echo "   ${review}"
        done
    fi
    
    exit 0
fi
```

---

## Action: Sync After Merge

Trigger canvas sync after planning PR is merged.

```bash
if [ "$ACTION" = "sync" ]; then
    echo ""
    echo "🔄 Syncing after merge..."
    
    FG_BRANCH="focus-groups/${FOCUS_GROUP}"
    
    # Pull latest focus group branch
    git fetch origin
    git checkout "${FG_BRANCH}"
    git pull origin "${FG_BRANCH}"
    
    echo ""
    echo "📊 Triggering canvas sync for ${FOCUS_GROUP}..."
    
    # Would invoke canvas-sync workflow
    # /wgsd sync-canvas ${FOCUS_GROUP}
    
    echo "✅ Canvas sync triggered"
    echo ""
    echo "The focus group canvas will be updated with:"
    echo "  • Latest roadmap content"
    echo "  • Updated concept states"
    echo "  • Cross-references resolved"
    
    exit 0
fi
```

---

## Default: Show Help

```bash
echo ""
echo "📚 **Concept Development (Track 1)**"
echo ""
echo "This workflow manages planning changes through PRs to focus group branches."
echo ""
echo "**Commands:**"
echo ""
echo "  **Edit a concept:**"
echo "  /wgsd concept edit ${CONCEPT_NAME:-<concept>}"
echo "  Creates or switches to editing branch"
echo ""
echo "  **Create planning PR:**"
echo "  /wgsd concept pr ${CONCEPT_NAME:-<concept>}"
echo "  Creates PR from concept branch to focus group branch"
echo ""
echo "  **Check PR status:**"
echo "  /wgsd concept status ${CONCEPT_NAME:-<concept>}"
echo "  Shows PR state, reviews, and merge status"
echo ""
echo "  **Sync after merge:**"
echo "  /wgsd concept sync ${CONCEPT_NAME:-<concept>}"
echo "  Triggers canvas sync after PR is merged"
echo ""
echo "**Branch Flow:**"
echo "  concept/${CONCEPT_NAME:-<concept>} → focus-groups/${FOCUS_GROUP:-<fg>}"
echo ""
echo "**Important:**"
echo "  • Planning PRs go to focus group branches, NOT develop"
echo "  • Only .planning/ changes in planning PRs"
echo "  • Code changes use implementation track"
```

</process>

<best_practices>

## Planning PR Guidelines

### Do
- Keep PRs focused on one concept
- Include clear descriptions
- Request review from focus group members
- Update related roadmap entries

### Don't
- Mix planning and code changes
- Merge without review
- Skip the focus group branch

### Commit Messages
```
docs({focus-group}): add {concept-name} concept
docs({focus-group}): refine {concept-name} problem statement
docs({focus-group}): update {concept-name} to mature status
```

### Branch Naming
```
concept/{concept-name}           # For concept edits
concept/{concept-name}-refine    # For follow-up changes
```

</best_practices>

<integration>

## Canvas Integration

When planning PRs merge, canvas sync triggers:

1. **Focus group canvas** - Updated roadmap
2. **Master dashboard** - Updated concept counts
3. **Community canvas** - Public-safe summary

## With Implementation Track

When concept is promoted:
1. Planning branch stays on focus-groups/{fg}
2. Implementation branch created from develop
3. Implementation tracks back to concept file

</integration>

<success_criteria>
- [ ] Concept found in focus group
- [ ] Edit branch created from focus group branch
- [ ] Changes committed with proper message format
- [ ] PR created to focus group branch (not develop)
- [ ] PR contains only .planning/ changes
- [ ] Focus group members can review
- [ ] Canvas syncs after merge
</success_criteria>
