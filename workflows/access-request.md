---
name: wgsd:access-request
description: Manage community access requests to private focus group channels
argument-hint: "[request|review|approve|deny] [args]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Enable community members to request access to focus groups:
- Submit access requests with reason
- Review queue for focus group owners
- Approval workflow with channel invitation
- Denial with explanation
- Track request history
</objective>

<libraries>
- workflows/lib/slack-api.md - Channel operations
- workflows/lib/channel-registry.md - Channel lookups
</libraries>

<process>

## Action: Request Access

Community member requests access to a focus group:

```bash
FOCUS_GROUP="{focus-group}"
USERNAME="{requesting-user}"
REASON="{reason}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📝 Submitting access request..."

# Clean username
USERNAME=$(echo "$USERNAME" | sed 's/@//')

# Verify focus group exists
FG_DIR=".planning/focus-groups/${FOCUS_GROUP}"
if [ ! -d "$FG_DIR" ]; then
    echo "❌ Focus group not found: ${FOCUS_GROUP}"
    echo ""
    echo "💡 Available focus groups:"
    for fg in .planning/focus-groups/*/; do
        FG_NAME=$(basename "$fg")
        echo "   • ${FG_NAME}"
    done
    exit 1
fi

# Check if already a member or has pending request
REQUEST_DIR=".planning/community/access-requests"
mkdir -p "$REQUEST_DIR"

EXISTING=$(grep -l "Requester:.*${USERNAME}" "$REQUEST_DIR"/*.md 2>/dev/null | \
    xargs grep -l "Focus Group:** ${FOCUS_GROUP}" 2>/dev/null | head -1)

if [ -n "$EXISTING" ]; then
    EXISTING_STATUS=$(grep "^**Status:**" "$EXISTING" | sed 's/.*: //')
    echo "ℹ️ Existing request found: $(basename "$EXISTING" .md)"
    echo "   Status: ${EXISTING_STATUS}"
    exit 0
fi

# Generate request ID
REQUEST_COUNT=$(ls "$REQUEST_DIR" 2>/dev/null | wc -l)
REQUEST_ID="REQ-$(date +%Y%m%d)-$(printf '%04d' $((REQUEST_COUNT + 1)))"
REQUEST_FILE="${REQUEST_DIR}/${REQUEST_ID}.md"

# Get community contribution stats (simplified)
FEEDBACK_COUNT=$(grep -rl "@${USERNAME}" .planning/community/feedback/ 2>/dev/null | wc -l)

cat > "$REQUEST_FILE" << EOF
# Access Request: ${REQUEST_ID}

**Requester:** @${USERNAME}
**Focus Group:** ${FOCUS_GROUP}
**Requested At:** ${TIMESTAMP}
**Status:** ⏳ Pending Review

---

## Request Details

### Why are you interested in joining?
${REASON}

---

## Community Engagement

| Metric | Count |
|--------|-------|
| Feedback submitted | ${FEEDBACK_COUNT} |
| Messages in community | _unknown_ |
| Ideas adopted | _check feedback records_ |

---

## Review

**Reviewer:** _pending_
**Decision:** _pending_
**Decision Date:** _pending_
**Notes:** _pending_

---

*Request submitted via WGSD access request system*
EOF

echo ""
echo "✅ Access request submitted: ${REQUEST_ID}"
echo ""
echo "📋 Request Details:"
echo "   Focus Group: ${FOCUS_GROUP}"
echo "   Status: Pending Review"
echo ""
echo "📢 Focus group owners will review your request."
echo "   You'll receive a notification once a decision is made."

# Notify focus group owners
OWNER_FILE="${FG_DIR}/OWNERS.md"
if [ -f "$OWNER_FILE" ]; then
    echo ""
    echo "📨 Notifying focus group owners..."
fi
```

**Notify focus group channel:**
```markdown
📝 **New Access Request**

**From:** @{username}
**Request ID:** {REQUEST_ID}

**Why they want to join:**
> {reason}

**Community Engagement:** {FEEDBACK_COUNT} feedback items submitted

To review:
- `/wgsd approve-access {REQUEST_ID}`
- `/wgsd deny-access {REQUEST_ID} [reason]`
```

---

## Action: Review Pending Requests

Show pending access requests for review:

```bash
FOCUS_GROUP="{focus-group:-all}"

echo "📋 Pending Access Requests"
echo ""

REQUEST_DIR=".planning/community/access-requests"

if [ ! -d "$REQUEST_DIR" ] || [ -z "$(ls -A "$REQUEST_DIR" 2>/dev/null)" ]; then
    echo "No access requests found."
    exit 0
fi

echo "| ID | Requester | Focus Group | Requested | Status |"
echo "|----|-----------|-------------|-----------|--------|"

PENDING_COUNT=0
for request_file in "$REQUEST_DIR"/*.md; do
    [ -f "$request_file" ] || continue
    
    STATUS=$(grep "^**Status:**" "$request_file" | sed 's/.*: //')
    
    if echo "$STATUS" | grep -q "Pending"; then
        REQUEST_ID=$(basename "$request_file" .md)
        USERNAME=$(grep "^**Requester:**" "$request_file" | sed 's/.*: //')
        FG=$(grep "^**Focus Group:**" "$request_file" | sed 's/.*: //')
        REQUESTED_AT=$(grep "^**Requested At:**" "$request_file" | sed 's/.*: //')
        
        # Filter by focus group if specified
        if [ "$FOCUS_GROUP" != "all" ] && [ "$FG" != "$FOCUS_GROUP" ]; then
            continue
        fi
        
        echo "| ${REQUEST_ID} | ${USERNAME} | ${FG} | ${REQUESTED_AT:0:10} | ⏳ Pending |"
        PENDING_COUNT=$((PENDING_COUNT + 1))
    fi
done

echo ""
echo "**Pending:** ${PENDING_COUNT}"
echo ""

if [ "$PENDING_COUNT" -gt 0 ]; then
    echo "💡 Actions:"
    echo "   /wgsd approve-access [ID]"
    echo "   /wgsd deny-access [ID] [reason]"
    echo "   /wgsd view-request [ID]"
fi
```

---

## Action: View Request Details

Show full details of a request:

```bash
REQUEST_ID="{request-id}"

echo "📋 Access Request Details: ${REQUEST_ID}"
echo ""

REQUEST_FILE=".planning/community/access-requests/${REQUEST_ID}.md"
if [ ! -f "$REQUEST_FILE" ]; then
    echo "❌ Request not found: ${REQUEST_ID}"
    exit 1
fi

cat "$REQUEST_FILE"
```

---

## Action: Approve Access Request

Approve and invite user to focus group:

```bash
REQUEST_ID="{request-id}"
REVIEWER="{reviewer:-moderator}"
NOTES="{notes:-Approved based on community engagement}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "✅ Approving access request..."

REQUEST_FILE=".planning/community/access-requests/${REQUEST_ID}.md"
if [ ! -f "$REQUEST_FILE" ]; then
    echo "❌ Request not found: ${REQUEST_ID}"
    exit 1
fi

USERNAME=$(grep "^**Requester:**" "$REQUEST_FILE" | sed 's/.*: @*//')
FOCUS_GROUP=$(grep "^**Focus Group:**" "$REQUEST_FILE" | sed 's/.*: //')

# Update request status
sed -i "s|^**Status:**.*|**Status:** ✅ Approved|" "$REQUEST_FILE"
sed -i "s|^**Reviewer:**.*|**Reviewer:** @${REVIEWER}|" "$REQUEST_FILE"
sed -i "s|^**Decision:**.*|**Decision:** Approved|" "$REQUEST_FILE"
sed -i "s|^**Decision Date:**.*|**Decision Date:** ${TIMESTAMP}|" "$REQUEST_FILE"
sed -i "s|^**Notes:**.*|**Notes:** ${NOTES}|" "$REQUEST_FILE"

# Get channel info
CHANNELS_FILE=".planning/focus-groups/${FOCUS_GROUP}/CHANNELS.md"
STUB=$(grep "^Stub:" "$CHANNELS_FILE" 2>/dev/null | sed 's/Stub: //' || echo "proj")
FG_CHANNEL="${STUB}-fg-${FOCUS_GROUP}"

# Add to contributors
CONTRIB_FILE=".planning/focus-groups/${FOCUS_GROUP}/CONTRIBUTORS.md"
if [ ! -f "$CONTRIB_FILE" ]; then
    cat > "$CONTRIB_FILE" << EOF
# ${FOCUS_GROUP} Focus Group Contributors

## Core Team
_(Focus group owners and maintainers)_

## Community Contributors
| User | Joined | Contribution |
|------|--------|--------------|
EOF
fi

echo "| @${USERNAME} | ${TIMESTAMP:0:10} | Access request approved |" >> "$CONTRIB_FILE"

echo ""
echo "✅ Access request approved!"
echo ""
echo "📋 Actions needed:"
echo "   1. Invite @${USERNAME} to #${FG_CHANNEL} via Slack"
echo ""
echo "📨 Notification to send to @${USERNAME}:"
echo ""
echo "───────────────────────────────────────"
cat << MSG
🎉 Good news! Your access request has been approved!

You've been granted access to the **${FOCUS_GROUP}** focus group.

**Channel:** #${FG_CHANNEL}

You should now be able to join the channel and participate in discussions.

Welcome aboard! 🚀
MSG
echo "───────────────────────────────────────"
```

**Actions to perform:**
1. Add user to focus group channel via Slack API
2. Send approval notification to user
3. Post welcome message in focus group channel

---

## Action: Deny Access Request

Deny request with explanation:

```bash
REQUEST_ID="{request-id}"
REASON="{reason}"
REVIEWER="{reviewer:-moderator}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "❌ Denying access request..."

REQUEST_FILE=".planning/community/access-requests/${REQUEST_ID}.md"
if [ ! -f "$REQUEST_FILE" ]; then
    echo "❌ Request not found: ${REQUEST_ID}"
    exit 1
fi

if [ -z "$REASON" ]; then
    echo "⚠️ Please provide a reason for denial"
    echo ""
    echo "Usage: /wgsd deny-access ${REQUEST_ID} \"reason\""
    exit 1
fi

USERNAME=$(grep "^**Requester:**" "$REQUEST_FILE" | sed 's/.*: @*//')
FOCUS_GROUP=$(grep "^**Focus Group:**" "$REQUEST_FILE" | sed 's/.*: //')

# Update request status
sed -i "s|^**Status:**.*|**Status:** ❌ Denied|" "$REQUEST_FILE"
sed -i "s|^**Reviewer:**.*|**Reviewer:** @${REVIEWER}|" "$REQUEST_FILE"
sed -i "s|^**Decision:**.*|**Decision:** Denied|" "$REQUEST_FILE"
sed -i "s|^**Decision Date:**.*|**Decision Date:** ${TIMESTAMP}|" "$REQUEST_FILE"
sed -i "s|^**Notes:**.*|**Notes:** ${REASON}|" "$REQUEST_FILE"

echo ""
echo "❌ Access request denied"
echo ""
echo "📨 Notification to send to @${USERNAME}:"
echo ""
echo "───────────────────────────────────────"
cat << MSG
Hi @${USERNAME},

Thank you for your interest in joining the **${FOCUS_GROUP}** focus group.

After review, we're not able to approve your request at this time.

**Reason:** ${REASON}

This doesn't mean your contributions aren't valued! You can still:
• Continue participating in the community channel
• Submit feedback on features
• Request access again in the future

If you have questions, feel free to reach out.
MSG
echo "───────────────────────────────────────"
```

---

## Action: Request History

Show request history for a user:

```bash
USERNAME="{username}"

USERNAME=$(echo "$USERNAME" | sed 's/@//')

echo "📊 Access Request History: @${USERNAME}"
echo ""

REQUEST_DIR=".planning/community/access-requests"

echo "| ID | Focus Group | Status | Date |"
echo "|----|-------------|--------|------|"

FOUND=0
for request_file in "$REQUEST_DIR"/*.md; do
    [ -f "$request_file" ] || continue
    
    if grep -q "Requester:.*${USERNAME}" "$request_file"; then
        REQUEST_ID=$(basename "$request_file" .md)
        FG=$(grep "^**Focus Group:**" "$request_file" | sed 's/.*: //')
        STATUS=$(grep "^**Status:**" "$request_file" | sed 's/.*: //')
        DATE=$(grep "^**Requested At:**" "$request_file" | sed 's/.*: //')
        
        echo "| ${REQUEST_ID} | ${FG} | ${STATUS} | ${DATE:0:10} |"
        FOUND=1
    fi
done

if [ "$FOUND" -eq 0 ]; then
    echo "No access requests found for @${USERNAME}"
fi
```

---

## Action: Stats Overview

Show access request statistics:

```bash
echo "📊 Access Request Statistics"
echo ""

REQUEST_DIR=".planning/community/access-requests"

if [ ! -d "$REQUEST_DIR" ]; then
    echo "No access requests yet."
    exit 0
fi

TOTAL=0
PENDING=0
APPROVED=0
DENIED=0

for request_file in "$REQUEST_DIR"/*.md; do
    [ -f "$request_file" ] || continue
    
    TOTAL=$((TOTAL + 1))
    STATUS=$(grep "^**Status:**" "$request_file" | sed 's/.*: //')
    
    case "$STATUS" in
        *Pending*) PENDING=$((PENDING + 1)) ;;
        *Approved*) APPROVED=$((APPROVED + 1)) ;;
        *Denied*) DENIED=$((DENIED + 1)) ;;
    esac
done

echo "| Metric | Count |"
echo "|--------|-------|"
echo "| Total Requests | ${TOTAL} |"
echo "| ⏳ Pending | ${PENDING} |"
echo "| ✅ Approved | ${APPROVED} |"
echo "| ❌ Denied | ${DENIED} |"

if [ "$TOTAL" -gt 0 ]; then
    APPROVAL_RATE=$((APPROVED * 100 / TOTAL))
    echo ""
    echo "**Approval Rate:** ${APPROVAL_RATE}%"
fi

# Requests by focus group
echo ""
echo "### By Focus Group"
echo ""
echo "| Focus Group | Requests |"
echo "|-------------|----------|"

for fg in .planning/focus-groups/*/; do
    FG_NAME=$(basename "$fg")
    FG_COUNT=$(grep -l "Focus Group:** ${FG_NAME}" "$REQUEST_DIR"/*.md 2>/dev/null | wc -l)
    [ "$FG_COUNT" -gt 0 ] && echo "| ${FG_NAME} | ${FG_COUNT} |"
done
```

</process>

<routing>
| Action | Description |
|--------|-------------|
| request-access [fg] | Submit access request |
| review-requests [fg] | Show pending requests |
| view-request [id] | View request details |
| approve-access [id] | Approve request |
| deny-access [id] [reason] | Deny request |
| history @user | Show user's request history |
| stats | Show request statistics |
</routing>

<success_criteria>
- Community members can submit requests with context
- Focus group owners see pending requests
- Approval triggers channel invitation
- Denial provides clear explanation
- Request history maintained for audit
</success_criteria>
