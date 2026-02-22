---
name: wgsd:moderate-feedback
description: Collect and triage community feedback for focus group development
argument-hint: "[collect|triage|move-to-focus-group|acknowledge] [args]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Enable community feedback collection and moderation triage:
- Record feedback from public community channel
- Categorize feedback (feature, bug, improvement, question)
- Triage to appropriate focus groups
- Create concepts from valuable feedback
- Notify community of triage decisions
</objective>

<libraries>
- workflows/lib/slack-api.md - Channel and message operations
- workflows/lib/channel-registry.md - Channel lookups
</libraries>

<process>

## Action: Collect Feedback

Record a community message as feedback for triage:

```bash
MESSAGE_LINK="{message-link}"
FEEDBACK_TYPE="{type:-feature_request}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
DATE_PREFIX=$(date +%Y%m%d)

echo "📝 Collecting feedback..."

# Parse message link
# Format: https://slack.com/archives/CHANNEL_ID/pTIMESTAMP
CHANNEL_ID=$(echo "$MESSAGE_LINK" | sed 's|.*/archives/\([^/]*\)/.*|\1|')
MESSAGE_TS=$(echo "$MESSAGE_LINK" | sed 's|.*/p\([0-9]*\).*|\1|' | sed 's/\(.\{10\}\)/\1./')

if [ -z "$CHANNEL_ID" ] || [ -z "$MESSAGE_TS" ]; then
    echo "❌ Invalid message link format"
    echo "Expected: https://slack.com/archives/CHANNEL_ID/pTIMESTAMP"
    exit 1
fi

# Generate feedback ID
FEEDBACK_DIR=".planning/community/feedback"
mkdir -p "$FEEDBACK_DIR"
FEEDBACK_COUNT=$(ls "$FEEDBACK_DIR" | wc -l)
FEEDBACK_ID="FB-${DATE_PREFIX}-$(printf '%04d' $((FEEDBACK_COUNT + 1)))"

# Create feedback record
FEEDBACK_FILE="${FEEDBACK_DIR}/${FEEDBACK_ID}.md"

cat > "$FEEDBACK_FILE" << EOF
# Feedback: ${FEEDBACK_ID}

**Type:** ${FEEDBACK_TYPE}
**Channel:** ${CHANNEL_ID}
**Message Link:** ${MESSAGE_LINK}
**Collected:** ${TIMESTAMP}
**Status:** pending_triage

---

## Original Message

> (Message content to be fetched from Slack)

---

## Triage

**Status:** ⏳ Pending
**Reviewer:** _pending_
**Focus Group:** _pending_
**Decision Date:** _pending_

---

## Attribution

**Original Author:** _pending_ (community member)
**Contributor Invited:** [ ] Not yet

---

*Collected via WGSD feedback system*
EOF

echo ""
echo "✅ Feedback recorded: ${FEEDBACK_ID}"
echo ""
echo "📋 Next steps:"
echo "   1. Review: /wgsd triage"
echo "   2. Move: /wgsd move-to-focus-group ${FEEDBACK_ID} [focus-group]"
echo ""

# Update triage queue
QUEUE_FILE="${FEEDBACK_DIR}/../TRIAGE-QUEUE.md"
if [ ! -f "$QUEUE_FILE" ]; then
    cat > "$QUEUE_FILE" << 'QEOF'
# Community Feedback Triage Queue

**Last Updated:** _timestamp_

## Pending Triage

| ID | Type | Collected | Status |
|----|------|-----------|--------|

## Recently Triaged

| ID | Type | Decision | Focus Group | Date |
|----|------|----------|-------------|------|

---
*Queue managed by WGSD feedback system*
QEOF
fi

# Add to queue
sed -i "s|^\| ID \| Type \| Collected \| Status \||& \n| ${FEEDBACK_ID} | ${FEEDBACK_TYPE} | ${TIMESTAMP} | ⏳ Pending |" "$QUEUE_FILE"
sed -i "s|Last Updated:.*|Last Updated: ${TIMESTAMP}|" "$QUEUE_FILE"

echo "📋 Added to triage queue"
```

