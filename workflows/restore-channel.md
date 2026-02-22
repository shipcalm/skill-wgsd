---
name: wgsd:restore-channel
description: Restore an archived WGSD channel back to active state
argument-hint: "<channel-name>"
allowed-tools:
  - Exec
  - AskUser
---

<objective>
Restore a previously archived WGSD Slack channel:
1. Unarchive the channel in Slack
2. Update registry status to "active"
3. Re-register with OpenClaw if needed
4. Optionally reinvite previous members

**Implements:** CHANNEL-08 (lifecycle management)

**When to restore:**
- Implementation needs more work
- Concept revisited for further development
- Focus group reactivated
</objective>

<process>

## Step 1: Parse Arguments

```bash
echo "📦 WGSD Channel Restore"
echo "═══════════════════════════════════════════════════"
echo ""

CHANNEL_NAME="${1:-}"
WORKSPACE="${WORKSPACE:-.}"

if [ -z "$CHANNEL_NAME" ]; then
  echo "❌ Channel name is required"
  echo ""
  echo "Usage: restore-channel <channel-name>"
  echo ""
  echo "Examples:"
  echo "  restore-channel mvn-impl-auth-v2"
  echo "  restore-channel mvn-fg-billing"
  exit 1
fi

# Remove leading # if present
CHANNEL_NAME=$(echo "$CHANNEL_NAME" | sed 's/^#//')

echo "📌 Channel: #$CHANNEL_NAME"
echo ""
```

## Step 2: Get Channel ID

```bash
echo "🔍 Looking up channel..."

SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.channels.slack.botToken')

if [ "$SLACK_TOKEN" == "null" ] || [ -z "$SLACK_TOKEN" ]; then
  echo "❌ No Slack bot token found"
  exit 1
fi

# First try to get from registry
CONFIG_PATH="$WORKSPACE/.planning/WGSD-CONFIG.md"
CHANNEL_ID=""

if [ -f "$CONFIG_PATH" ]; then
  CHANNEL_ID=$(grep "| $CHANNEL_NAME |" "$CONFIG_PATH" | awk -F'|' '{print $3}' | tr -d ' ' | head -1)
fi

# If not in registry, search Slack (including archived)
if [ -z "$CHANNEL_ID" ]; then
  echo "   Not in registry, searching Slack..."
  
  # Note: Slack API includes archived channels in list if they exist
  CHANNEL_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel,private_channel&limit=500" \
    -H "Authorization: Bearer $SLACK_TOKEN" | \
    jq -r --arg name "$CHANNEL_NAME" '.channels[] | select(.name == $name) | .id')
fi

if [ -z "$CHANNEL_ID" ]; then
  echo "❌ Channel #$CHANNEL_NAME not found"
  echo ""
  echo "The channel may have been deleted (not just archived)."
  echo "Archived channels can be restored; deleted channels cannot."
  exit 1
fi

echo "✅ Found channel: $CHANNEL_ID"
echo ""
```

## Step 3: Check Current State

```bash
echo "🔍 Checking channel state..."

CHANNEL_INFO=$(curl -s -X GET "https://slack.com/api/conversations.info?channel=$CHANNEL_ID" \
  -H "Authorization: Bearer $SLACK_TOKEN")

API_OK=$(echo "$CHANNEL_INFO" | jq -r '.ok')

if [ "$API_OK" != "true" ]; then
  ERROR=$(echo "$CHANNEL_INFO" | jq -r '.error // "unknown"')
  if [ "$ERROR" = "channel_not_found" ]; then
    echo "❌ Channel was deleted (not archived)"
    echo "   Deleted channels cannot be restored."
    echo "   Consider creating a new channel with the same name."
    exit 1
  fi
  echo "❌ Failed to get channel info: $ERROR"
  exit 1
fi

IS_ARCHIVED=$(echo "$CHANNEL_INFO" | jq -r '.channel.is_archived')

if [ "$IS_ARCHIVED" = "false" ]; then
  echo "⚠️  Channel is already active"
  
  # Update registry if it shows archived
  if [ -f "$CONFIG_PATH" ] && grep -q "| $CHANNEL_NAME |.*| archived |" "$CONFIG_PATH"; then
    echo "📝 Updating registry to match Slack state..."
    sed -i "s/| $CHANNEL_NAME |\\(.*\\)| archived |/| $CHANNEL_NAME |\\1| active |/g" "$CONFIG_PATH"
    echo "✅ Registry updated"
  fi
  
  exit 0
fi

echo "✅ Channel is archived and can be restored"

# Get channel details
CHANNEL_TYPE=""
if echo "$CHANNEL_NAME" | grep -qE '-fg-'; then
  CHANNEL_TYPE="fg"
elif echo "$CHANNEL_NAME" | grep -qE '-cpt-'; then
  CHANNEL_TYPE="cpt"
elif echo "$CHANNEL_NAME" | grep -qE '-impl-'; then
  CHANNEL_TYPE="impl"
elif echo "$CHANNEL_NAME" | grep -qE '-dev$'; then
  CHANNEL_TYPE="dev"
elif echo "$CHANNEL_NAME" | grep -qE '-community$'; then
  CHANNEL_TYPE="community"
fi

echo "   Type: $CHANNEL_TYPE"
echo "   Topic: $(echo "$CHANNEL_INFO" | jq -r '.channel.topic.value // "none"')"
echo ""
```

## Step 4: Unarchive Channel

