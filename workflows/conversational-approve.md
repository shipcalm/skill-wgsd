---
name: wgsd:workflow:conversational-approve
description: Slack-native conversational approval workflow with channel context inference
triggers:
  - command: "/wgsd approve {concept}"
  - command: "/wgsd approve {concept} --comment {comment}"
  - command: "/wgsd approve {concept} --fg {focus_group}"
  - slack_reaction: "white_check_mark" on approval prompt
  - slack_reaction: "thumbsup" on approval prompt
  - slack_thread_reply: "approved|lgtm|+1|ship it" in discussion thread
---

# Conversational Approve Workflow

Enable frictionless approvals through natural Slack interactions.

---

## Overview

Conversational approval transforms approvals from formal commands to natural conversation:

1. **Slash command** - `/wgsd approve oauth-integration`
2. **Emoji reaction** - ✅ or 👍 on approval prompt
3. **Thread reply** - "LGTM", "approved", "+1" in discussion
4. **Quick reply** - Just say "approve" in the FG channel

All methods infer context from where the action happens.

---

## Entry Points

### Slash Command Approval

```bash
# Simplest: Approve from focus group channel (FG inferred)
/wgsd approve oauth-integration

# With comment
/wgsd approve oauth-integration --comment "Token flow looks good"

# Explicit focus group (when not in FG channel)
/wgsd approve oauth-integration --fg security

# Short form aliases
/wgsd +1 oauth-integration
/wgsd lgtm oauth-integration
```

### Reaction Approval

React to an approval prompt message with:
- ✅ (`:white_check_mark:`)
- 👍 (`:thumbsup:`)
- ✔️ (`:heavy_check_mark:`)

### Thread Reply Approval

In a discussion thread, reply with:
- "approved"
- "LGTM"
- "+1"
- "ship it"
- "looks good"
- "approve"

---

## Workflow Steps

```yaml
workflow:
  name: conversational-approve
  version: "1.0"
  
  inputs:
    concept_name: string          # Required: Concept slug
    channel_id: string | null     # Optional: Current channel (for context)
    user_id: string               # Required: Approving user
    message_ts: string | null     # Optional: For reaction/thread context
    comment: string | null        # Optional: Approval comment
    focus_group: string | null    # Optional: Explicit FG override
  
  steps:
    - id: resolve_context
      action: infer_approval_context
      description: Determine FG from channel or explicit param
      
    - id: validate_user
      action: check_user_authority
      description: Verify user can approve for this FG
      
    - id: find_concept
      action: locate_concept_path
      description: Find concept directory
      
    - id: check_status
      action: verify_approvable
      description: Ensure concept is pending/blocked, not already approved
      
    - id: record_approval
      action: execute_approval
      description: Update matrix, clear blocks, check completion
      
    - id: send_confirmation
      action: post_confirmation
      description: Ephemeral confirmation to approver
      
    - id: update_channel
      action: notify_channel
      description: Post/update approval message in channel
      
    - id: check_full_approval
      action: trigger_roadmap_if_complete
      description: Auto-merge to roadmap if all FGs approved
```

---

## Context Inference

### infer_approval_context

Determine focus group from the approval context.

```bash
# Usage: infer_approval_context <channel_id> <user_id> <explicit_fg>
# Returns: Focus group slug or error
infer_approval_context() {
  local channel_id="$1"
  local user_id="$2"
  local explicit_fg="$3"
  local workspace="${4:-.}"
  
  # Priority 1: Explicit FG always wins
  if [ -n "$explicit_fg" ]; then
    echo "INFERRED:explicit:$explicit_fg"
    echo "$explicit_fg"
    return 0
  fi
  
  # Priority 2: Channel-based inference
  if [ -n "$channel_id" ]; then
    # Get channel name from registry or API
    local config_path="$workspace/.planning/WGSD-CONFIG.md"
    
    if [ -f "$config_path" ]; then
      # Look up channel in registry
      local channel_entry=$(grep "| $channel_id |" "$config_path" | head -1)
      
      if [ -n "$channel_entry" ]; then
        local channel_name=$(echo "$channel_entry" | awk -F'|' '{print $2}' | tr -d ' ')
        
        # Extract FG from channel name pattern: {stub}-fg-{fg_name}
        local fg=$(echo "$channel_name" | sed -n 's/.*-fg-\(.*\)/\1/p')
        
        if [ -n "$fg" ]; then
          echo "INFERRED:channel:$fg"
          echo "$fg"
          return 0
        fi
      fi
    fi
  fi
  
  # Priority 3: User's primary focus group
  if [ -n "$user_id" ]; then
    local user_fgs=$(approval_matrix_get_user_fgs "$user_id" "$workspace")
    local primary_fg=$(echo "$user_fgs" | awk '{print $1}')
    
    if [ -n "$primary_fg" ]; then
      echo "INFERRED:user_primary:$primary_fg"
      echo "$primary_fg"
      return 0
    fi
  fi
  
  echo "ERROR: Could not infer focus group"
  echo "Use --fg to specify which focus group you're approving for"
  return 1
}
```

