---
name: wgsd:lib:slack-api
description: Slack API wrapper library for WGSD channel and canvas operations
---

# Slack API Library

Reusable Slack API functions for WGSD channel management.

---

## Configuration

### Token Location
Slack bot token is stored in OpenClaw configuration at:
```
~/.openclaw/openclaw.json → channels.slack.botToken
```

---

## slack_get_token

Get the Slack bot token from OpenClaw configuration.

```bash
# Usage: slack_get_token
# Returns: Token string or exits with error
slack_get_token() {
  local config_file="$HOME/.openclaw/openclaw.json"
  
  if [ ! -f "$config_file" ]; then
    echo "ERROR: OpenClaw config not found at $config_file"
    return 1
  fi
  
  local token=$(jq -r '.channels.slack.botToken // empty' "$config_file")
  
  if [ -z "$token" ]; then
    echo "ERROR: No Slack bot token found in OpenClaw config"
    echo "PATH: channels.slack.botToken"
    return 1
  fi
  
  echo "$token"
  return 0
}
```

---

## slack_api_call

Generic Slack API call wrapper with error handling.

```bash
# Usage: slack_api_call <method> <endpoint> <json_data>
# Returns: API response JSON
slack_api_call() {
  local method="$1"
  local endpoint="$2"
  local data="$3"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    echo "$token"
    return 1
  fi
  
  local url="https://slack.com/api/${endpoint}"
  local response
  
  if [ "$method" = "GET" ]; then
    response=$(curl -s -X GET "$url" \
      -H "Authorization: Bearer $token" \
      -H "Content-Type: application/json")
  else
    response=$(curl -s -X POST "$url" \
      -H "Authorization: Bearer $token" \
      -H "Content-Type: application/json" \
      -d "$data")
  fi
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" != "true" ]; then
    local error=$(echo "$response" | jq -r '.error // "unknown_error"')
    echo "API_ERROR: $error"
    echo "ENDPOINT: $endpoint"
    echo "RESPONSE: $response"
    return 1
  fi
  
  echo "$response"
  return 0
}
```

---

## slack_create_channel

Create a Slack channel (private or public).

```bash
# Usage: slack_create_channel <name> <is_private> [topic] [purpose]
# Arguments:
#   name       - Channel name (validated against WGSD naming convention)
#   is_private - "true" or "false"
#   topic      - Optional channel topic
#   purpose    - Optional channel purpose (description)
# Returns: Channel ID or error
slack_create_channel() {
  local name="$1"
  local is_private="${2:-true}"
  local topic="${3:-}"
  local purpose="${4:-}"
  
  # Validate required args
  if [ -z "$name" ]; then
    echo "ERROR: Channel name is required"
    return 1
  fi
  
  # Normalize is_private to boolean
  case "$is_private" in
    true|yes|1|private) is_private="true" ;;
    false|no|0|public) is_private="false" ;;
    *)
      echo "ERROR: is_private must be 'true' or 'false'"
      return 1
      ;;
  esac
  
  echo "🔄 Creating channel: #$name (private: $is_private)"
  
  # Create the channel
  local create_data=$(jq -n \
    --arg name "$name" \
    --argjson is_private "$is_private" \
    '{name: $name, is_private: $is_private}')
  
  local response=$(slack_api_call POST "conversations.create" "$create_data")
  
  if [ $? -ne 0 ]; then
    # Check for specific errors
    if echo "$response" | grep -q "name_taken"; then
      echo "ERROR: Channel #$name already exists"
      # Try to get the existing channel ID
      local existing=$(slack_find_channel "$name")
      if [ $? -eq 0 ]; then
        echo "EXISTING_ID: $existing"
      fi
    fi
    return 1
  fi
  
  local channel_id=$(echo "$response" | jq -r '.channel.id')
  echo "✅ Channel created: #$name ($channel_id)"
  
  # Set topic if provided
  if [ -n "$topic" ]; then
    slack_set_topic "$channel_id" "$topic" >/dev/null 2>&1 || true
  fi
  
  # Set purpose if provided
  if [ -n "$purpose" ]; then
    slack_set_purpose "$channel_id" "$purpose" >/dev/null 2>&1 || true
  fi
  
  echo "CHANNEL_ID:$channel_id"
  return 0
}
```