```bash
echo "📦 Restoring channel in Slack..."

UNARCHIVE_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.unarchive \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL_ID\"}")

UNARCHIVE_OK=$(echo "$UNARCHIVE_RESPONSE" | jq -r '.ok')
UNARCHIVE_ERROR=$(echo "$UNARCHIVE_RESPONSE" | jq -r '.error // empty')

if [ "$UNARCHIVE_OK" = "true" ]; then
  echo "✅ Channel unarchived successfully"
elif [ "$UNARCHIVE_ERROR" = "not_archived" ]; then
  echo "⚠️  Channel was not archived (already active)"
else
  echo "❌ Failed to unarchive channel"
  echo "   Error: $UNARCHIVE_ERROR"
  
  if [ "$UNARCHIVE_ERROR" = "restricted_action" ]; then
    echo ""
    echo "This may be due to workspace restrictions."
    echo "Contact workspace admin to unarchive the channel."
  fi
  
  exit 1
fi

echo ""
```

## Step 5: Update Registry

```bash
echo "📝 Updating channel registry..."

RESTORE_DATE=$(date +%Y-%m-%d)

if [ -f "$CONFIG_PATH" ]; then
  if grep -q "| $CHANNEL_NAME |" "$CONFIG_PATH"; then
    # Update status from archived to active
    sed -i "s/| $CHANNEL_NAME |\\(.*\\)| archived |/| $CHANNEL_NAME |\\1| active |/g" "$CONFIG_PATH"
    echo "✅ Registry updated: status → active"
  else
    echo "⚠️  Channel not in registry (adding as active)"
    RESTORE_LINE="| $CHANNEL_NAME | $CHANNEL_ID | $CHANNEL_TYPE | active | $RESTORE_DATE |"
    
    if grep -q "^|[-|]*|$" "$CONFIG_PATH"; then
      sed -i "/^|[-|]*|$/a $RESTORE_LINE" "$CONFIG_PATH"
      echo "✅ Added to registry as active"
    fi
  fi
else
  echo "⚠️  WGSD config not found - registry not updated"
fi

echo ""
```

## Step 6: Re-register with OpenClaw

```bash
echo "⚙️  Registering with OpenClaw..."

PATCH_JSON="{\"channels\":{\"slack\":{\"channels\":{\"$CHANNEL_ID\":{\"allow\":true,\"requireMention\":false}}}}}"

if command -v openclaw &> /dev/null; then
  openclaw gateway config.patch "$PATCH_JSON" 2>/dev/null
  if [ $? -eq 0 ]; then
    echo "✅ Channel registered with OpenClaw"
  else
    echo "⚠️  Auto-registration failed. Run manually:"
    echo "   openclaw gateway config.patch '$PATCH_JSON'"
  fi
else
  echo "📋 Run this command to enable monitoring:"
  echo ""
  echo "openclaw gateway config.patch '$PATCH_JSON'"
fi

echo ""
```

## Step 7: Post Restoration Message

```bash
echo "💬 Posting restoration notice..."

# Post a message to the channel about its restoration
RESTORE_MESSAGE="📦 *Channel Restored*\n\nThis channel has been restored from archive.\n\n*Restored:* $RESTORE_DATE\n*By:* Jarvis (WGSD automation)"

POST_RESPONSE=$(curl -s -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL_ID\",\"text\":\"$RESTORE_MESSAGE\"}")

if [ "$(echo "$POST_RESPONSE" | jq -r '.ok')" = "true" ]; then
  echo "✅ Restoration notice posted"
else
  echo "⚠️  Could not post restoration notice (non-fatal)"
fi

echo ""
```

## Step 8: Summary

```bash
echo "═══════════════════════════════════════════════════"
echo "✅ Channel Restored"
echo "═══════════════════════════════════════════════════"
echo ""
echo "   Channel: #$CHANNEL_NAME"
echo "   ID:      $CHANNEL_ID"
echo "   Type:    $CHANNEL_TYPE"
echo "   Date:    $RESTORE_DATE"
echo ""
echo "📋 The channel is now active and monitored by Jarvis"
echo ""

# Type-specific guidance
case "$CHANNEL_TYPE" in
  impl)
    echo "🚀 Implementation channel restored"
    echo "   Continue the implementation work or re-archive when done."
    ;;
  fg)
    echo "🎯 Focus group channel restored"
    echo "   Continue ideation and concept development."
    ;;
  cpt)
    echo "💡 Concept channel restored"
    echo "   Continue refining the concept for implementation."
    ;;
  *)
    echo "   Channel is ready for use."
    ;;
esac

echo ""
echo "═══════════════════════════════════════════════════"

# Export for calling workflows
export RESTORED_CHANNEL_ID="$CHANNEL_ID"
export RESTORED_CHANNEL_NAME="$CHANNEL_NAME"
```

</process>

<success_criteria>
- [ ] Channel unarchived in Slack successfully
- [ ] Registry status updated to "active"
- [ ] Channel re-registered with OpenClaw
- [ ] Restoration notice posted to channel
- [ ] Clear error messages for deleted channels
</success_criteria>

## Usage Examples

```bash
# Restore an implementation that needs more work
/wgsd restore-channel mvn-impl-auth-v2

# Restore a focus group
/wgsd restore-channel mvn-fg-billing

# Restore a concept for further development
/wgsd restore-channel mvn-cpt-legacy-api
```

## Error Handling

| Error | Meaning | Resolution |
|-------|---------|------------|
| `channel_not_found` | Channel was deleted | Create new channel |
| `not_archived` | Channel already active | No action needed |
| `restricted_action` | Workspace restrictions | Contact admin |

## Integration

Called by:
- Manual channel restoration operations
- `workflows/reopen-implementation.md` - If implementation needs continuation

See also:
- `workflows/archive-channel.md` - To archive channels
- `workflows/create-channel.md` - To create new channels