### get_channel_focus_group

Get focus group associated with a Slack channel.

```bash
# Usage: get_channel_focus_group <channel_id> <workspace>
# Returns: Focus group slug or empty
get_channel_focus_group() {
  local channel_id="$1"
  local workspace="${2:-.}"
  
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  
  if [ ! -f "$config_path" ]; then
    return 1
  fi
  
  # Get channel name
  local channel_entry=$(grep "| $channel_id |" "$config_path" | head -1)
  
  if [ -z "$channel_entry" ]; then
    # Try Slack API lookup
    local token=$(slack_get_token)
    if [ -n "$token" ]; then
      local response=$(curl -s "https://slack.com/api/conversations.info?channel=$channel_id" \
        -H "Authorization: Bearer $token")
      local channel_name=$(echo "$response" | jq -r '.channel.name // empty')
      
      # Match against FG pattern
      if echo "$channel_name" | grep -q "\-fg-"; then
        local fg=$(echo "$channel_name" | sed -n 's/.*-fg-\(.*\)/\1/p')
        echo "$fg"
        return 0
      fi
    fi
    return 1
  fi
  
  local channel_name=$(echo "$channel_entry" | awk -F'|' '{print $2}' | tr -d ' ')
  
  # Extract focus group from naming pattern
  local fg=$(echo "$channel_name" | sed -n 's/.*-fg-\(.*\)/\1/p')
  
  if [ -n "$fg" ]; then
    echo "$fg"
    return 0
  fi
  
  return 1
}
```

---

## Core Approval Handler

### handle_conversational_approve

Main handler for all conversational approval methods.

