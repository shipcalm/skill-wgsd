---
name: wgsd:create-channel
description: Create WGSD Slack channels with automatic privacy, topic, and registry management
argument-hint: "<type> <name> [topic]"
allowed-tools:
  - Exec
  - AskUser
---

<objective>
Create Slack channels with WGSD naming conventions, appropriate privacy settings, 
and automatic registration. Supports all channel types: dev, community, fg, cpt, impl.

**Dependencies:**
- `workflows/lib/slack-api.md` - Slack API operations
- `workflows/lib/channel-registry.md` - Channel tracking
- `workflows/lib/naming.md` - Naming conventions
</objective>

<process>

## Step 0: Parse Arguments

```bash
# Arguments can be:
#   1. Full channel name: mvn-fg-security
#   2. Type and name: fg security (with stub from registry)
#   3. Just type for dev/community: dev

CHANNEL_TYPE=""
CHANNEL_NAME=""
CHANNEL_FULL=""
TOPIC=""
WORKSPACE="${WORKSPACE:-.}"

# Detect argument pattern
if [ "$#" -eq 1 ]; then
  # Could be full channel name or type
  arg="$1"
  if echo "$arg" | grep -qE '^[a-z]+-'; then
    # Full channel name
    CHANNEL_FULL="$arg"
  else
    # Just type (dev or community)
    CHANNEL_TYPE="$arg"
  fi
elif [ "$#" -eq 2 ]; then
  # Type and name
  CHANNEL_TYPE="$1"
  CHANNEL_NAME="$2"
elif [ "$#" -ge 3 ]; then
  # Type, name, and topic
  CHANNEL_TYPE="$1"
  CHANNEL_NAME="$2"
  shift 2
  TOPIC="$*"
fi
```

## Step 1: Load Libraries

```bash
echo "📚 Loading WGSD libraries..."

# Source library functions (conceptually - these are inline for workflow execution)

# From naming.md
source_naming() {
  validate_stub() {
    local stub="$1"
    if [ -z "$stub" ]; then return 1; fi
    if [ ${#stub} -lt 2 ] || [ ${#stub} -gt 5 ]; then return 1; fi
    if ! echo "$stub" | grep -qE '^[a-z]+$'; then return 1; fi
    return 0
  }
  
  generate_channel_name() {
    local stub="$1" type="$2" name="$3"
    case "$type" in
      dev) echo "${stub}-dev" ;;
      community) echo "${stub}-community" ;;
      fg|focus-group) echo "${stub}-fg-${name}" ;;
      cpt|concept) echo "${stub}-cpt-${name}" ;;
      impl|implementation) echo "${stub}-impl-${name}" ;;
      *) return 1 ;;
    esac
  }
  
  parse_channel_name() {
    local channel=$(echo "$1" | sed 's/^#//')
    if echo "$channel" | grep -qE '^[a-z]+-fg-'; then
      echo "stub=$(echo "$channel" | sed 's/-fg-.*//') type=fg name=$(echo "$channel" | sed 's/.*-fg-//')"
    elif echo "$channel" | grep -qE '^[a-z]+-cpt-'; then
      echo "stub=$(echo "$channel" | sed 's/-cpt-.*//') type=cpt name=$(echo "$channel" | sed 's/.*-cpt-//')"
    elif echo "$channel" | grep -qE '^[a-z]+-impl-'; then
      echo "stub=$(echo "$channel" | sed 's/-impl-.*//') type=impl name=$(echo "$channel" | sed 's/.*-impl-//')"
    elif echo "$channel" | grep -qE '^[a-z]+-dev$'; then
      echo "stub=$(echo "$channel" | sed 's/-dev$//') type=dev name="
    elif echo "$channel" | grep -qE '^[a-z]+-community$'; then
      echo "stub=$(echo "$channel" | sed 's/-community$//') type=community name="
    else
      return 1
    fi
  }
}
source_naming

# From slack-api.md
source_slack_api() {
  get_channel_privacy() {
    case "$1" in
      community) echo "false" ;;
      *) echo "true" ;;
    esac
  }
  
  get_channel_topic() {
    local type="$1" name="$2" stub="$3"
    case "$type" in
      dev) echo "🛠️ Core development channel for $stub | WGSD managed" ;;
      community) echo "💬 Community feedback and discussion | Public" ;;
      fg|focus-group) echo "🎯 Focus group: $name | Planning & ideation" ;;
      cpt|concept) echo "💡 Concept: $name | Feature development" ;;
      impl|implementation) echo "🚀 Implementation: $name | Active execution (1-3 days)" ;;
      *) echo "WGSD managed channel" ;;
    esac
  }
  
  get_channel_purpose() {
    local type="$1" name="$2"
    case "$type" in
      dev) echo "Main development coordination. Architecture, planning, cross-cutting concerns." ;;
      community) echo "Public feedback channel. Ideas here may be promoted to focus groups." ;;
      fg) echo "Focus group for ${name}. Concepts developed here before implementation." ;;
      cpt) echo "Concept discussion for ${name}. Promote to implementation when mature." ;;
      impl) echo "Active implementation of ${name}. Target: 1-3 days execution." ;;
      *) echo "WGSD managed channel" ;;
    esac
  }
}
source_slack_api

echo "✅ Libraries loaded"
```