---

## slack_archive_channel

Archive a Slack channel.

```bash
# Usage: slack_archive_channel <channel_id>
# Returns: Success message or error
slack_archive_channel() {
  local channel_id="$1"
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel ID is required"
    return 1
  fi
  
  echo "🗃️  Archiving channel: $channel_id"
  
  local data=$(jq -n --arg channel "$channel_id" '{channel: $channel}')
  local response=$(slack_api_call POST "conversations.archive" "$data")
  
  if [ $? -ne 0 ]; then
    if echo "$response" | grep -q "already_archived"; then
      echo "⚠️  Channel already archived"
      return 0
    fi
    return 1
  fi
  
  echo "✅ Channel archived successfully"
  return 0
}
```

---

## slack_unarchive_channel

Restore an archived Slack channel.

```bash
# Usage: slack_unarchive_channel <channel_id>
# Returns: Success message or error
slack_unarchive_channel() {
  local channel_id="$1"
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel ID is required"
    return 1
  fi
  
  echo "📦 Restoring channel: $channel_id"
  
  local data=$(jq -n --arg channel "$channel_id" '{channel: $channel}')
  local response=$(slack_api_call POST "conversations.unarchive" "$data")
  
  if [ $? -ne 0 ]; then
    if echo "$response" | grep -q "not_archived"; then
      echo "⚠️  Channel is not archived"
      return 0
    fi
    return 1
  fi
  
  echo "✅ Channel restored successfully"
  return 0
}
```

---

## slack_invite_users

Invite users to a channel.

```bash
# Usage: slack_invite_users <channel_id> <user_ids>
# Arguments:
#   channel_id - Channel to invite to
#   user_ids   - Comma-separated user IDs
# Returns: Success message or error
slack_invite_users() {
  local channel_id="$1"
  local user_ids="$2"
  
  if [ -z "$channel_id" ] || [ -z "$user_ids" ]; then
    echo "ERROR: Channel ID and user IDs are required"
    return 1
  fi
  
  echo "👥 Inviting users to channel..."
  
  local data=$(jq -n \
    --arg channel "$channel_id" \
    --arg users "$user_ids" \
    '{channel: $channel, users: $users}')
  
  local response=$(slack_api_call POST "conversations.invite" "$data")
  
  if [ $? -ne 0 ]; then
    if echo "$response" | grep -q "already_in_channel"; then
      echo "⚠️  Users already in channel"
      return 0
    fi
    return 1
  fi
  
  echo "✅ Users invited successfully"
  return 0
}
```

---

## slack_join_channel

Join a public channel (for bot to auto-join).

```bash
# Usage: slack_join_channel <channel_id>
# Returns: Success message or error
slack_join_channel() {
  local channel_id="$1"
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel ID is required"
    return 1
  fi
  
  echo "🤖 Bot joining channel..."
  
  local data=$(jq -n --arg channel "$channel_id" '{channel: $channel}')
  local response=$(slack_api_call POST "conversations.join" "$data")
  
  if [ $? -ne 0 ]; then
    # Private channels require invite, not join
    if echo "$response" | grep -q "method_not_supported_for_channel_type"; then
      echo "ℹ️  Private channel - bot should already be member as creator"
      return 0
    fi
    return 1
  fi
  
  echo "✅ Bot joined channel"
  return 0
}
```

---

## slack_get_channel

Get channel information.

```bash
# Usage: slack_get_channel <channel_id>
# Returns: Channel JSON or error
slack_get_channel() {
  local channel_id="$1"
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel ID is required"
    return 1
  fi
  
  local response=$(slack_api_call GET "conversations.info?channel=$channel_id")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "$response" | jq '.channel'
  return 0
}
```