**Post confirmation in thread:**
```markdown
✅ Feedback collected as **{FEEDBACK_ID}**

This feedback will be reviewed by our team and triaged to the appropriate focus group.

Type: {feedback_type}
Status: Pending triage
```

---

## Action: Show Triage Queue

Display pending feedback for triage:

```bash
echo "📋 Community Feedback Triage Queue"
echo ""

FEEDBACK_DIR=".planning/community/feedback"
QUEUE_FILE=".planning/community/TRIAGE-QUEUE.md"

if [ ! -d "$FEEDBACK_DIR" ]; then
    echo "No feedback collected yet."
    echo ""
    echo "💡 Collect feedback with:"
    echo "   /wgsd collect [message-link]"
    exit 0
fi

PENDING_COUNT=0
echo "## Pending Triage"
echo ""
echo "| ID | Type | Author | Collected |"
echo "|----|------|--------|-----------|"

for feedback_file in "$FEEDBACK_DIR"/*.md; do
    [ -f "$feedback_file" ] || continue
    
    FEEDBACK_ID=$(basename "$feedback_file" .md)
    STATUS=$(grep "^**Status:**" "$feedback_file" 2>/dev/null | head -1 | sed 's/.*: //')
    
    if [ "$STATUS" = "pending_triage" ] || [ "$STATUS" = "⏳ Pending" ]; then
        TYPE=$(grep "^**Type:**" "$feedback_file" | sed 's/.*: //')
        COLLECTED=$(grep "^**Collected:**" "$feedback_file" | sed 's/.*: //')
        AUTHOR=$(grep "^**Original Author:**" "$feedback_file" | sed 's/.*: //' | head -c 20)
        
        echo "| ${FEEDBACK_ID} | ${TYPE} | ${AUTHOR:-unknown} | ${COLLECTED:0:10} |"
        PENDING_COUNT=$((PENDING_COUNT + 1))
    fi
done

echo ""
echo "**Total pending:** ${PENDING_COUNT}"
echo ""

if [ "$PENDING_COUNT" -gt 0 ]; then
    echo "💡 Actions:"
    echo "   /wgsd move-to-focus-group [ID] [focus-group]"
    echo "   /wgsd acknowledge [ID]"
fi
```

---

## Action: Move to Focus Group

Triage feedback to a focus group, creating a concept:

```bash
FEEDBACK_ID="{feedback-id}"
FOCUS_GROUP="{focus-group}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📤 Moving feedback to focus group..."

FEEDBACK_FILE=".planning/community/feedback/${FEEDBACK_ID}.md"
if [ ! -f "$FEEDBACK_FILE" ]; then
    echo "❌ Feedback not found: ${FEEDBACK_ID}"
    exit 1
fi

# Verify focus group exists
FG_DIR=".planning/focus-groups/${FOCUS_GROUP}"
if [ ! -d "$FG_DIR" ]; then
    echo "❌ Focus group not found: ${FOCUS_GROUP}"
    echo ""
    echo "💡 Available focus groups:"
    ls -1 .planning/focus-groups/ 2>/dev/null
    exit 1
fi

# Extract feedback details
ORIGINAL_LINK=$(grep "^**Message Link:**" "$FEEDBACK_FILE" | sed 's/.*: //')
FEEDBACK_TYPE=$(grep "^**Type:**" "$FEEDBACK_FILE" | sed 's/.*: //')
AUTHOR=$(grep "^**Original Author:**" "$FEEDBACK_FILE" | sed 's/.*: //')

# Generate concept name from feedback ID
CONCEPT_NAME=$(echo "$FEEDBACK_ID" | tr '[:upper:]' '[:lower:]' | tr '-' '_')

# Create concept from feedback
CONCEPT_DIR="${FG_DIR}/concepts"
mkdir -p "$CONCEPT_DIR"
CONCEPT_FILE="${CONCEPT_DIR}/${CONCEPT_NAME}.md"

cat > "$CONCEPT_FILE" << EOF
# Concept: ${CONCEPT_NAME}

**Focus Group:** ${FOCUS_GROUP}
**Status:** 📝 Draft
**Created:** ${TIMESTAMP}
**Source:** Community Feedback (${FEEDBACK_ID})

---

## Origin

**From:** Community channel
**Original Author:** ${AUTHOR:-Unknown community member}
**Feedback Type:** ${FEEDBACK_TYPE}
**Original Link:** ${ORIGINAL_LINK}

---

## Description

> (Content from original community feedback)

---

## Discussion

_Add discussion notes as the concept develops._

---

## Community Attribution

This concept originated from community feedback.

**Original Contributor:** ${AUTHOR:-Unknown}
**Feedback ID:** ${FEEDBACK_ID}
**Contribution Type:** Original idea

---

*Created via WGSD feedback triage*
EOF

echo "✅ Created concept: ${CONCEPT_NAME}"

# Update feedback status
sed -i "s|^**Status:**.*|**Status:** ✅ Triaged|" "$FEEDBACK_FILE"
sed -i "s|**Focus Group:** _pending_|**Focus Group:** ${FOCUS_GROUP}|" "$FEEDBACK_FILE"
sed -i "s|**Decision Date:** _pending_|**Decision Date:** ${TIMESTAMP}|" "$FEEDBACK_FILE"

# Add triage decision section
cat >> "$FEEDBACK_FILE" << EOF

## Triage Decision

**Decision:** Move to focus group
**Focus Group:** ${FOCUS_GROUP}
**Concept Created:** ${CONCEPT_NAME}
**Triaged By:** (moderator)
**Triaged At:** ${TIMESTAMP}
EOF

echo ""
echo "📋 Feedback ${FEEDBACK_ID} → Focus Group: ${FOCUS_GROUP}"
echo "   Concept created: ${CONCEPT_NAME}"
echo ""
echo "💡 Next steps:"
echo "   1. Invite contributor: /wgsd invite-contributor ${AUTHOR} ${FOCUS_GROUP}"
echo "   2. Acknowledge in community: /wgsd acknowledge ${FEEDBACK_ID}"
```

**Actions to take:**
1. Post to focus group channel about new community-sourced concept
2. Update focus group canvas with new concept
3. Offer to invite original contributor

---

## Action: Acknowledge Feedback

Post acknowledgment to community about triage decision:

```bash
FEEDBACK_ID="{feedback-id}"

echo "📢 Acknowledging feedback..."

FEEDBACK_FILE=".planning/community/feedback/${FEEDBACK_ID}.md"
if [ ! -f "$FEEDBACK_FILE" ]; then
    echo "❌ Feedback not found: ${FEEDBACK_ID}"
    exit 1
fi

STATUS=$(grep "^**Status:**" "$FEEDBACK_FILE" | sed 's/.*: //')
if [ "$STATUS" != "✅ Triaged" ]; then
    echo "⚠️ Feedback not yet triaged (Status: ${STATUS})"
    echo ""
    echo "💡 Triage first with:"
    echo "   /wgsd move-to-focus-group ${FEEDBACK_ID} [focus-group]"
    exit 1
fi

FOCUS_GROUP=$(grep "^**Focus Group:**" "$FEEDBACK_FILE" | sed 's/.*: //' | grep -v "_pending_")
ORIGINAL_LINK=$(grep "^**Message Link:**" "$FEEDBACK_FILE" | sed 's/.*: //')
AUTHOR=$(grep "^**Original Author:**" "$FEEDBACK_FILE" | sed 's/.*: //')

echo ""
echo "Ready to post acknowledgment:"
echo ""
echo "───────────────────────────────────────"
cat << EOF
Thanks for your feedback, ${AUTHOR}! 🙏

We've reviewed your suggestion and it's now being explored by our **${FOCUS_GROUP}** team.

📊 **Status:** Under active consideration
🎯 **Focus Group:** ${FOCUS_GROUP}

We appreciate your contribution to making the product better! If you're interested in participating in the development discussion, let us know and we can invite you to the focus group.
EOF
echo "───────────────────────────────────────"
echo ""
echo "💡 Post this in reply to the original message"
echo "   Original: ${ORIGINAL_LINK}"

# Mark as acknowledged
sed -i "s|^**Status:** ✅ Triaged|**Status:** ✅ Acknowledged|" "$FEEDBACK_FILE"
```