## Step 2: Resolve Channel Name

```bash
echo ""
echo "🔍 Resolving channel name..."

# If we have a full channel name, parse it
if [ -n "$CHANNEL_FULL" ]; then
  PARSED=$(parse_channel_name "$CHANNEL_FULL")
  if [ $? -ne 0 ]; then
    echo "❌ Invalid channel name format: $CHANNEL_FULL"
    echo ""
    echo "📋 Expected format: {stub}-{type}[-{name}]"
    echo "   Examples: mvn-dev, mvn-fg-security, mvn-impl-auth-v2"
    exit 1
  fi
  
  eval "$PARSED"
  CHANNEL_TYPE="$type"
  CHANNEL_NAME="$name"
  STUB="$stub"
else
  # Get stub from registry or ask user
  STUB=$(grep "^\*\*Stub:\*\*" "$WORKSPACE/.planning/WGSD-CONFIG.md" 2>/dev/null | awk '{print $2}')
  
  if [ -z "$STUB" ]; then
    echo "⚠️  No stub found in WGSD config"
    echo "Enter the channel stub (2-5 lowercase letters):"
    read -r STUB
    
    if ! validate_stub "$STUB"; then
      echo "❌ Invalid stub: $STUB"
      exit 1
    fi
  fi
  
  # Generate full channel name
  CHANNEL_FULL=$(generate_channel_name "$STUB" "$CHANNEL_TYPE" "$CHANNEL_NAME")
  if [ $? -ne 0 ]; then
    echo "❌ Invalid channel type: $CHANNEL_TYPE"
    echo "   Valid types: dev, community, fg, cpt, impl"
    exit 1
  fi
fi

echo "✅ Channel name resolved:"
echo "   Full name: #$CHANNEL_FULL"
echo "   Stub: $STUB"
echo "   Type: $CHANNEL_TYPE"
[ -n "$CHANNEL_NAME" ] && echo "   Name: $CHANNEL_NAME"
```

## Step 3: Determine Privacy and Topic

```bash
echo ""
echo "⚙️  Determining channel settings..."

IS_PRIVATE=$(get_channel_privacy "$CHANNEL_TYPE")

# Generate topic if not provided
if [ -z "$TOPIC" ]; then
  TOPIC=$(get_channel_topic "$CHANNEL_TYPE" "$CHANNEL_NAME" "$STUB")
fi

PURPOSE=$(get_channel_purpose "$CHANNEL_TYPE" "$CHANNEL_NAME")

echo "   Private: $IS_PRIVATE"
echo "   Topic: $TOPIC"
```

## Step 4: Get Slack Token

```bash
echo ""
echo "🔑 Getting Slack credentials..."

SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.channels.slack.botToken')

if [ "$SLACK_TOKEN" == "null" ] || [ -z "$SLACK_TOKEN" ]; then
  echo "❌ No Slack bot token found in OpenClaw config"
  echo "   Path: channels.slack.botToken"
  exit 1
fi

echo "✅ Slack token retrieved"
```

## Step 5: Create the Channel

```bash
echo ""
echo "🚀 Creating Slack channel..."
echo "   Name: #$CHANNEL_FULL"
echo "   Private: $IS_PRIVATE"

# Build request JSON
CREATE_JSON=$(jq -n \
  --arg name "$CHANNEL_FULL" \
  --argjson is_private "$IS_PRIVATE" \
  '{name: $name, is_private: $is_private}')

# Create channel
RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$CREATE_JSON")

OK=$(echo "$RESPONSE" | jq -r '.ok')
ERROR=$(echo "$RESPONSE" | jq -r '.error // empty')

if [ "$OK" != "true" ]; then
  if [ "$ERROR" = "name_taken" ]; then
    echo "⚠️  Channel #$CHANNEL_FULL already exists"
    
    # Try to find existing channel ID
    EXISTING=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel,private_channel&limit=500" \
      -H "Authorization: Bearer $SLACK_TOKEN" | \
      jq -r --arg name "$CHANNEL_FULL" '.channels[] | select(.name == $name) | .id')
    
    if [ -n "$EXISTING" ]; then
      CHANNEL_ID="$EXISTING"
      echo "   Existing channel ID: $CHANNEL_ID"
    else
      echo "❌ Could not find existing channel. May need manual lookup."
      exit 1
    fi
  else
    echo "❌ Failed to create channel"
    echo "   Error: $ERROR"
    echo "   Response: $RESPONSE"
    exit 1
  fi
else
  CHANNEL_ID=$(echo "$RESPONSE" | jq -r '.channel.id')
  echo "✅ Channel created successfully"
  echo "   ID: $CHANNEL_ID"
fi
```