---

## slack_find_channel

Find a channel by name.

```bash
# Usage: slack_find_channel <channel_name>
# Returns: Channel ID or error
slack_find_channel() {
  local name="$1"
  
  # Remove leading # if present
  name=$(echo "$name" | sed 's/^#//')
  
  if [ -z "$name" ]; then
    echo "ERROR: Channel name is required"
    return 1
  fi
  
  # Search public channels
  local response=$(slack_api_call GET "conversations.list?types=public_channel,private_channel&limit=1000")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  local channel_id=$(echo "$response" | jq -r --arg name "$name" \
    '.channels[] | select(.name == $name) | .id')
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel #$name not found"
    return 1
  fi
  
  echo "$channel_id"
  return 0
}
```

---

## slack_list_channels

List channels matching criteria.

```bash
# Usage: slack_list_channels [types] [prefix]
# Arguments:
#   types  - Channel types: public_channel,private_channel (default: both)
#   prefix - Optional name prefix filter
# Returns: JSON array of channels
slack_list_channels() {
  local types="${1:-public_channel,private_channel}"
  local prefix="$2"
  
  local response=$(slack_api_call GET "conversations.list?types=$types&limit=1000")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  if [ -n "$prefix" ]; then
    echo "$response" | jq --arg prefix "$prefix" \
      '[.channels[] | select(.name | startswith($prefix))]'
  else
    echo "$response" | jq '.channels'
  fi
  
  return 0
}
```

---

## slack_set_topic

Set channel topic.

```bash
# Usage: slack_set_topic <channel_id> <topic>
# Returns: Success message or error
slack_set_topic() {
  local channel_id="$1"
  local topic="$2"
  
  if [ -z "$channel_id" ] || [ -z "$topic" ]; then
    echo "ERROR: Channel ID and topic are required"
    return 1
  fi
  
  local data=$(jq -n \
    --arg channel "$channel_id" \
    --arg topic "$topic" \
    '{channel: $channel, topic: $topic}')
  
  local response=$(slack_api_call POST "conversations.setTopic" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "✅ Topic set"
  return 0
}
```

---

## slack_set_purpose

Set channel purpose/description.

```bash
# Usage: slack_set_purpose <channel_id> <purpose>
# Returns: Success message or error
slack_set_purpose() {
  local channel_id="$1"
  local purpose="$2"
  
  if [ -z "$channel_id" ] || [ -z "$purpose" ]; then
    echo "ERROR: Channel ID and purpose are required"
    return 1
  fi
  
  local data=$(jq -n \
    --arg channel "$channel_id" \
    --arg purpose "$purpose" \
    '{channel: $channel, purpose: $purpose}')
  
  local response=$(slack_api_call POST "conversations.setPurpose" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "✅ Purpose set"
  return 0
}
```

---

## slack_create_canvas

Create a canvas attached to a channel.

```bash
# Usage: slack_create_canvas <channel_id> <title> <markdown_content>
# Returns: Canvas ID or error
slack_create_canvas() {
  local channel_id="$1"
  local title="$2"
  local content="$3"
  
  if [ -z "$channel_id" ] || [ -z "$title" ]; then
    echo "ERROR: Channel ID and title are required"
    return 1
  fi
  
  # Default content if not provided
  content="${content:-# $title\n\n*Canvas content will be populated by WGSD*}"
  
  echo "📋 Creating canvas: $title"
  
  local data=$(jq -n \
    --arg channel "$channel_id" \
    --arg title "$title" \
    --arg content "$content" \
    '{
      channel_id: $channel,
      title: $title,
      document_content: {
        type: "markdown",
        markdown: $content
      }
    }')
  
  local response=$(slack_api_call POST "conversations.canvases.create" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  local canvas_id=$(echo "$response" | jq -r '.canvas_id')
  echo "✅ Canvas created: $canvas_id"
  echo "CANVAS_ID:$canvas_id"
  return 0
}
```

