---
name: wgsd:archive-channel
description: Archive a WGSD channel with state preservation and registry update
argument-hint: "<channel-name> [reason]"
allowed-tools:
  - Exec
  - AskUser
---

<objective>
Archive a WGSD Slack channel with proper cleanup:
1. Archive the channel in Slack
2. Update registry status to "archived"
3. Optionally preserve final state
4. Record archive reason

**Implements:** CHANNEL-08 (lifecycle management)

**When to archive:**
- Implementation completed and merged
- Focus group no longer active
- Concept abandoned or merged into another
</objective>

<process>

## Step 1: Parse Arguments

```bash
echo "🗃️  WGSD Channel Archive"
echo "═══════════════════════════════════════════════════"
echo ""

CHANNEL_NAME="${1:-}"
REASON="${2:-Archive requested}"
WORKSPACE="${WORKSPACE:-.}"
FORCE="${FORCE:-false}"

if [ -z "$CHANNEL_NAME" ]; then
  echo "❌ Channel name is required"
  echo ""
  echo "Usage: archive-channel <channel-name> [reason]"
  echo ""
  echo "Examples:"
  echo "  archive-channel mvn-impl-auth-v2 \"Implementation complete\""
  echo "  archive-channel mvn-fg-billing \"Focus group merged into core\""
  exit 1
fi

# Remove leading # if present
CHANNEL_NAME=$(echo "$CHANNEL_NAME" | sed 's/^#//')

echo "📌 Channel: #$CHANNEL_NAME"
echo "📋 Reason: $REASON"
echo ""
```

## Step 2: Validate Channel Type

```bash
echo "🔍 Validating channel..."

# Parse channel name to get type
parse_channel() {
  local channel="$1"
  if echo "$channel" | grep -qE '^[a-z]+-fg-'; then
    echo "type=fg"
  elif echo "$channel" | grep -qE '^[a-z]+-cpt-'; then
    echo "type=cpt"
  elif echo "$channel" | grep -qE '^[a-z]+-impl-'; then
    echo "type=impl"
  elif echo "$channel" | grep -qE '^[a-z]+-dev$'; then
    echo "type=dev"
  elif echo "$channel" | grep -qE '^[a-z]+-community$'; then
    echo "type=community"
  else
    echo "type=unknown"
  fi
}

CHANNEL_INFO=$(parse_channel "$CHANNEL_NAME")
eval "$CHANNEL_INFO"

echo "   Type: $type"

# Protect core channels from accidental archive
if [ "$type" = "dev" ] || [ "$type" = "community" ]; then
  if [ "$FORCE" != "true" ]; then
    echo ""
    echo "⚠️  WARNING: You're trying to archive a core channel!"
    echo "   #$CHANNEL_NAME is a $type channel."
    echo ""
    echo "   Archiving core channels is usually not recommended."
    echo "   Are you sure? (yes/no)"
    read -r CONFIRM
    
    if [ "$CONFIRM" != "yes" ]; then
      echo "❌ Archive cancelled"
      exit 1
    fi
  fi
fi

echo "✅ Channel validated"
echo ""
```

## Step 3: Get Channel ID

```bash
echo "🔍 Looking up channel ID..."

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

# If not in registry, search Slack
if [ -z "$CHANNEL_ID" ]; then
  echo "   Not in registry, searching Slack..."
  
  CHANNEL_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel,private_channel&limit=500" \
    -H "Authorization: Bearer $SLACK_TOKEN" | \
    jq -r --arg name "$CHANNEL_NAME" '.channels[] | select(.name == $name) | .id')
fi

if [ -z "$CHANNEL_ID" ]; then
  echo "❌ Channel #$CHANNEL_NAME not found"
  exit 1
fi

echo "✅ Found channel: $CHANNEL_ID"
echo ""
```

## Step 4: Check Current State

```bash
echo "🔍 Checking channel state..."

CHANNEL_INFO=$(curl -s -X GET "https://slack.com/api/conversations.info?channel=$CHANNEL_ID" \
  -H "Authorization: Bearer $SLACK_TOKEN")

IS_ARCHIVED=$(echo "$CHANNEL_INFO" | jq -r '.channel.is_archived')

if [ "$IS_ARCHIVED" = "true" ]; then
  echo "⚠️  Channel is already archived"
  
  # Still update registry if needed
  if [ -f "$CONFIG_PATH" ] && grep -q "| $CHANNEL_NAME |.*| active |" "$CONFIG_PATH"; then
    echo "📝 Updating registry to match Slack state..."
    sed -i "s/| $CHANNEL_NAME |\\(.*\\)| active |/| $CHANNEL_NAME |\\1| archived |/g" "$CONFIG_PATH"
    echo "✅ Registry updated"
  fi
  
  exit 0
fi

echo "✅ Channel is active and can be archived"
echo ""
```

## Step 5: Preserve Final State (Optional)

