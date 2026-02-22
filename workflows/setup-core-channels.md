---
name: wgsd:setup-core-channels
description: Create the two core WGSD channels - dev (private) and community (public)
argument-hint: "<stub>"
allowed-tools:
  - Exec
  - AskUser
---

<objective>
Create both core WGSD channels for a project:
1. **{stub}-dev** - Private development channel for the core team
2. **{stub}-community** - Public channel for customer/community feedback

This workflow is called during `wgsd init` to establish the foundational channels.

**Implements:** CHANNEL-03, CHANNEL-04
</objective>

<process>

## Step 1: Validate Stub

```bash
echo "🔧 WGSD Core Channel Setup"
echo "═══════════════════════════════════════════════════"
echo ""

STUB="${1:-}"
WORKSPACE="${WORKSPACE:-.}"

# Get stub from config if not provided
if [ -z "$STUB" ]; then
  STUB=$(grep "^\*\*Stub:\*\*" "$WORKSPACE/.planning/WGSD-CONFIG.md" 2>/dev/null | awk '{print $2}')
fi

# Still no stub? Ask user
if [ -z "$STUB" ]; then
  echo "Enter the channel stub (2-5 lowercase letters):"
  echo "Examples: mvn, oc, wgsd, acme"
  read -r STUB
fi

# Validate stub format
if [ ${#STUB} -lt 2 ] || [ ${#STUB} -gt 5 ]; then
  echo "❌ Stub must be 2-5 characters (got ${#STUB})"
  exit 1
fi

if ! echo "$STUB" | grep -qE '^[a-z]+$'; then
  echo "❌ Stub must be lowercase letters only"
  exit 1
fi

echo "📌 Stub: $STUB"
echo ""
```

## Step 2: Get Slack Token

```bash
echo "🔑 Getting Slack credentials..."

SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.channels.slack.botToken')

if [ "$SLACK_TOKEN" == "null" ] || [ -z "$SLACK_TOKEN" ]; then
  echo "❌ No Slack bot token found in OpenClaw config"
  exit 1
fi

echo "✅ Slack token retrieved"
echo ""
```

## Step 3: Create Dev Channel (Private)

```bash
echo "═══════════════════════════════════════════════════"
echo "📦 Creating Dev Channel (Private)"
echo "═══════════════════════════════════════════════════"

DEV_CHANNEL="${STUB}-dev"
DEV_TOPIC="🛠️ Core development channel for $STUB | WGSD managed"
DEV_PURPOSE="Main development coordination. Architecture decisions, planning discussions, cross-cutting concerns. All core team members should be here."

echo "Creating: #$DEV_CHANNEL"
echo "Privacy: Private (internal team only)"

# Create the channel
DEV_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$DEV_CHANNEL\",\"is_private\":true}")

DEV_OK=$(echo "$DEV_RESPONSE" | jq -r '.ok')
DEV_ERROR=$(echo "$DEV_RESPONSE" | jq -r '.error // empty')

if [ "$DEV_OK" = "true" ]; then
  DEV_ID=$(echo "$DEV_RESPONSE" | jq -r '.channel.id')
  echo "✅ Created #$DEV_CHANNEL ($DEV_ID)"
elif [ "$DEV_ERROR" = "name_taken" ]; then
  echo "⚠️  Channel #$DEV_CHANNEL already exists"
  # Find existing channel
  DEV_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=private_channel&limit=500" \
    -H "Authorization: Bearer $SLACK_TOKEN" | \
    jq -r --arg name "$DEV_CHANNEL" '.channels[] | select(.name == $name) | .id')
  if [ -n "$DEV_ID" ]; then
    echo "   Found existing: $DEV_ID"
  else
    echo "   Could not find existing channel ID"
  fi
else
  echo "❌ Failed to create #$DEV_CHANNEL: $DEV_ERROR"
  DEV_ID=""
fi

# Set topic and purpose
if [ -n "$DEV_ID" ]; then
  curl -s -X POST https://slack.com/api/conversations.setTopic \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"channel\":\"$DEV_ID\",\"topic\":\"$DEV_TOPIC\"}" >/dev/null
    
  curl -s -X POST https://slack.com/api/conversations.setPurpose \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"channel\":\"$DEV_ID\",\"purpose\":\"$DEV_PURPOSE\"}" >/dev/null
    
  echo "✅ Topic and purpose set"
fi

echo ""
```