---

## openclaw_register_channel

Register a channel in OpenClaw configuration for monitoring.

```bash
# Usage: openclaw_register_channel <channel_id> <channel_name>
# Returns: Success message or error
openclaw_register_channel() {
  local channel_id="$1"
  local channel_name="$2"
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Channel ID is required"
    return 1
  fi
  
  echo "⚙️  Registering channel with OpenClaw..."
  
  # Note: This would ideally use openclaw gateway config.patch
  # For now, we document the required patch
  
  echo "PATCH_REQUIRED:"
  echo "{\"channels\":{\"slack\":{\"channels\":{\"$channel_id\":{\"allow\":true,\"requireMention\":false}}}}}"
  
  echo "✅ Channel registered (apply patch via openclaw gateway config.patch)"
  return 0
}
```

---

## Channel Type Helpers

### get_channel_privacy

Determine if a channel type should be private.

```bash
# Usage: get_channel_privacy <type>
# Returns: "true" or "false"
get_channel_privacy() {
  local type="$1"
  
  case "$type" in
    community)
      echo "false"  # Public channel
      ;;
    dev|fg|cpt|impl|focus-group|concept|implementation)
      echo "true"   # Private channel
      ;;
    *)
      echo "true"   # Default to private
      ;;
  esac
}
```

### get_channel_topic

Generate standard topic for channel type.

```bash
# Usage: get_channel_topic <type> <name> <stub>
# Returns: Topic string
get_channel_topic() {
  local type="$1"
  local name="$2"
  local stub="$3"
  
  case "$type" in
    dev)
      echo "🛠️ Core development channel for $stub | WGSD managed"
      ;;
    community)
      echo "💬 Community feedback and discussion | Public"
      ;;
    fg|focus-group)
      echo "🎯 Focus group: $name | Planning & ideation"
      ;;
    cpt|concept)
      echo "💡 Concept: $name | Feature development"
      ;;
    impl|implementation)
      echo "🚀 Implementation: $name | Active execution (1-3 days)"
      ;;
    *)
      echo "WGSD managed channel"
      ;;
  esac
}
```

### get_channel_purpose

Generate standard purpose/description for channel type.

```bash
# Usage: get_channel_purpose <type> <name>
# Returns: Purpose string
get_channel_purpose() {
  local type="$1"
  local name="$2"
  
  case "$type" in
    dev)
      echo "Main development coordination channel. All team members discuss architecture, planning, and cross-cutting concerns here."
      ;;
    community)
      echo "Public channel for community feedback, feature requests, and general discussion. Ideas here may be promoted to focus groups."
      ;;
    fg|focus-group)
      echo "Long-lived focus group for ${name} development. Concepts are developed here before promotion to implementation."
      ;;
    cpt|concept)
      echo "Concept discussion for ${name}. Once mature, this concept can be promoted to an active implementation."
      ;;
    impl|implementation)
      echo "Active implementation of ${name}. Target: 1-3 days. Direct code changes on develop branch."
      ;;
    *)
      echo "WGSD managed channel for $name"
      ;;
  esac
}
```

---

## Usage Examples

```bash
# Create a private dev channel
slack_create_channel "mvn-dev" "true" "Core development" "Main dev channel"

# Create a public community channel  
slack_create_channel "mvn-community" "false" "Community feedback"

# Archive an implementation channel
slack_archive_channel "C0123456789"

# Find and restore a channel
channel_id=$(slack_find_channel "mvn-impl-auth-v2")
slack_unarchive_channel "$channel_id"

# List all project channels
slack_list_channels "public_channel,private_channel" "mvn-"

# Create a canvas for a channel
slack_create_canvas "C0123456789" "Security Roadmap" "# Security Focus Group\n\n## Concepts\n- [ ] SSO Integration"
```

---

*Library created for WGSD Phase 3*
