---
name: wgsd:promote-concept
description: Promote a mature concept to implementation with approval gates and owner assignment
argument-hint: "[concept-name] [@owner]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Promote a concept from "Ready for Implementation" to active implementation with:
- Approval gate verification (minimum 2 votes)
- Owner assignment (required before implementation starts)
- Priority queue placement
- Focus group natural scaling check
- Canvas dashboard update

This workflow handles the critical transition from social planning to focused execution.
</objective>

<libraries>
- workflows/lib/approval-system.md - Vote counting and state transitions
- workflows/lib/priority-queue.md - Queue management
</libraries>

<process>

## Step 1: Validate Concept Exists and Find Location

```bash
REPO_NAME=$(basename $(pwd))
CONCEPT_NAME="{concept-name}"
OWNER="{owner}"  # Optional, can be assigned later

# Search for concept across all focus groups
CONCEPT_FILE=""
FOCUS_GROUP=""

echo "🔍 Searching for concept: ${CONCEPT_NAME}"

for fg_dir in .planning/focus-groups/*/; do
    if [ -f "${fg_dir}concepts/${CONCEPT_NAME}.md" ]; then
        CONCEPT_FILE="${fg_dir}concepts/${CONCEPT_NAME}.md"
        FOCUS_GROUP=$(basename "$fg_dir")
        echo "📍 Found: ${CONCEPT_NAME} in ${FOCUS_GROUP} focus group"
        break
    fi
done

if [ -z "$CONCEPT_FILE" ]; then
    echo "❌ Concept '${CONCEPT_NAME}' not found"
    echo "💡 Available concepts:"
    find .planning/focus-groups -name "*.md" -path "*/concepts/*" | \
        sed 's|.planning/focus-groups/||;s|/concepts/| → |;s|\.md||'
    exit 1
fi
```

## Step 2: Check Current Status

```bash
CURRENT_STATUS=$(grep "^\*\*Status:\*\*" "$CONCEPT_FILE" | head -1 | sed 's/\*\*Status:\*\* //')

echo "📊 Current status: ${CURRENT_STATUS}"

case "$CURRENT_STATUS" in
    *"Draft"*)
        echo "❌ Concept is still in Draft status"
        echo "💡 Move through exploration and maturation first:"
        echo "   📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready"
        exit 1
        ;;
    *"Exploring"*)
        echo "❌ Concept is still being explored"
        echo "💡 Mature the concept through focus group discussion first"
        exit 1
        ;;
    *"Mature"*)
        echo "⏳ Concept is mature - checking approval status..."
        # Continue to approval check
        ;;
    *"Ready"*)
        echo "✅ Concept is already ready for implementation"
        echo "💡 Assign an owner to start: /wgsd assign ${CONCEPT_NAME} @user"
        # Skip to owner assignment
        ;;
    *"In Implementation"*)
        echo "⚠️ Concept is already being implemented"
        echo "💡 Check implementation status: /wgsd impl-status ${CONCEPT_NAME}"
        exit 0
        ;;
    *)
        echo "⚠️ Unknown status: ${CURRENT_STATUS}"
        ;;
esac
```

## Step 3: Check Approval Gate