## Step 4: Create Community Channel (Public)

```bash
echo "═══════════════════════════════════════════════════"
echo "🌍 Creating Community Channel (Public)"
echo "═══════════════════════════════════════════════════"

COMMUNITY_CHANNEL="${STUB}-community"
COMMUNITY_TOPIC="💬 Community feedback and discussion | Public"
COMMUNITY_PURPOSE="Public channel for community feedback, feature requests, and general discussion. Great ideas here may be promoted to focus groups for deeper development."

echo "Creating: #$COMMUNITY_CHANNEL"
echo "Privacy: Public (anyone can join)"

# Create the channel
COMMUNITY_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$COMMUNITY_CHANNEL\",\"is_private\":false}")

COMMUNITY_OK=$(echo "$COMMUNITY_RESPONSE" | jq -r '.ok')
COMMUNITY_ERROR=$(echo "$COMMUNITY_RESPONSE" | jq -r '.error // empty')

if [ "$COMMUNITY_OK" = "true" ]; then
  COMMUNITY_ID=$(echo "$COMMUNITY_RESPONSE" | jq -r '.channel.id')
  echo "✅ Created #$COMMUNITY_CHANNEL ($COMMUNITY_ID)"
elif [ "$COMMUNITY_ERROR" = "name_taken" ]; then
  echo "⚠️  Channel #$COMMUNITY_CHANNEL already exists"
  # Find existing channel
  COMMUNITY_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel&limit=500" \
    -H "Authorization: Bearer $SLACK_TOKEN" | \
    jq -r --arg name "$COMMUNITY_CHANNEL" '.channels[] | select(.name == $name) | .id')
  if [ -n "$COMMUNITY_ID" ]; then
    echo "   Found existing: $COMMUNITY_ID"
  else
    echo "   Could not find existing channel ID"
  fi
else
  echo "❌ Failed to create #$COMMUNITY_CHANNEL: $COMMUNITY_ERROR"
  COMMUNITY_ID=""
fi

# Set topic and purpose
if [ -n "$COMMUNITY_ID" ]; then
  curl -s -X POST https://slack.com/api/conversations.setTopic \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"channel\":\"$COMMUNITY_ID\",\"topic\":\"$COMMUNITY_TOPIC\"}" >/dev/null
    
  curl -s -X POST https://slack.com/api/conversations.setPurpose \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"channel\":\"$COMMUNITY_ID\",\"purpose\":\"$COMMUNITY_PURPOSE\"}" >/dev/null
    
  echo "✅ Topic and purpose set"
fi

echo ""
```

## Step 5: Update Registry

```bash
echo "═══════════════════════════════════════════════════"
echo "📝 Updating Channel Registry"
echo "═══════════════════════════════════════════════════"

CONFIG_PATH="$WORKSPACE/.planning/WGSD-CONFIG.md"
TODAY=$(date +%Y-%m-%d)

if [ -f "$CONFIG_PATH" ]; then
  # Add channels to table
  if [ -n "$DEV_ID" ]; then
    DEV_LINE="| $DEV_CHANNEL | $DEV_ID | dev | active | $TODAY |"
    if ! grep -q "| $DEV_CHANNEL |" "$CONFIG_PATH"; then
      # Find the table and add after header
      if grep -q "^|[-|]*|$" "$CONFIG_PATH"; then
        sed -i "/^|[-|]*|$/a $DEV_LINE" "$CONFIG_PATH"
        echo "✅ Registered: #$DEV_CHANNEL"
      fi
    else
      echo "ℹ️  #$DEV_CHANNEL already in registry"
    fi
  fi
  
  if [ -n "$COMMUNITY_ID" ]; then
    COMMUNITY_LINE="| $COMMUNITY_CHANNEL | $COMMUNITY_ID | community | active | $TODAY |"
    if ! grep -q "| $COMMUNITY_CHANNEL |" "$CONFIG_PATH"; then
      if grep -q "^|[-|]*|$" "$CONFIG_PATH"; then
        sed -i "/^|[-|]*|$/a $COMMUNITY_LINE" "$CONFIG_PATH"
        echo "✅ Registered: #$COMMUNITY_CHANNEL"
      fi
    else
      echo "ℹ️  #$COMMUNITY_CHANNEL already in registry"
    fi
  fi
else
  echo "⚠️  WGSD config not found - channels not registered"
fi

echo ""
```

