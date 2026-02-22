# GitHub PR Library

**Purpose:** GitHub Pull Request integration for WGSD two-track development

---

## Overview

This library provides PR operations for both development tracks:
- **Track 1 (Planning):** PRs to focus group branches for concept/roadmap changes
- **Track 2 (Implementation):** PRs to develop branch for code changes

---

## Configuration

```bash
# Get GitHub token from environment or config
GITHUB_TOKEN="${GITHUB_TOKEN:-$(gh auth token 2>/dev/null)}"

# Detect repository info
REPO_OWNER=$(git remote get-url origin | sed 's/.*github.com[:/]\([^/]*\)\/.*/\1/')
REPO_NAME=$(git remote get-url origin | sed 's/.*\/\([^.]*\)\.git.*/\1/')
```

---

## Create Planning PR

Create a PR for planning/concept changes to a focus group branch.

### Usage
```bash
create_planning_pr "concept-name" "focus-group-name" "Description of changes"
```

### Implementation
```bash
create_planning_pr() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local DESCRIPTION="$3"
    
    local HEAD_BRANCH="concept/${CONCEPT_NAME}"
    local BASE_BRANCH="focus-groups/${FOCUS_GROUP}"
    local TITLE="feat(${FOCUS_GROUP}): update ${CONCEPT_NAME} concept"
    
    # Ensure we're on the correct branch
    CURRENT_BRANCH=$(git branch --show-current)
    if [ "$CURRENT_BRANCH" != "$HEAD_BRANCH" ]; then
        echo "❌ Not on expected branch. Current: ${CURRENT_BRANCH}, Expected: ${HEAD_BRANCH}"
        return 1
    fi
    
    # Push branch if not already pushed
    if ! git ls-remote --heads origin "$HEAD_BRANCH" | grep -q "$HEAD_BRANCH"; then
        git push -u origin "$HEAD_BRANCH"
    fi
    
    # Create PR via GitHub CLI
    PR_URL=$(gh pr create \
        --base "$BASE_BRANCH" \
        --head "$HEAD_BRANCH" \
        --title "$TITLE" \
        --body "$DESCRIPTION" \
        --label "planning" \
        --label "$FOCUS_GROUP" \
        2>&1)
    
    if [ $? -eq 0 ]; then
        echo "✅ Planning PR created: $PR_URL"
        echo "$PR_URL"
    else
        echo "❌ PR creation failed: $PR_URL"
        return 1
    fi
}
```

### PR Body Template
```markdown
## Planning Change: {concept-name}

**Focus Group:** {focus-group}
**Type:** Concept/Planning Update

### Changes
{description}

### Files Changed
- `.planning/focus-groups/{focus-group}/concepts/{concept-name}.md`
- `.planning/focus-groups/{focus-group}/ROADMAP.md` (if applicable)

### Review
This is a **planning PR** - review for clarity, completeness, and alignment with focus group direction.

---
*This PR will merge to focus-groups/{focus-group} branch, not develop.*
```

---

## Create Implementation PR

Create a PR for code changes to the develop branch.

### Usage
```bash
create_implementation_pr "implementation-name" "Description of implementation"
```

### Implementation
```bash
create_implementation_pr() {
    local IMPL_NAME="$1"
    local DESCRIPTION="$2"
    
    local HEAD_BRANCH="implementations/${IMPL_NAME}"
    local BASE_BRANCH="develop"
    local TITLE="feat: ${IMPL_NAME} implementation"
    
    # Get source focus group from concept-source.md
    local SOURCE_FG=""
    if [ -f ".planning/active-implementations/${IMPL_NAME}-impl/concept-source.md" ]; then
        SOURCE_FG=$(grep "Source Focus Group:" ".planning/active-implementations/${IMPL_NAME}-impl/concept-source.md" | sed 's/.*: //')
    fi
    
    # Ensure we're on the correct branch
    CURRENT_BRANCH=$(git branch --show-current)
    if [ "$CURRENT_BRANCH" != "$HEAD_BRANCH" ]; then
        echo "❌ Not on expected branch. Current: ${CURRENT_BRANCH}, Expected: ${HEAD_BRANCH}"
        return 1
    fi
    
    # Push branch
    git push -u origin "$HEAD_BRANCH"
    
    # Build labels
    LABELS="implementation"
    if [ -n "$SOURCE_FG" ]; then
        LABELS="${LABELS},${SOURCE_FG}"
    fi
    
    # Create PR
    PR_URL=$(gh pr create \
        --base "$BASE_BRANCH" \
        --head "$HEAD_BRANCH" \
        --title "$TITLE" \
        --body "$DESCRIPTION" \
        --label "$LABELS" \
        2>&1)
    
    if [ $? -eq 0 ]; then
        echo "✅ Implementation PR created: $PR_URL"
        echo "$PR_URL"
    else
        echo "❌ PR creation failed: $PR_URL"
        return 1
    fi
}
```