```bash
# Source approval system library
# In actual execution, this would load the library functions

echo "🗳️ Checking approval votes..."

# Count approvals and rejections
APPROVE_COUNT=$(grep -E "\| @.+ \| (👍|✅|\+1) \|" "$CONCEPT_FILE" 2>/dev/null | wc -l)
REJECT_COUNT=$(grep -E "\| @.+ \| (👎|❌|\-1) \|" "$CONCEPT_FILE" 2>/dev/null | wc -l)
MIN_VOTES=2

echo "   Approvals: ${APPROVE_COUNT}/${MIN_VOTES}"
echo "   Rejections: ${REJECT_COUNT}"

# Check for rejections first
if [ "$REJECT_COUNT" -gt 0 ]; then
    echo ""
    echo "⚠️ This concept has objections that should be addressed:"
    grep -E "\| @.+ \| (👎|❌) \|" "$CONCEPT_FILE" | while read line; do
        USER=$(echo "$line" | awk -F'|' '{print $2}' | xargs)
        COMMENT=$(echo "$line" | awk -F'|' '{print $4}' | xargs)
        echo "   ${USER}: ${COMMENT}"
    done
    echo ""
    echo "💡 Discuss with the team to resolve objections before promoting"
    exit 1
fi

# Check approval threshold
if [ "$APPROVE_COUNT" -lt "$MIN_VOTES" ]; then
    REMAINING=$((MIN_VOTES - APPROVE_COUNT))
    echo ""
    echo "❌ Insufficient approvals (${APPROVE_COUNT}/${MIN_VOTES})"
    echo "💡 Need ${REMAINING} more approval(s) from focus group members"
    echo ""
    echo "**To approve:**"
    echo "  • React with 👍 to the voting message, OR"
    echo "  • Run: /wgsd approve ${CONCEPT_NAME}"
    echo ""
    
    # Initialize voting if not already started
    if ! grep -q "## Approval Status" "$CONCEPT_FILE"; then
        echo "🗳️ Starting voting process..."
        # Would call initialize_voting from approval-system.md
    fi
    
    exit 1
fi

echo "✅ Approval threshold met! (${APPROVE_COUNT} approvals)"
```

## Step 4: Check Natural Scaling (Focus Group Implementation Limit)

```bash
echo ""
echo "📊 Checking focus group scaling..."

# Count active implementations for this focus group
FG_IMPL_COUNT=0
if [ -d .planning/active-implementations ]; then
    FG_IMPL_COUNT=$(find .planning/active-implementations -name "concept-source.md" \
        -exec grep -l "Source Focus Group: ${FOCUS_GROUP}" {} \; 2>/dev/null | wc -l)
fi

echo "   ${FOCUS_GROUP} focus group has ${FG_IMPL_COUNT} active implementation(s)"

if [ "$FG_IMPL_COUNT" -ge 1 ]; then
    echo ""
    echo "⚠️ **Scaling Warning:** ${FOCUS_GROUP} already has an active implementation"
    echo ""
    echo "WGSD recommends ~1 implementation per focus group to:"
    echo "  • Prevent context switching"
    echo "  • Maintain domain expertise focus"
    echo "  • Enable natural team scaling"
    echo ""
    echo "**Options:**"
    echo "1. Wait for current implementation to complete"
    echo "2. Override with justification: /wgsd promote ${CONCEPT_NAME} --override \"<reason>\""
    echo ""
    
    # Check for override flag
    if [ -z "$OVERRIDE_REASON" ]; then
        echo "No override provided. Promotion blocked."
        exit 1
    else
        echo "⚠️ Override accepted: ${OVERRIDE_REASON}"
        echo "   Logging override decision..."
        # Log the override
        echo "- $(date -u +"%Y-%m-%dT%H:%M:%SZ"): Scaling override for ${CONCEPT_NAME} - Reason: ${OVERRIDE_REASON}" >> \
            .planning/focus-groups/${FOCUS_GROUP}/STATE.md
    fi
fi

echo "✅ Scaling check passed"
```

## Step 5: Require Owner Assignment

```bash
echo ""
echo "👤 Checking owner assignment..."

# Owner can be provided as argument or prompted
if [ -z "$OWNER" ] || [ "$OWNER" = "{owner}" ]; then
    echo ""
    echo "📋 **Owner Required**"
    echo ""
    echo "An owner must be assigned before implementation can begin."
    echo "The owner commits to:"
    echo "  • 1-3 day implementation timeline"
    echo "  • Daily status updates"
    echo "  • Driving the implementation to completion"
    echo ""
    echo "**To assign an owner:**"
    echo "  /wgsd assign ${CONCEPT_NAME} @user"
    echo ""
    echo "**To claim it yourself:**"
    echo "  /wgsd claim ${CONCEPT_NAME}"
    echo ""
    
    # Add to queue without owner
    echo "📋 Adding ${CONCEPT_NAME} to implementation queue (awaiting owner)..."
    # Would call add_to_queue from priority-queue.md
    
    # Update concept status
    sed -i "s/\*\*Status:\*\* .*/\*\*Status:\*\* 🚀 Ready for Implementation/" "$CONCEPT_FILE"
    
    exit 0
fi

# Validate owner is a focus group participant
echo "   Validating owner: @${OWNER}"

# In practice, would check Slack channel membership
# For now, accept any valid username format
if [[ ! "$OWNER" =~ ^@?[a-zA-Z0-9_.-]+$ ]]; then
    echo "❌ Invalid owner format: ${OWNER}"
    exit 1
fi

# Remove @ prefix if present for consistency
OWNER=$(echo "$OWNER" | sed 's/^@//')

echo "✅ Owner assigned: @${OWNER}"
```