```bash
echo "💾 Preserving channel state..."

# Get channel message count and last activity
CHANNEL_DATA=$(echo "$CHANNEL_INFO" | jq '{
  name: .channel.name,
  id: .channel.id,
  created: .channel.created,
  creator: .channel.creator,
  topic: .channel.topic.value,
  purpose: .channel.purpose.value,
  num_members: .channel.num_members
}')

echo "   Messages: $(echo "$CHANNEL_INFO" | jq -r '.channel.num_members // "unknown"') members"
echo ""

# Save archive metadata
ARCHIVE_DATE=$(date +%Y-%m-%d)
ARCHIVE_TIME=$(date +%H:%M:%S)

ARCHIVE_RECORD=$(cat <<EOF
### Archived: $CHANNEL_NAME

- **Channel ID:** $CHANNEL_ID
- **Archive Date:** $ARCHIVE_DATE $ARCHIVE_TIME
- **Reason:** $REASON
- **Type:** $type
- **Topic:** $(echo "$CHANNEL_INFO" | jq -r '.channel.topic.value // "none"')

EOF
)

echo "$ARCHIVE_RECORD"
echo ""
```

## Step 6: Archive the Channel

```bash
echo "🗃️  Archiving channel in Slack..."

ARCHIVE_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.archive \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL_ID\"}")

ARCHIVE_OK=$(echo "$ARCHIVE_RESPONSE" | jq -r '.ok')
ARCHIVE_ERROR=$(echo "$ARCHIVE_RESPONSE" | jq -r '.error // empty')

if [ "$ARCHIVE_OK" = "true" ]; then
  echo "✅ Channel archived in Slack"
elif [ "$ARCHIVE_ERROR" = "already_archived" ]; then
  echo "⚠️  Channel was already archived"
else
  echo "❌ Failed to archive channel"
  echo "   Error: $ARCHIVE_ERROR"
  exit 1
fi

echo ""
```

## Step 7: Update Registry

```bash
echo "📝 Updating channel registry..."

if [ -f "$CONFIG_PATH" ]; then
  # Update status from active to archived
  if grep -q "| $CHANNEL_NAME |" "$CONFIG_PATH"; then
    sed -i "s/| $CHANNEL_NAME |\\(.*\\)| active |/| $CHANNEL_NAME |\\1| archived |/g" "$CONFIG_PATH"
    echo "✅ Registry updated: status → archived"
  else
    echo "⚠️  Channel not in registry (adding as archived)"
    ARCHIVE_LINE="| $CHANNEL_NAME | $CHANNEL_ID | $type | archived | $ARCHIVE_DATE |"
    
    if grep -q "^|[-|]*|$" "$CONFIG_PATH"; then
      sed -i "/^|[-|]*|$/a $ARCHIVE_LINE" "$CONFIG_PATH"
      echo "✅ Added to registry as archived"
    fi
  fi
else
  echo "⚠️  WGSD config not found - registry not updated"
fi

echo ""
```

## Step 8: Deregister from OpenClaw (Optional)

```bash
echo "⚙️  OpenClaw cleanup..."

# For archived channels, we can optionally disable monitoring
# to reduce noise. The channel can be re-enabled if restored.

echo "ℹ️  Channel remains in OpenClaw config but is inactive"
echo "   To remove monitoring, use:"
echo "   openclaw gateway config.patch '{\"channels\":{\"slack\":{\"channels\":{\"$CHANNEL_ID\":null}}}}'"

echo ""
```

## Step 9: Summary

```bash
echo "═══════════════════════════════════════════════════"
echo "✅ Channel Archived"
echo "═══════════════════════════════════════════════════"
echo ""
echo "   Channel: #$CHANNEL_NAME"
echo "   ID:      $CHANNEL_ID"
echo "   Type:    $type"
echo "   Reason:  $REASON"
echo "   Date:    $ARCHIVE_DATE"
echo ""
echo "📋 To restore this channel:"
echo "   /wgsd restore-channel $CHANNEL_NAME"
echo ""
echo "⚠️  Archived channels can be restored but not undeleted"
echo "═══════════════════════════════════════════════════"

# Export for calling workflows
export ARCHIVED_CHANNEL_ID="$CHANNEL_ID"
export ARCHIVED_CHANNEL_NAME="$CHANNEL_NAME"
```

</process>

<success_criteria>
- [ ] Channel archived in Slack successfully
- [ ] Registry status updated to "archived"
- [ ] Archive reason recorded
- [ ] Final state preserved (metadata)
- [ ] Core channels protected with confirmation
- [ ] OpenClaw cleanup instructions provided
</success_criteria>

## Usage Examples

```bash
# Archive completed implementation
/wgsd archive-channel mvn-impl-auth-v2 "Implementation complete and merged"

# Archive abandoned concept
/wgsd archive-channel mvn-cpt-legacy-api "Concept abandoned - superseded by v2"

# Archive focus group
/wgsd archive-channel mvn-fg-billing "Merged into core development"

# Force archive core channel (rare)
FORCE=true /wgsd archive-channel mvn-dev "Project sunset"
```

## Integration

Called by:
- `workflows/complete-implementation.md` - When implementation merges
- Manual channel cleanup operations

See also:
- `workflows/restore-channel.md` - To unarchive channels