## Step 6: Register with OpenClaw

```bash
echo "═══════════════════════════════════════════════════"
echo "⚙️  OpenClaw Registration"
echo "═══════════════════════════════════════════════════"

PATCHES=""

if [ -n "$DEV_ID" ]; then
  PATCHES="${PATCHES}\"$DEV_ID\":{\"allow\":true,\"requireMention\":false},"
fi

if [ -n "$COMMUNITY_ID" ]; then
  PATCHES="${PATCHES}\"$COMMUNITY_ID\":{\"allow\":true,\"requireMention\":false},"
fi

if [ -n "$PATCHES" ]; then
  # Remove trailing comma
  PATCHES=$(echo "$PATCHES" | sed 's/,$//')
  PATCH_JSON="{\"channels\":{\"slack\":{\"channels\":{$PATCHES}}}}"
  
  echo "📋 Registering channels with OpenClaw..."
  
  if command -v openclaw &> /dev/null; then
    openclaw gateway config.patch "$PATCH_JSON" 2>/dev/null
    if [ $? -eq 0 ]; then
      echo "✅ Channels registered with OpenClaw"
    else
      echo "⚠️  Auto-registration failed. Run manually:"
      echo "   openclaw gateway config.patch '$PATCH_JSON'"
    fi
  else
    echo "Run this command to enable monitoring:"
    echo ""
    echo "openclaw gateway config.patch '$PATCH_JSON'"
  fi
fi

echo ""
```

## Step 7: Summary

```bash
echo "═══════════════════════════════════════════════════"
echo "🎉 Core Channels Ready"
echo "═══════════════════════════════════════════════════"
echo ""
echo "📦 Dev Channel (Private):"
if [ -n "$DEV_ID" ]; then
  echo "   #$DEV_CHANNEL ($DEV_ID)"
  echo "   Purpose: Internal team development discussions"
else
  echo "   ❌ Not created"
fi
echo ""
echo "🌍 Community Channel (Public):"
if [ -n "$COMMUNITY_ID" ]; then
  echo "   #$COMMUNITY_CHANNEL ($COMMUNITY_ID)"
  echo "   Purpose: External community feedback"
else
  echo "   ❌ Not created"
fi
echo ""
echo "✅ Jarvis will monitor both channels automatically"
echo "═══════════════════════════════════════════════════"

# Export for calling workflows
export DEV_CHANNEL_ID="$DEV_ID"
export DEV_CHANNEL_NAME="$DEV_CHANNEL"
export COMMUNITY_CHANNEL_ID="$COMMUNITY_ID"
export COMMUNITY_CHANNEL_NAME="$COMMUNITY_CHANNEL"
```

</process>

<success_criteria>
- [ ] Dev channel created as private (CHANNEL-03)
- [ ] Community channel created as public (CHANNEL-04)
- [ ] Both channels have appropriate topics and purposes
- [ ] Channels registered in WGSD-CONFIG.md
- [ ] OpenClaw registration instructions provided
- [ ] Channel IDs exported for calling workflow
</success_criteria>

## Usage

```bash
# During init - creates mvn-dev and mvn-community
/wgsd setup-core-channels mvn

# Using workspace stub from config
WORKSPACE=/path/to/marvin /wgsd setup-core-channels
```

## Integration

Called by:
- `workflows/init.md` - During initial WGSD setup

Exports:
- `DEV_CHANNEL_ID` / `DEV_CHANNEL_NAME`
- `COMMUNITY_CHANNEL_ID` / `COMMUNITY_CHANNEL_NAME`
