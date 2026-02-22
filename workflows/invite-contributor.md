---
name: wgsd:invite-contributor
description: Invite community contributors to private focus group channels
argument-hint: "[invite|status|pending] [@user] [focus-group]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Manage community contributor invitations to private development channels:
- Auto-invite when feedback becomes a concept
- Manual invitation for engaged community members
- Track invitation status and responses
- Maintain contributor records for attribution
</objective>

<libraries>
- workflows/lib/slack-api.md - Channel invitations
- workflows/lib/channel-registry.md - Channel lookups
</libraries>

<process>

## Action: Invite Contributor

Send invitation to a community member:

```bash
USERNAME="{username}"
FOCUS_GROUP="{focus-group}"
REASON="{reason:-community feedback}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📨 Inviting contributor to focus group..."

# Clean username (remove @)
USERNAME=$(echo "$USERNAME" | sed 's/@//')

# Verify focus group exists
FG_DIR=".planning/focus-groups/${FOCUS_GROUP}"
if [ ! -d "$FG_DIR" ]; then
    echo "❌ Focus group not found: ${FOCUS_GROUP}"
    exit 1
fi

# Get channel info
CHANNELS_FILE="${FG_DIR}/CHANNELS.md"
STUB=$(grep "^Stub:" "$CHANNELS_FILE" 2>/dev/null | sed 's/Stub: //' || echo "proj")
FG_CHANNEL="${STUB}-fg-${FOCUS_GROUP}"

# Create invitation record
INVITE_DIR=".planning/community/invitations"
mkdir -p "$INVITE_DIR"
INVITE_ID="INV-$(date +%Y%m%d)-$(echo "$USERNAME" | head -c 8)"
INVITE_FILE="${INVITE_DIR}/${INVITE_ID}.md"

cat > "$INVITE_FILE" << EOF
# Invitation: ${INVITE_ID}

**Invitee:** @${USERNAME}
**Focus Group:** ${FOCUS_GROUP}
**Channel:** #${FG_CHANNEL}
**Reason:** ${REASON}

**Invited At:** ${TIMESTAMP}
**Status:** 📨 Pending
**Response:** _awaiting_

---

## Invitation Message

Hi @${USERNAME}! 👋

Your contributions in our community channel caught our attention!

We'd love to invite you to join our **${FOCUS_GROUP}** focus group, where our team is actively developing features based on community feedback like yours.

**What you'll get:**
- Early access to development discussions
- Opportunity to shape features
- Direct input on implementation
- Attribution in our roadmap

**Channel:** #${FG_CHANNEL}

Would you like to join? Reply with:
- ✅ "yes" or "accept" to join
- ❌ "no thanks" to decline

---

*Invited via WGSD contributor system*
EOF

echo ""
echo "✅ Invitation created: ${INVITE_ID}"
echo ""
echo "📨 Send this DM to @${USERNAME}:"
echo ""
echo "───────────────────────────────────────"
cat << MSG
Hi @${USERNAME}! 👋

Your contributions in our community channel caught our attention!

We'd love to invite you to join our **${FOCUS_GROUP}** focus group, where our team is actively developing features based on community feedback like yours.

**What you'll get:**
• Early access to development discussions
• Opportunity to shape features  
• Direct input on implementation
• Attribution in our roadmap

Would you like to join? Just reply "yes" or "no thanks" 🙂
MSG
echo "───────────────────────────────────────"
echo ""
echo "💡 After user responds, update with:"
echo "   /wgsd invitation accept ${INVITE_ID}"
echo "   /wgsd invitation decline ${INVITE_ID}"
```

---

## Action: Auto-Invite from Feedback

Automatically trigger invitation when feedback becomes concept:

```bash
FEEDBACK_ID="{feedback-id}"
FOCUS_GROUP="{focus-group}"

echo "🤖 Processing auto-invite from feedback..."

FEEDBACK_FILE=".planning/community/feedback/${FEEDBACK_ID}.md"
if [ ! -f "$FEEDBACK_FILE" ]; then
    echo "❌ Feedback not found: ${FEEDBACK_ID}"
    exit 1
fi

# Get author from feedback
AUTHOR=$(grep "^**Original Author:**" "$FEEDBACK_FILE" | sed 's/.*: //' | sed 's/@//')

if [ -z "$AUTHOR" ] || [ "$AUTHOR" = "_pending_" ] || [ "$AUTHOR" = "Unknown" ]; then
    echo "⚠️ No author information in feedback"
    echo ""
    echo "💡 Manually invite with:"
    echo "   /wgsd invite-contributor @username ${FOCUS_GROUP}"
    exit 0
fi

# Check if already invited
INVITE_DIR=".planning/community/invitations"
EXISTING=$(grep -l "@${AUTHOR}" "$INVITE_DIR"/*.md 2>/dev/null | \
    xargs grep -l "Focus Group:** ${FOCUS_GROUP}" 2>/dev/null | head -1)

if [ -n "$EXISTING" ]; then
    EXISTING_ID=$(basename "$EXISTING" .md)
    EXISTING_STATUS=$(grep "^**Status:**" "$EXISTING" | sed 's/.*: //')
    echo "ℹ️ Already invited: ${EXISTING_ID} (${EXISTING_STATUS})"
    exit 0
fi

echo ""
echo "📋 Ready to invite @${AUTHOR} to ${FOCUS_GROUP}"
echo ""
echo "💡 Send invitation with:"
echo "   /wgsd invite-contributor @${AUTHOR} ${FOCUS_GROUP}"
```

---

## Action: Process Invitation Response

Handle accept or decline:

```bash
ACTION="{accept|decline}"
INVITE_ID="{invite-id}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

INVITE_FILE=".planning/community/invitations/${INVITE_ID}.md"
if [ ! -f "$INVITE_FILE" ]; then
    echo "❌ Invitation not found: ${INVITE_ID}"
    exit 1
fi

USERNAME=$(grep "^**Invitee:**" "$INVITE_FILE" | sed 's/.*: @*//')
FOCUS_GROUP=$(grep "^**Focus Group:**" "$INVITE_FILE" | sed 's/.*: //')
FG_CHANNEL=$(grep "^**Channel:**" "$INVITE_FILE" | sed 's/.*: #//')

if [ "$ACTION" = "accept" ]; then
    echo "✅ Processing acceptance..."
    
    # Update invitation record
    sed -i "s|^**Status:**.*|**Status:** ✅ Accepted|" "$INVITE_FILE"
    sed -i "s|^**Response:**.*|**Response:** Accepted at ${TIMESTAMP}|" "$INVITE_FILE"
    
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
    
    # Add user to contributors
    REASON=$(grep "^**Reason:**" "$INVITE_FILE" | sed 's/.*: //')
    echo "| @${USERNAME} | ${TIMESTAMP:0:10} | ${REASON} |" >> "$CONTRIB_FILE"
    
    echo ""
    echo "✅ @${USERNAME} accepted invitation!"
    echo ""
    echo "📋 Actions needed:"
    echo "   1. Add @${USERNAME} to #${FG_CHANNEL} via Slack"
    echo "   2. Post welcome message in channel"
    echo ""
    echo "📝 Welcome message:"
    echo ""
    echo "───────────────────────────────────────"
    cat << MSG
👋 Welcome @${USERNAME} to the ${FOCUS_GROUP} focus group!

${USERNAME} is joining us from the community based on their valuable feedback about ${REASON}.

Looking forward to your contributions! Feel free to jump into any ongoing discussions.
MSG
    echo "───────────────────────────────────────"
    
elif [ "$ACTION" = "decline" ]; then
    echo "📝 Processing decline..."
    
    # Update invitation record
    sed -i "s|^**Status:**.*|**Status:** ❌ Declined|" "$INVITE_FILE"
    sed -i "s|^**Response:**.*|**Response:** Declined at ${TIMESTAMP}|" "$INVITE_FILE"
    
    echo ""
    echo "✅ Recorded decline from @${USERNAME}"
    echo ""
    echo "📝 Optional response to send:"
    echo ""
    echo "───────────────────────────────────────"
    cat << MSG
No problem at all! Thanks for letting us know.

Your feedback is still valuable and we'll continue developing based on your suggestions. You can always reach out if you change your mind!
MSG
    echo "───────────────────────────────────────"
fi
```

---

## Action: Show Pending Invitations

List all pending invitations:

```bash
echo "📨 Pending Invitations"
echo ""

INVITE_DIR=".planning/community/invitations"

if [ ! -d "$INVITE_DIR" ] || [ -z "$(ls -A "$INVITE_DIR" 2>/dev/null)" ]; then
    echo "No invitations found."
    exit 0
fi

echo "| ID | User | Focus Group | Invited | Status |"
echo "|----|------|-------------|---------|--------|"

PENDING_COUNT=0
for invite_file in "$INVITE_DIR"/*.md; do
    [ -f "$invite_file" ] || continue
    
    INVITE_ID=$(basename "$invite_file" .md)
    STATUS=$(grep "^**Status:**" "$invite_file" | sed 's/.*: //')
    
    if echo "$STATUS" | grep -q "Pending"; then
        USERNAME=$(grep "^**Invitee:**" "$invite_file" | sed 's/.*: //')
        FOCUS_GROUP=$(grep "^**Focus Group:**" "$invite_file" | sed 's/.*: //')
        INVITED_AT=$(grep "^**Invited At:**" "$invite_file" | sed 's/.*: //')
        
        echo "| ${INVITE_ID} | ${USERNAME} | ${FOCUS_GROUP} | ${INVITED_AT:0:10} | 📨 Pending |"
        PENDING_COUNT=$((PENDING_COUNT + 1))
    fi
done

echo ""
echo "**Pending:** ${PENDING_COUNT}"
echo ""

if [ "$PENDING_COUNT" -gt 0 ]; then
    echo "💡 When user responds:"
    echo "   /wgsd invitation accept [ID]"
    echo "   /wgsd invitation decline [ID]"
fi
```

---

## Action: Invitation Status

Check status of a specific user:

```bash
USERNAME="{username}"

USERNAME=$(echo "$USERNAME" | sed 's/@//')

echo "📊 Invitation Status: @${USERNAME}"
echo ""

INVITE_DIR=".planning/community/invitations"

FOUND=0
for invite_file in "$INVITE_DIR"/*.md; do
    [ -f "$invite_file" ] || continue
    
    if grep -q "Invitee:.*${USERNAME}" "$invite_file"; then
        INVITE_ID=$(basename "$invite_file" .md)
        STATUS=$(grep "^**Status:**" "$invite_file" | sed 's/.*: //')
        FOCUS_GROUP=$(grep "^**Focus Group:**" "$invite_file" | sed 's/.*: //')
        INVITED_AT=$(grep "^**Invited At:**" "$invite_file" | sed 's/.*: //')
        
        echo "**${INVITE_ID}**"
        echo "  Focus Group: ${FOCUS_GROUP}"
        echo "  Status: ${STATUS}"
        echo "  Invited: ${INVITED_AT}"
        echo ""
        FOUND=1
    fi
done

if [ "$FOUND" -eq 0 ]; then
    echo "No invitations found for @${USERNAME}"
    echo ""
    echo "💡 Send invitation:"
    echo "   /wgsd invite-contributor @${USERNAME} [focus-group]"
fi
```

---

## Action: List Contributors

Show all community contributors across focus groups:

```bash
echo "👥 Community Contributors"
echo ""

for fg_dir in .planning/focus-groups/*/; do
    [ -d "$fg_dir" ] || continue
    
    FG_NAME=$(basename "$fg_dir")
    CONTRIB_FILE="${fg_dir}CONTRIBUTORS.md"
    
    if [ -f "$CONTRIB_FILE" ]; then
        CONTRIB_COUNT=$(grep -c "^| @" "$CONTRIB_FILE" 2>/dev/null || echo "0")
        
        if [ "$CONTRIB_COUNT" -gt 0 ]; then
            echo "## ${FG_NAME} (${CONTRIB_COUNT} community contributors)"
            echo ""
            grep "^| @" "$CONTRIB_FILE" | while read -r line; do
                USER=$(echo "$line" | awk -F'|' '{print $2}' | xargs)
                JOINED=$(echo "$line" | awk -F'|' '{print $3}' | xargs)
                CONTRIB=$(echo "$line" | awk -F'|' '{print $4}' | xargs)
                echo "  • ${USER} - ${CONTRIB} (${JOINED})"
            done
            echo ""
        fi
    fi
done
```

</process>

<routing>
| Action | Description |
|--------|-------------|
| invite-contributor @user [fg] | Send invitation |
| auto-invite [feedback-id] [fg] | Process auto-invite |
| accept [invite-id] | Process acceptance |
| decline [invite-id] | Process decline |
| pending | Show pending invitations |
| status @user | Check user's invitation status |
| contributors | List all community contributors |
</routing>

<success_criteria>
- Invitations sent with clear value proposition
- Acceptance adds user to focus group channel
- Contributors tracked with attribution
- Decline handled gracefully
- Pending invitations visible for follow-up
</success_criteria>