```bash
# Usage: handle_conversational_approve <concept> <user_id> <channel_id> [--fg fg] [--comment comment]
handle_conversational_approve() {
  local concept_name=""
  local focus_group=""
  local user_id=""
  local channel_id=""
  local comment=""
  local message_ts=""
  local workspace="."
  
  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --fg|--focus-group)
        focus_group="$2"
        shift 2
        ;;
      --comment|-c)
        comment="$2"
        shift 2
        ;;
      --user|-u)
        user_id="$2"
        shift 2
        ;;
      --channel)
        channel_id="$2"
        shift 2
        ;;
      --message-ts|--ts)
        message_ts="$2"
        shift 2
        ;;
      --workspace|-w)
        workspace="$2"
        shift 2
        ;;
      *)
        if [ -z "$concept_name" ]; then
          concept_name="$1"
        fi
        shift
        ;;
    esac
  done
  
  # Validate required inputs
  if [ -z "$concept_name" ]; then
    echo "❌ Usage: /wgsd approve <concept> [--fg <focus_group>] [--comment <text>]"
    return 1
  fi
  
  if [ -z "$user_id" ]; then
    echo "❌ User context required"
    return 1
  fi
  
  echo "🔄 Processing approval..."
  
  # Step 1: Infer focus group context
  local inferred=$(infer_approval_context "$channel_id" "$user_id" "$focus_group" "$workspace")
  
  if echo "$inferred" | grep -q "^ERROR"; then
    echo "$inferred"
    return 1
  fi
  
  focus_group=$(echo "$inferred" | tail -1)
  local inference_source=$(echo "$inferred" | head -1 | cut -d: -f2)
  
  # Step 2: Validate user authority
  if [ "$(approval_matrix_can_approve "$user_id" "$focus_group")" != "true" ]; then
    echo "❌ You don't have authority to approve for *${focus_group}*"
    echo ""
    echo "💡 You can approve for: $(approval_matrix_get_user_fgs "$user_id" "$workspace" | xargs | tr ' ' ', ')"
    return 1
  fi
  
  # Step 3: Find concept
  local concept_path=$(find_concept_path "$concept_name" "$workspace")
  
  if [ -z "$concept_path" ]; then
    echo "❌ Concept not found: *${concept_name}*"
    echo ""
    echo "💡 Use \`/wgsd list-concepts\` to see available concepts"
    return 1
  fi
  
  local impact_file="$concept_path/impact-matrix.md"
  
  if [ ! -f "$impact_file" ]; then
    echo "❌ No impact matrix found for *${concept_name}*"
    return 1
  fi
  
  # Step 4: Check current status
  local impacts=$(impact_parse_file "$impact_file")
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ -z "$current_status" ] || [ "$current_status" = "null" ]; then
    echo "❌ Focus group *${focus_group}* has no declared impact on this concept"
    return 1
  fi
  
  if [ "$current_status" = "approved" ]; then
    echo "✅ Already approved! *${focus_group}* previously approved this concept."
    return 0
  fi
  
  if [ "$current_status" = "rejected" ]; then
    echo "⚠️  This concept was previously rejected by *${focus_group}*"
    echo "Use \`/wgsd approve ${concept_name} --force\` to override the rejection"
    return 1
  fi
  
  # Step 5: Execute approval
  local today=$(date +%Y-%m-%d)
  
  impact_update "$impact_file" "$focus_group" "status" "approved"
  impact_update "$impact_file" "$focus_group" "approver" "@$user_id"
  impact_update "$impact_file" "$focus_group" "approved_date" "$today"
  
  if [ -n "$comment" ]; then
    impact_update "$impact_file" "$focus_group" "approval_comment" "$comment"
  fi
  
  # Step 6: Auto-unblock dependents
  approval_matrix_auto_unblock "$concept_path" "$focus_group"
  
  # Step 7: Check for full approval
  local matrix_state=$(approval_matrix_get "$impact_file")
  local approved=$(echo "$matrix_state" | jq -r '.approved')
  local total=$(echo "$matrix_state" | jq -r '.total_approvals')
  local completion=$(echo "$matrix_state" | jq -r '.completion_percent')
  local fully_approved=$(echo "$matrix_state" | jq -r '.fully_approved')
  
  # Step 8: Output confirmation
  echo "✅ *Approved!*"
  echo ""
  echo "📋 *${concept_name}* approved by @${user_id} for *${focus_group}*"
  
  if [ -n "$comment" ]; then
    echo "💬 Comment: ${comment}"
  fi
  
  echo ""
  echo "📊 *Progress:* ${approved}/${total} (${completion}%)"
  
  # Generate progress bar
  local progress=$(generate_progress_bar "$completion")
  echo "$progress"
  
  # Step 9: Handle full approval
  if [ "$fully_approved" = "true" ]; then
    echo ""
    echo "🎉 *All focus groups have approved!*"
    echo ""
    echo "📍 Next steps:"
    echo "  • Auto-merging to \`roadmap\` branch..."
    echo "  • Create implementation: \`/wgsd create-implementation ${concept_name}\`"
    
    # Trigger roadmap merge
    echo ""
    echo "FULLY_APPROVED:$concept_name"
    trigger_roadmap_merge "$concept_path"
  else
    # Show remaining approvals
    local blockers=$(approval_matrix_get_blockers "$impact_file")
    local remaining=$(echo "$blockers" | jq -r '.[].focus_group' | tr '\n' ', ' | sed 's/,$//')
    
    echo ""
    echo "⏳ *Awaiting:* ${remaining}"
  fi
  
  # Step 10: Post channel notification
  if [ -n "$channel_id" ]; then
    post_approval_notification "$channel_id" "$concept_name" "$focus_group" "$user_id" "$comment"
  fi
  
  return 0
}
```

---

## Reaction Handler

### handle_reaction_approval

Process approval via emoji reaction on approval prompt.