## Step 6: Select Priority Level

```bash
echo ""
echo "📊 Setting implementation priority..."

# Default to medium, can be overridden
PRIORITY="${PRIORITY:-medium}"

echo "   Priority: ${PRIORITY}"
echo ""
echo "💡 Change priority with: /wgsd set-priority ${CONCEPT_NAME} [critical|high|medium|low]"
```

## Step 7: Update Concept Status

```bash
echo ""
echo "📝 Updating concept status..."

TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Update status to Ready for Implementation
sed -i "s/\*\*Status:\*\* .*/\*\*Status:\*\* 🚀 Ready for Implementation/" "$CONCEPT_FILE"

# Add promotion record
cat >> "$CONCEPT_FILE" << EOF

## Promotion Record

**Promoted:** ${TIMESTAMP}
**Approved By:** $(grep -E "\| @.+ \| (👍|✅) \|" "$CONCEPT_FILE" | sed 's/| @\([^ ]*\) |.*/\1/' | tr '\n' ', ' | sed 's/, $//')
**Priority:** ${PRIORITY}
**Assigned Owner:** @${OWNER}
**Focus Group:** ${FOCUS_GROUP}
EOF

# Commit changes
git add "$CONCEPT_FILE"
git commit -m "feat(${FOCUS_GROUP}): promote ${CONCEPT_NAME} to ready [approved]"

echo "✅ Concept status updated"
```

## Step 8: Add to Implementation Queue

```bash
echo ""
echo "📋 Adding to implementation queue..."

QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"

# Ensure queue file exists
if [ ! -f "$QUEUE_FILE" ]; then
    # Would call initialize_queue from priority-queue.md
    echo "Creating queue file..."
fi

# Map priority to section
case "$PRIORITY" in
    critical) SECTION="🔴 Critical" ;;
    high) SECTION="🟠 High" ;;
    medium) SECTION="🟡 Medium" ;;
    low) SECTION="🟢 Low" ;;
esac

echo "   Added to queue: ${PRIORITY} priority"
echo "   Position will be determined by priority tier"

# Update queue stats
git add "$QUEUE_FILE"
git commit -m "docs: add ${CONCEPT_NAME} to implementation queue [${PRIORITY}]"
```

## Step 9: Sync Canvas Dashboard

```bash
echo ""
echo "🖼️ Updating canvas dashboard..."

# Trigger canvas sync for:
# 1. Focus group canvas - show concept as promoted
# 2. Master dashboard - show new queue item

# Would invoke workflows/canvas-sync.md
echo "   Syncing focus group canvas: #${REPO_NAME}-${FOCUS_GROUP}"
echo "   Syncing master dashboard"

echo "✅ Canvas updated"
```

## Step 10: Notify Team