## Step 6: Set Topic and Purpose

```bash
echo ""
echo "📋 Setting channel topic and purpose..."

# Set topic
TOPIC_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.setTopic \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL_ID\",\"topic\":\"$TOPIC\"}")

if [ "$(echo "$TOPIC_RESPONSE" | jq -r '.ok')" = "true" ]; then
  echo "✅ Topic set"
else
  echo "⚠️  Could not set topic (non-fatal)"
fi

# Set purpose
PURPOSE_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.setPurpose \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL_ID\",\"purpose\":\"$PURPOSE\"}")

if [ "$(echo "$PURPOSE_RESPONSE" | jq -r '.ok')" = "true" ]; then
  echo "✅ Purpose set"
else
  echo "⚠️  Could not set purpose (non-fatal)"
fi
```

## Step 7: Register Channel

```bash
echo ""
echo "📝 Registering channel..."

# Update WGSD-CONFIG.md
CONFIG_PATH="$WORKSPACE/.planning/WGSD-CONFIG.md"
TODAY=$(date +%Y-%m-%d)

if [ -f "$CONFIG_PATH" ]; then
  # Add to channel table
  TABLE_LINE="| $CHANNEL_FULL | $CHANNEL_ID | $CHANNEL_TYPE | active | $TODAY |"
  
  # Check if table exists
  if grep -q "^| Channel |" "$CONFIG_PATH"; then
    # Insert after header row (after the |---|---|...| line)
    sed -i "/^|[-|]*$/a $TABLE_LINE" "$CONFIG_PATH"
    echo "✅ Added to registry: $CHANNEL_FULL"
  else
    echo "⚠️  No channel table found in config (manual update needed)"
  fi
else
  echo "⚠️  No WGSD config found (channel not registered)"
fi
```

## Step 8: Register with OpenClaw

```bash
echo ""
echo "⚙️  Registering with OpenClaw..."

# Generate the patch command
PATCH_JSON="{\"channels\":{\"slack\":{\"channels\":{\"$CHANNEL_ID\":{\"allow\":true,\"requireMention\":false}}}}}"

echo "📋 Run this command to enable monitoring:"
echo ""
echo "openclaw gateway config.patch '$PATCH_JSON'"
echo ""

# Try to apply automatically
if command -v openclaw &> /dev/null; then
  echo "🔄 Attempting automatic registration..."
  openclaw gateway config.patch "$PATCH_JSON" 2>/dev/null
  if [ $? -eq 0 ]; then
    echo "✅ Channel registered with OpenClaw"
  else
    echo "⚠️  Automatic registration failed - apply patch manually"
  fi
else
  echo "ℹ️  openclaw command not available - apply patch manually"
fi
```

## Step 9: Summary

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "🎉 Channel Ready"
echo "═══════════════════════════════════════════════════"
echo ""
echo "   Channel: #$CHANNEL_FULL"
echo "   ID:      $CHANNEL_ID"
echo "   Type:    $CHANNEL_TYPE"
echo "   Private: $IS_PRIVATE"
echo "   Topic:   $TOPIC"
echo ""

if [ "$IS_PRIVATE" = "true" ]; then
  echo "🔒 This is a private channel. Invite team members with:"
  echo "   /invite @username"
else
  echo "🌍 This is a public channel. Anyone can join."
fi

echo ""
echo "✅ Jarvis will monitor this channel automatically"
echo "═══════════════════════════════════════════════════"

# Export for calling workflows
export CREATED_CHANNEL_ID="$CHANNEL_ID"
export CREATED_CHANNEL_NAME="$CHANNEL_FULL"
export CREATED_CHANNEL_TYPE="$CHANNEL_TYPE"
```

</process>

<success_criteria>
- [ ] Channel name validated against WGSD conventions
- [ ] Privacy correctly set (dev/fg/cpt/impl=private, community=public)
- [ ] Channel created via Slack API
- [ ] Topic and purpose set appropriately
- [ ] Channel registered in WGSD-CONFIG.md
- [ ] OpenClaw registration instructions provided
- [ ] Channel ID exported for calling workflows
</success_criteria>

## Usage Examples

```bash
# Create core dev channel (private)
/wgsd create-channel dev

# Create community channel (public)
/wgsd create-channel community

# Create focus group channel (private)
/wgsd create-channel fg security

# Create concept channel (private)
/wgsd create-channel cpt byof-filesystem

# Create implementation channel (private)
/wgsd create-channel impl auth-v2

# Create with full name
/wgsd create-channel mvn-fg-security

# Create with custom topic
/wgsd create-channel fg security "Security and auth features"
```

## Integration

This workflow is called by:
- `workflows/init.md` - Creates dev + community channels
- `workflows/create-focus-group.md` - Creates focus group channel
- `workflows/create-concept.md` - Creates concept channel
- `workflows/create-implementation.md` - Creates implementation channel

The workflow exports `CREATED_CHANNEL_ID` and `CREATED_CHANNEL_NAME` for use by calling workflows.