```bash
# Usage: handle_reaction_approval <channel_id> <message_ts> <user_id> <reaction>
handle_reaction_approval() {
  local channel_id="$1"
  local message_ts="$2"
  local user_id="$3"
  local reaction="$4"
  local workspace="${5:-.}"
  
  # Valid approval reactions
  case "$reaction" in
    white_check_mark|heavy_check_mark|thumbsup|+1|check)
      ;;
    *)
      # Not an approval reaction
      return 0
      ;;
  esac
  
  echo "🔄 Processing reaction approval..."
  
  # Get concept from message context
  # In practice, we'd look up the message_ts in our tracking store
  # to find the associated concept and focus group
  
  local concept_slug=$(lookup_concept_from_message "$channel_id" "$message_ts" "$workspace")
  
  if [ -z "$concept_slug" ]; then
    echo "⚠️  Could not identify concept from this message"
    echo "💡 Use \`/wgsd approve <concept>\` instead"
    return 1
  fi
  
  # Infer focus group from channel
  local focus_group=$(get_channel_focus_group "$channel_id" "$workspace")
  
  if [ -z "$focus_group" ]; then
    echo "⚠️  Could not identify focus group from this channel"
    return 1
  fi
  
  # Execute approval
  handle_conversational_approve "$concept_slug" \
    --user "$user_id" \
    --channel "$channel_id" \
    --fg "$focus_group" \
    --workspace "$workspace"
}
```

---

## Thread Reply Handler

### handle_thread_approval

Process approval via thread reply keywords.

```bash
# Usage: handle_thread_approval <channel_id> <thread_ts> <user_id> <message_text>
handle_thread_approval() {
  local channel_id="$1"
  local thread_ts="$2"
  local user_id="$3"
  local message_text="$4"
  local workspace="${5:-.}"
  
  # Normalize message
  local normalized=$(echo "$message_text" | tr '[:upper:]' '[:lower:]' | tr -d '[:punct:]')
  
  # Check for approval keywords
  local is_approval="false"
  local comment=""
  
  case "$normalized" in
    "approved"|"approve"|"lgtm"|"looks good"|"ship it"|"shipit"|"+1")
      is_approval="true"
      ;;
    lgtm*)
      is_approval="true"
      comment=$(echo "$message_text" | sed 's/^[Ll][Gg][Tt][Mm][[:space:]]*//')
      ;;
    "looks good to me"|"lg tm")
      is_approval="true"
      ;;
  esac
  
  if [ "$is_approval" != "true" ]; then
    # Not an approval message - just a regular thread reply
    return 0
  fi
  
  echo "🔄 Detected approval in thread reply..."
  
  # Get concept from thread context
  local concept_slug=$(lookup_concept_from_thread "$channel_id" "$thread_ts" "$workspace")
  
  if [ -z "$concept_slug" ]; then
    echo "ℹ️  Thread not associated with an approval request"
    return 0
  fi
  
  # Infer focus group from channel
  local focus_group=$(get_channel_focus_group "$channel_id" "$workspace")
  
  if [ -z "$focus_group" ]; then
    echo "⚠️  Could not identify focus group"
    return 1
  fi
  
  # Execute approval
  handle_conversational_approve "$concept_slug" \
    --user "$user_id" \
    --channel "$channel_id" \
    --fg "$focus_group" \
    --comment "$comment" \
    --workspace "$workspace"
}
```

---

## Notification Functions

### post_approval_notification

Post approval confirmation to channel.

```bash
# Usage: post_approval_notification <channel_id> <concept> <focus_group> <approver> [comment]
post_approval_notification() {
  local channel_id="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local approver="$4"
  local comment="${5:-}"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  # Generate confirmation message
  local message_json=$(template_approval_confirmed "$concept_slug" "$focus_group" "$approver" "$comment")
  local text=$(echo "$message_json" | jq -r '.text')
  local blocks=$(echo "$message_json" | jq -c '.blocks')
  
  # Post to channel
  local response=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg text "$text" \
      --argjson blocks "$blocks" \
      '{channel: $channel, text: $text, blocks: $blocks}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    echo "NOTIFICATION_SENT:$channel_id"
    return 0
  else
    echo "WARNING: Could not send notification: $(echo "$response" | jq -r '.error')"
    return 0  # Non-fatal
  fi
}
```

### trigger_roadmap_merge

Trigger roadmap merge when fully approved.

```bash
# Usage: trigger_roadmap_merge <concept_path>
trigger_roadmap_merge() {
  local concept_path="$1"
  local concept_slug=$(basename "$concept_path")
  
  echo "🔀 Triggering roadmap merge for ${concept_slug}..."
  
  # Call the merge-to-roadmap workflow
  # In practice, this would invoke the workflow directly
  
  echo "TRIGGER:merge-to-roadmap:$concept_slug"
  
  # The actual merge is handled by workflows/merge-to-roadmap.md
  # which was implemented in Phase 13
  
  return 0
}
```

---

## Message Tracking

### track_approval_prompt

