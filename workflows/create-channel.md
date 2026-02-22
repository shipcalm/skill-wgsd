---
name: wgsd:create-channel
description: Helper workflow to create Slack channels via exec tool and Slack API with naming validation
argument-hint: "[channel-name] [topic]"
allowed-tools:
  - Exec
  - AskUser
---

<objective>
Create a Slack channel using exec tool and Slack API with OpenClaw bot token.
This is a helper workflow for other WGSD operations that need channel creation.

**Naming Validation:** All channel names are validated against WGSD conventions before creation.
</objective>

<process>

## Step 0: Validate Channel Name (WGSD Naming Convention)

Before creating the channel, validate the name against WGSD conventions.

See: `workflows/lib/naming.md` for full naming rules.

```bash
CHANNEL_NAME="{channel-name}"

# Source naming library functions
source_naming_lib() {
  # Auto-correct function
  auto_correct_name() {
    echo "$1" | \
      tr '[:upper:]' '[:lower:]' | \
      tr '_' '-' | \
      tr ' ' '-' | \
      tr -cd 'a-z0-9-' | \
      sed 's/--*/-/g' | \
      sed 's/^-//' | \
      sed 's/-$//' | \
      cut -c1-30
  }
  
  # Parse channel name
  parse_channel_name() {
    local channel=$(echo "$1" | sed 's/^#//')
    
    if echo "$channel" | grep -qE '^[a-z]+-fg-[a-z0-9-]+$'; then
      echo "stub=$(echo "$channel" | sed 's/-fg-.*//')"
      echo "type=fg"
      echo "name=$(echo "$channel" | sed 's/^[a-z]*-fg-//')"
      return 0
    elif echo "$channel" | grep -qE '^[a-z]+-cpt-[a-z0-9-]+$'; then
      echo "stub=$(echo "$channel" | sed 's/-cpt-.*//')"
      echo "type=cpt"
      echo "name=$(echo "$channel" | sed 's/^[a-z]*-cpt-//')"
      return 0
    elif echo "$channel" | grep -qE '^[a-z]+-impl-[a-z0-9-]+$'; then
      echo "stub=$(echo "$channel" | sed 's/-impl-.*//')"
      echo "type=impl"
      echo "name=$(echo "$channel" | sed 's/^[a-z]*-impl-//')"
      return 0
    elif echo "$channel" | grep -qE '^[a-z]+-dev$'; then
      echo "stub=$(echo "$channel" | sed 's/-dev$//')"
      echo "type=dev"
      return 0
    elif echo "$channel" | grep -qE '^[a-z]+-community$'; then
      echo "stub=$(echo "$channel" | sed 's/-community$//')"
      echo "type=community"
      return 0
    else
      return 1
    fi
  }
}

source_naming_lib

# Check total length
if [ ${#CHANNEL_NAME} -gt 80 ]; then
  echo "❌ Channel name too long (${#CHANNEL_NAME} chars, max 80)"
  exit 1
fi

# Try to parse the channel name
PARSED=$(parse_channel_name "$CHANNEL_NAME" 2>/dev/null)
PARSE_RESULT=$?

if [ $PARSE_RESULT -ne 0 ]; then
  # Name doesn't match WGSD pattern - attempt auto-correction
  CORRECTED=$(auto_correct_name "$CHANNEL_NAME")
  
  echo "⚠️  Channel name '$CHANNEL_NAME' doesn't follow WGSD naming convention"
  echo ""
  echo "📋 WGSD Naming Rules:"
  echo "   Pattern: {stub}-{type}[-{name}]"
  echo "   Types: dev, community, fg, cpt, impl"
  echo "   Examples: mvn-dev, mvn-fg-security, mvn-impl-auth-v2"
  echo ""
  
  # If it looks like a partial name, suggest completion
  if echo "$CORRECTED" | grep -qE '^[a-z0-9-]+$'; then
    echo "💡 Suggestions:"
    echo "   • For a focus group: <stub>-fg-$CORRECTED"
    echo "   • For an implementation: <stub>-impl-$CORRECTED"
    echo ""
  fi
  
  echo "Please provide a valid WGSD channel name or bypass validation if intentional."
  exit 1
fi

echo "✅ Channel name validated: $CHANNEL_NAME"
echo "   $PARSED"
```

## Step 1: Get Slack Token