```bash
echo ""
echo "📢 Sending notifications..."

# Focus group notification
FG_MESSAGE="🚀 **Concept Promoted: ${CONCEPT_NAME}**

The concept has been approved and is ready for implementation!

**Owner:** @${OWNER}
**Priority:** ${PRIORITY}
**Approvers:** $(grep -E "\| @.+ \| (👍|✅) \|" "$CONCEPT_FILE" | sed 's/| @\([^ ]*\) |.*/\1/' | tr '\n' ', ' | sed 's/, $//')

**Next Steps:**
1. @${OWNER} will start the implementation
2. Implementation channel will be created
3. Daily status updates in the impl channel

*Great teamwork on maturing this concept!*"

echo "📨 Message to #${REPO_NAME}-${FOCUS_GROUP}:"
echo "$FG_MESSAGE"
echo ""

# Dev channel notification (brief)
DEV_MESSAGE="📋 **Queue Update:** \`${CONCEPT_NAME}\` added to implementation queue (${PRIORITY}) - Owner: @${OWNER}"
echo "📨 Message to #${REPO_NAME}-dev:"
echo "$DEV_MESSAGE"
```

## Step 11: Return Success

```
---

## ✅ Concept Promoted: {Concept Name}

**Focus Group:** {focus-group}
**Owner:** @{owner}
**Priority:** {priority}
**Status:** 🚀 Ready for Implementation

### Approvals
{list of approvers}

### Next Steps

The concept is now in the implementation queue.

**To start implementation:**
```
/wgsd create-implementation {concept-name}
```

**What happens next:**
1. Implementation channel created (#${repo-name}-impl-{concept-name})
2. Git branch and worktree set up
3. Traditional GSD workflow begins
4. Daily status updates required

**For the owner (@{owner}):**
- Commit to 1-3 day implementation timeline
- Post daily progress updates
- Flag blockers immediately

---
```

</process>

<variations>

## Approve a Concept

When user says "approve [concept]":

```bash
CONCEPT_NAME="{concept-name}"
USER="{current-user}"  # From Slack context

# Find concept and record vote
# ... find concept ...

record_vote "$CONCEPT_NAME" "$FOCUS_GROUP" "$USER" "approve" ""

echo "✅ Your approval has been recorded for ${CONCEPT_NAME}"

# Check if threshold now met
check_approval_status "$CONCEPT_NAME" "$FOCUS_GROUP"
if [ $? -eq 0 ]; then
    echo "🎉 Approval threshold reached! Ready for owner assignment."
    echo "   /wgsd assign ${CONCEPT_NAME} @user"
fi
```

## Claim a Concept

When user says "claim [concept]":

```bash
CONCEPT_NAME="{concept-name}"
OWNER="{current-user}"  # From Slack context

# This is self-assignment
echo "📋 Claiming ${CONCEPT_NAME} for yourself..."

# Run promote with current user as owner
# This workflow handles all validation
```

## Assign Owner

When user says "assign [concept] @user":

```bash
CONCEPT_NAME="{concept-name}"
OWNER="{specified-user}"

# Validate concept is in "Ready" state
# Update concept file with owner
# Add to queue with owner
# Notify owner of assignment
```

</variations>

<error_handling>

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Concept not found | Invalid name or not created | Check spelling, list available concepts |
| Insufficient approvals | < 2 votes | Continue focus group discussion |
| Has objections | Rejection votes present | Address concerns before promoting |
| Scaling limit | FG already has active impl | Wait or use override |
| No owner | Owner not specified | Assign or claim the concept |
| Invalid owner | Bad username format | Use valid Slack username |

## Error Messages

```bash
# Concept not ready
echo "❌ Cannot promote: Concept must be in 'Mature' or 'Ready' state"
echo "   Current: ${CURRENT_STATUS}"
echo "   Path: Draft → Exploring → Mature → Ready"

# Scaling override needed
echo "⚠️ Focus group scaling limit reached"
echo "   Use: /wgsd promote ${CONCEPT} --override \"reason\""
```

</error_handling>

<success_criteria>
- [ ] Concept found and status validated
- [ ] Approval gate verified (2+ approvals, no objections)
- [ ] Focus group scaling check passed (or override provided)
- [ ] Owner assigned and validated
- [ ] Priority level set
- [ ] Concept status updated to "Ready for Implementation"
- [ ] Added to implementation queue
- [ ] Canvas dashboards synced
- [ ] Team notifications sent
- [ ] Clear next steps provided
</success_criteria>