**Post to community channel as reply to original message.**

---

## Action: Smart Triage Suggestion

Suggest focus group based on feedback content:

```bash
MESSAGE_CONTENT="{message-content}"

echo "🤖 Analyzing feedback for focus group suggestion..."
echo ""

# Keyword-based routing
suggest_focus_group() {
    local content="$1"
    content=$(echo "$content" | tr '[:upper:]' '[:lower:]')
    
    # Check keywords
    if echo "$content" | grep -qE "auth|login|sso|password|2fa|security|permission"; then
        echo "security"
    elif echo "$content" | grep -qE "signup|onboard|getting started|tutorial|new user|welcome"; then
        echo "onboarding"
    elif echo "$content" | grep -qE "payment|invoice|billing|subscription|charge|price"; then
        echo "billing"
    elif echo "$content" | grep -qE "api|integration|webhook|rest|graphql|sdk"; then
        echo "api"
    elif echo "$content" | grep -qE "slow|fast|performance|speed|load|optimize"; then
        echo "performance"
    elif echo "$content" | grep -qE "ui|ux|design|button|layout|interface"; then
        echo "design"
    else
        echo "general"
    fi
}

SUGGESTED_FG=$(suggest_focus_group "$MESSAGE_CONTENT")

echo "📊 Analysis Results:"
echo ""
echo "   Suggested Focus Group: **${SUGGESTED_FG}**"
echo ""
echo "   Keywords detected:"
echo "$MESSAGE_CONTENT" | grep -oE "auth|login|sso|password|signup|onboard|payment|billing|api|integration|performance|speed|ui|design" | sort -u | sed 's/^/      - /'
echo ""

# List available focus groups
echo "📋 Available Focus Groups:"
for fg in .planning/focus-groups/*/; do
    FG_NAME=$(basename "$fg")
    CONCEPT_COUNT=$(ls "$fg/concepts/" 2>/dev/null | wc -l)
    echo "   • ${FG_NAME} (${CONCEPT_COUNT} concepts)"
done
```

---

## Action: Bulk Triage View

Show feedback categorized by suggested focus group:

```bash
echo "📊 Feedback by Suggested Focus Group"
echo ""

FEEDBACK_DIR=".planning/community/feedback"

# Group by suggestion
declare -A BY_FG

for feedback_file in "$FEEDBACK_DIR"/*.md; do
    [ -f "$feedback_file" ] || continue
    
    STATUS=$(grep "^**Status:**" "$feedback_file" | sed 's/.*: //' | head -1)
    [ "$STATUS" = "pending_triage" ] || continue
    
    FEEDBACK_ID=$(basename "$feedback_file" .md)
    TYPE=$(grep "^**Type:**" "$feedback_file" | sed 's/.*: //')
    
    # Add to group (simplified - in real impl would analyze content)
    echo "| ${FEEDBACK_ID} | ${TYPE} | general |"
done

echo ""
echo "💡 Quick triage:"
echo "   /wgsd move-to-focus-group [ID] [focus-group]"
```

</process>

<routing>
| Action | Description |
|--------|-------------|
| collect [link] | Record message as feedback |
| triage, queue | Show triage queue |
| move-to-focus-group [id] [fg] | Move feedback to focus group |
| acknowledge [id] | Post acknowledgment |
| suggest [content] | Suggest focus group |
| bulk | Show grouped triage view |
</routing>

<feedback_types>
| Emoji | Type | Description |
|-------|------|-------------|
| 💡 | feature_request | New feature idea |
| 🐛 | bug_report | Something broken |
| ✨ | improvement | Enhancement to existing |
| ❓ | question | How does X work? |
| 🎉 | praise | Positive feedback |
</feedback_types>

<success_criteria>
- Feedback captured with full context and link
- Triage queue provides clear view of pending items
- Move command creates concept with attribution
- Community receives acknowledgment
- Focus group suggestions are relevant
</success_criteria>