Store mapping from message to concept for reaction tracking.

```bash
# Usage: track_approval_prompt <channel_id> <message_ts> <concept_slug> <focus_group>
track_approval_prompt() {
  local channel_id="$1"
  local message_ts="$2"
  local concept_slug="$3"
  local focus_group="$4"
  local workspace="${5:-.}"
  
  local tracking_file="$workspace/.planning/approval-tracking.json"
  
  # Initialize if not exists
  if [ ! -f "$tracking_file" ]; then
    echo '{"prompts": {}}' > "$tracking_file"
  fi
  
  # Add tracking entry
  local key="${channel_id}:${message_ts}"
  
  python3 -c "
import json
from datetime import datetime

with open('$tracking_file', 'r') as f:
    data = json.load(f)

data['prompts']['$key'] = {
    'channel_id': '$channel_id',
    'message_ts': '$message_ts',
    'concept': '$concept_slug',
    'focus_group': '$focus_group',
    'tracked_at': datetime.utcnow().isoformat() + 'Z'
}

with open('$tracking_file', 'w') as f:
    json.dump(data, f, indent=2)

print('TRACKED:$key')
"
  return 0
}
```

### lookup_concept_from_message

Look up concept associated with a message.

```bash
# Usage: lookup_concept_from_message <channel_id> <message_ts> <workspace>
lookup_concept_from_message() {
  local channel_id="$1"
  local message_ts="$2"
  local workspace="${3:-.}"
  
  local tracking_file="$workspace/.planning/approval-tracking.json"
  
  if [ ! -f "$tracking_file" ]; then
    return 1
  fi
  
  local key="${channel_id}:${message_ts}"
  
  python3 -c "
import json

with open('$tracking_file', 'r') as f:
    data = json.load(f)

entry = data.get('prompts', {}).get('$key')
if entry:
    print(entry['concept'])
else:
    exit(1)
" 2>/dev/null
}
```

### lookup_concept_from_thread

Look up concept from discussion thread.

```bash
# Usage: lookup_concept_from_thread <channel_id> <thread_ts> <workspace>
lookup_concept_from_thread() {
  local channel_id="$1"
  local thread_ts="$2"
  local workspace="${3:-.}"
  
  local tracking_file="$workspace/.planning/approval-tracking.json"
  
  if [ ! -f "$tracking_file" ]; then
    return 1
  fi
  
  # Thread replies have thread_ts matching the parent message
  # So we look up by parent message ts
  lookup_concept_from_message "$channel_id" "$thread_ts" "$workspace"
}
```

---

## Quick Aliases

### Handle shorthand commands

```bash
# /wgsd +1 <concept> → /wgsd approve <concept>
# /wgsd lgtm <concept> → /wgsd approve <concept> --comment "LGTM"
# /wgsd ship <concept> → /wgsd approve <concept> --comment "Ship it!"

handle_approval_alias() {
  local alias="$1"
  shift
  local args="$@"
  
  case "$alias" in
    "+1"|"plus1")
      handle_conversational_approve $args
      ;;
    "lgtm")
      handle_conversational_approve $args --comment "LGTM"
      ;;
    "ship"|"shipit")
      handle_conversational_approve $args --comment "Ship it! 🚀"
      ;;
    *)
      echo "Unknown alias: $alias"
      return 1
      ;;
  esac
}
```

---

## Usage Examples

```bash
# Simple approval from FG channel (FG inferred)
/wgsd approve oauth-integration

# Approval with comment
/wgsd approve oauth-integration --comment "Token handling looks solid"

# Approval for specific FG (from any channel)
/wgsd approve oauth-integration --fg security

# Quick aliases
/wgsd lgtm oauth-integration
/wgsd +1 oauth-integration

# Reaction: ✅ on approval prompt message → auto-approve

# Thread reply: "LGTM" in discussion thread → auto-approve
```

---

## Error Messages

| Situation | Message |
|-----------|---------|
| No FG inferred | "Could not infer focus group. Use --fg to specify." |
| No authority | "You don't have authority to approve for {fg}" |
| Already approved | "Already approved! {fg} previously approved this concept." |
| Concept not found | "Concept not found: {name}" |
| No impact declared | "{fg} has no declared impact on this concept" |
| Previously rejected | "This concept was previously rejected. Use --force to override." |

---

*Workflow created for WGSD Phase 14 - Slack-Native Approval Workflow*