### PR Body Template
```markdown
## Implementation: {implementation-name}

**Source Concept:** {concept-name}
**Focus Group:** {focus-group}
**Owner:** {owner}

### Summary
{description}

### Changes
- Core implementation of {concept-name}
- Related tests and documentation

### Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

### Checklist
- [ ] Code follows project conventions
- [ ] No unintended file changes
- [ ] Ready for code review

---
*This PR will merge to develop (trunk). Implementation channel: #{repo}-impl-{name}*
```

---

## Detect PR Type

Analyze branch name to determine development track.

### Usage
```bash
detect_pr_type "branch-name"
# Returns: "planning" | "implementation" | "unknown"
```

### Implementation
```bash
detect_pr_type() {
    local BRANCH_NAME="$1"
    
    case "$BRANCH_NAME" in
        concept/*)
            echo "planning"
            ;;
        focus-groups/*)
            echo "planning"
            ;;
        implementations/*)
            echo "implementation"
            ;;
        impl/*)
            echo "implementation"
            ;;
        feature/*)
            # Feature branches go to develop
            echo "implementation"
            ;;
        *)
            echo "unknown"
            ;;
    esac
}
```

---

## Get PR Status

Check the status of a PR for a branch.

### Usage
```bash
get_pr_status "branch-name"
```

### Implementation
```bash
get_pr_status() {
    local BRANCH_NAME="$1"
    
    # Get PR info
    PR_INFO=$(gh pr view "$BRANCH_NAME" --json state,number,url,mergedAt,reviews 2>&1)
    
    if [ $? -ne 0 ]; then
        echo "no_pr"
        return 0
    fi
    
    STATE=$(echo "$PR_INFO" | jq -r '.state')
    MERGED_AT=$(echo "$PR_INFO" | jq -r '.mergedAt')
    
    if [ "$STATE" = "MERGED" ] || [ "$MERGED_AT" != "null" ]; then
        echo "merged"
    elif [ "$STATE" = "CLOSED" ]; then
        echo "closed"
    elif [ "$STATE" = "OPEN" ]; then
        # Check review status
        REVIEWS=$(echo "$PR_INFO" | jq -r '.reviews')
        APPROVED=$(echo "$REVIEWS" | jq '[.[] | select(.state == "APPROVED")] | length')
        CHANGES=$(echo "$REVIEWS" | jq '[.[] | select(.state == "CHANGES_REQUESTED")] | length')
        
        if [ "$CHANGES" -gt 0 ]; then
            echo "changes_requested"
        elif [ "$APPROVED" -gt 0 ]; then
            echo "approved"
        else
            echo "open"
        fi
    else
        echo "unknown"
    fi
}
```

### Status Values
| Status | Description |
|--------|-------------|
| no_pr | No PR exists for this branch |
| open | PR is open, awaiting review |
| approved | PR has approvals |
| changes_requested | PR needs changes |
| merged | PR has been merged |
| closed | PR was closed without merge |

---

## List Project PRs

List all open PRs for the project.

### Usage
```bash
list_project_prs "planning" | "implementation" | "all"
```

### Implementation
```bash
list_project_prs() {
    local FILTER="$1"
    
    # Get all open PRs
    PRS=$(gh pr list --json number,title,headRefName,baseRefName,author,labels --limit 50)
    
    case "$FILTER" in
        planning)
            echo "$PRS" | jq '[.[] | select(.baseRefName | startswith("focus-groups/"))]'
            ;;
        implementation)
            echo "$PRS" | jq '[.[] | select(.baseRefName == "develop" or .baseRefName == "main")]'
            ;;
        all)
            echo "$PRS"
            ;;
    esac
}
```

### Output Format
```json
[
  {
    "number": 42,
    "title": "feat(security): update auth-v2 concept",
    "headRefName": "concept/auth-v2",
    "baseRefName": "focus-groups/security",
    "author": {"login": "alice"},
    "labels": [{"name": "planning"}, {"name": "security"}]
  }
]
```

---

## Merge PR

Merge a PR with optional squash.