```bash
SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.channels.slack.botToken')

if [ "$SLACK_TOKEN" == "null" ] || [ -z "$SLACK_TOKEN" ]; then
    echo "❌ No Slack bot token found in OpenClaw config"
    exit 1
fi

echo "✅ Slack token retrieved"
```

## Step 2: Create Channel

```bash
CHANNEL_NAME="{channel-name}"
CHANNEL_TOPIC="{topic}"

echo "🚀 Creating channel: #${CHANNEL_NAME}"
echo "📋 Topic: ${CHANNEL_TOPIC}"

# Create channel via Slack API
CHANNEL_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"${CHANNEL_NAME}\",\"is_private\":false}")

# Parse response
CHANNEL_OK=$(echo "$CHANNEL_RESPONSE" | jq -r '.ok')
CHANNEL_ID=$(echo "$CHANNEL_RESPONSE" | jq -r '.channel.id')
CHANNEL_ERROR=$(echo "$CHANNEL_RESPONSE" | jq -r '.error // empty')

if [ "$CHANNEL_OK" == "true" ] && [ "$CHANNEL_ID" != "null" ]; then
    echo "✅ Channel created successfully"
    echo "   Name: #${CHANNEL_NAME}"
    echo "   ID: ${CHANNEL_ID}"
else
    echo "❌ Channel creation failed"
    echo "   Error: ${CHANNEL_ERROR}"
    echo "   Response: ${CHANNEL_RESPONSE}"
    exit 1
fi
```

## Step 3: Set Topic (if provided)

```bash
if [ ! -z "$CHANNEL_TOPIC" ]; then
    echo "📋 Setting channel topic..."
    
    TOPIC_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.setTopic \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"${CHANNEL_ID}\",\"topic\":\"${CHANNEL_TOPIC}\"}")
    
    TOPIC_OK=$(echo "$TOPIC_RESPONSE" | jq -r '.ok')
    
    if [ "$TOPIC_OK" == "true" ]; then
        echo "✅ Channel topic set"
    else
        echo "⚠️  Channel created but topic setting failed"
    fi
fi
```

## Step 4: Return Channel Info

```bash
echo ""
echo "🎉 Channel Ready:"
echo "   Name: #${CHANNEL_NAME}"
echo "   ID: ${CHANNEL_ID}"
echo "   Topic: ${CHANNEL_TOPIC}"
echo ""
echo "✅ Jarvis is automatically a member and can monitor without mentions"
```

## Step 5: Add to OpenClaw Config

```bash
echo "⚙️  Adding channel to OpenClaw config..."

# Use gateway config.patch to add channel to allowed list
PATCH_JSON="{\"channels\":{\"slack\":{\"channels\":{\"${CHANNEL_ID}\":{\"allow\":true,\"requireMention\":false}}}}}"

echo "Patching OpenClaw config with channel ${CHANNEL_ID}..."
```

Use gateway config.patch tool to add the new channel to OpenClaw configuration with:
- `allow: true` - Enable monitoring  
- `requireMention: false` - No @ mentions needed

This ensures Jarvis can monitor the new channel immediately.

## Step 6: Export Variables

```bash
# Export for calling workflow
export CREATED_CHANNEL_ID="$CHANNEL_ID"
export CREATED_CHANNEL_NAME="$CHANNEL_NAME"

echo "✅ Channel fully configured and ready for monitoring"
```

</process>

<success_criteria>
- [ ] Slack bot token retrieved from OpenClaw config
- [ ] Channel created successfully via Slack API  
- [ ] Channel topic set (if provided)
- [ ] Channel ID and name returned to calling workflow
- [ ] Error handling for API failures
</success_criteria>

## Usage Examples

```bash
# Create WGSD methodology channel
/wgsd create-channel wgsd "WGSD methodology and best practices"

# Create focus group channel  
/wgsd create-channel mvn-security "Marvin security focus group"

# Create implementation channel
/wgsd create-channel mvn-impl-auth-v2 "Auth v2 implementation - 1-3 day execution"
```

## Integration with Other Workflows

This workflow is called by:
- `create-focus-group.md` - Focus group channel creation
- `create-implementation.md` - Implementation channel creation  
- Any other WGSD operation requiring new channels

The exec tool + Slack API pattern enables full WGSD automation without requiring additional OpenClaw message tool features.