### Usage
```bash
merge_pr "branch-name" "squash" | "merge" | "rebase"
```

### Implementation
```bash
merge_pr() {
    local BRANCH_NAME="$1"
    local MERGE_METHOD="${2:-squash}"
    
    # Verify PR is approved
    STATUS=$(get_pr_status "$BRANCH_NAME")
    if [ "$STATUS" != "approved" ]; then
        echo "❌ PR is not approved. Current status: $STATUS"
        return 1
    fi
    
    # Merge
    case "$MERGE_METHOD" in
        squash)
            gh pr merge "$BRANCH_NAME" --squash --delete-branch
            ;;
        merge)
            gh pr merge "$BRANCH_NAME" --merge --delete-branch
            ;;
        rebase)
            gh pr merge "$BRANCH_NAME" --rebase --delete-branch
            ;;
    esac
    
    if [ $? -eq 0 ]; then
        echo "✅ PR merged successfully"
    else
        echo "❌ Merge failed"
        return 1
    fi
}
```

---

## Get Correct Base Branch

Determine the correct base branch for a given change type.

### Usage
```bash
get_base_branch "planning" "security"
get_base_branch "implementation"
```

### Implementation
```bash
get_base_branch() {
    local CHANGE_TYPE="$1"
    local FOCUS_GROUP="$2"
    
    case "$CHANGE_TYPE" in
        planning)
            if [ -z "$FOCUS_GROUP" ]; then
                echo "❌ Focus group required for planning changes"
                return 1
            fi
            echo "focus-groups/${FOCUS_GROUP}"
            ;;
        implementation)
            # Check if develop or main exists
            if git show-ref --verify --quiet refs/heads/develop; then
                echo "develop"
            elif git show-ref --verify --quiet refs/heads/main; then
                echo "main"
            else
                echo "❌ No develop or main branch found"
                return 1
            fi
            ;;
        *)
            echo "❌ Unknown change type: $CHANGE_TYPE"
            return 1
            ;;
    esac
}
```

---

## PR Webhook Handler

Handle PR merge events to trigger canvas sync.

### On Planning PR Merge
```bash
on_planning_pr_merge() {
    local PR_NUMBER="$1"
    local BASE_BRANCH="$2"
    
    # Extract focus group from base branch
    FOCUS_GROUP=$(echo "$BASE_BRANCH" | sed 's/focus-groups\///')
    
    echo "📦 Planning PR #${PR_NUMBER} merged to ${FOCUS_GROUP}"
    
    # Trigger canvas sync for focus group
    # This would invoke workflows/canvas-sync.md
    echo "🔄 Triggering canvas sync for ${FOCUS_GROUP}"
}
```

### On Implementation PR Merge
```bash
on_implementation_pr_merge() {
    local PR_NUMBER="$1"
    local HEAD_BRANCH="$2"
    
    # Extract implementation name
    IMPL_NAME=$(echo "$HEAD_BRANCH" | sed 's/implementations\///')
    
    echo "🎉 Implementation PR #${PR_NUMBER} merged"
    
    # Mark implementation as complete
    # Update master dashboard
    # Archive implementation channel
    echo "✅ Completing implementation: ${IMPL_NAME}"
}
```

---

## Error Handling

### Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `no_remote` | Branch not pushed | `git push -u origin $BRANCH` |
| `base_not_found` | Invalid base branch | Check focus group exists |
| `already_exists` | PR already open | Use existing PR |
| `merge_conflict` | Conflicts with base | Resolve conflicts locally |
| `not_approved` | Missing reviews | Request reviews |

### Error Recovery
```bash
handle_pr_error() {
    local ERROR="$1"
    
    case "$ERROR" in
        *"already exists"*)
            echo "💡 PR already exists. Use 'gh pr view' to see it."
            ;;
        *"merge conflict"*)
            echo "💡 Merge conflicts detected. Pull latest and resolve locally."
            ;;
        *)
            echo "❌ PR error: $ERROR"
            ;;
    esac
}
```

---

## Integration with WGSD

### Usage in Workflows
```markdown
## In concept-development.md:
Source lib/github-pr.md for PR operations:
- create_planning_pr when creating PR for concept changes
- get_pr_status to check merge status

## In implementation-workflow.md:
Source lib/github-pr.md for PR operations:
- create_implementation_pr when creating code PR
- merge_pr after approval
```

---

*Library created for Phase 5: Workflow Engine*
*Supports INTEGRATE-07: GitHub PR workflow integration*
