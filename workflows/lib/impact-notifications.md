---
name: wgsd:lib:impact-notifications
description: Slack notification system for cross-cutting impact declarations with Phase 14 rich approval prompts
dependencies:
  - workflows/lib/approval-templates.md
  - workflows/lib/slack-api.md
  - workflows/conversational-approve.md
---

# Impact Notifications Library

Send and track notifications to focus groups when concepts declare impacts on them.

---

## Overview

When a concept declares impact on a focus group:
1. Focus group channel receives a **rich approval prompt** (Phase 14)
2. Notification includes priority, type, description, and approval matrix state
3. **Quick actions** enable approve/reject/block directly from message
4. **Discussion threads** capture concerns before approval
5. Re-notification occurs when impacts change
6. **Message tracking** enables reaction-based approvals

---

## Notification Types

| Type | Trigger | Content |
|------|---------|---------|
| `NEW_IMPACT` | New impact declared | Full notification with concept details |
| `UPDATED_IMPACT` | Priority/type/desc changed | Update notification with changes |
| `REMOVED_IMPACT` | Impact removed | Removal notification |
| `APPROVED_IMPACT` | Impact approved | Confirmation to concept channel |
| `REJECTED_IMPACT` | Impact rejected | Rejection with feedback |

---

## impact_notify_new

Send rich approval prompt for a new impact declaration (Phase 14 enhanced).

```bash
# Usage: impact_notify_new <workspace> <concept_slug> <impact_json>
# Returns: Success/error with message ID, tracks message for reaction-based approval
impact_notify_new() {
  local workspace="$1"
  local concept_slug="$2"
  local impact_json="$3"
  
  # Parse impact details
  local focus_group=$(echo "$impact_json" | jq -r '.focus_group')
  local priority=$(echo "$impact_json" | jq -r '.priority')
  local type=$(echo "$impact_json" | jq -r '.type')
  local description=$(echo "$impact_json" | jq -r '.description')
  
  # Get project stub from registry
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  local stub=""
  if [ -f "$config_path" ]; then
    stub=$(grep "^\*\*Stub:\*\*" "$config_path" | awk '{print $2}')
    [ -z "$stub" ] && stub=$(grep "^stub:" "$config_path" | awk '{print $2}')
  fi
  
  if [ -z "$stub" ]; then
    echo "ERROR: Could not determine project stub"
    return 1
  fi
  
  # Build channel name for focus group
  local channel_name="${stub}-fg-${focus_group}"
  
  # Get channel ID from registry
  local channel_line=$(grep "| $channel_name |" "$config_path" 2>/dev/null | head -1)
  local channel_id=""
  
  if [ -n "$channel_line" ]; then
    channel_id=$(echo "$channel_line" | awk -F'|' '{print $3}' | tr -d ' ')
  fi
  
  if [ -z "$channel_id" ]; then
    echo "⚠️  Channel not found in registry: $channel_name"
    echo "CHANNEL_NOT_FOUND:$channel_name"
    # Don't fail - channel might not exist yet
    return 0
  fi
  
  # Phase 14: Generate rich approval prompt using template
  echo "📤 Sending rich approval prompt to #$channel_name..."
  
  local token=$(jq -r '.channels.slack.botToken // empty' "$HOME/.openclaw/openclaw.json")
  if [ -z "$token" ]; then
    echo "ERROR: No Slack token found"
    return 1
  fi
  
  # Generate rich Block Kit message using approval template
  local prompt_json=$(template_approval_prompt "$workspace" "$concept_slug" "$focus_group" "$impact_json")
  local text=$(echo "$prompt_json" | jq -r '.text')
  local blocks=$(echo "$prompt_json" | jq -c '.blocks')
  
  # Send rich message
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
    local ts=$(echo "$response" | jq -r '.ts')
    echo "✅ Rich approval prompt sent to #$channel_name"
    echo "MESSAGE_ID:$ts"
    echo "CHANNEL_ID:$channel_id"
    
    # Phase 14: Track message for reaction-based approval
    track_approval_prompt "$channel_id" "$ts" "$concept_slug" "$focus_group" "$workspace"
    
    # Phase 14: Create discussion thread starter
    create_discussion_thread "$channel_id" "$ts" "$concept_slug" "$focus_group" ""
    
    # Phase 14: Update impact-matrix.md with prompt tracking
    local concept_path=$(find_concept_path "$concept_slug" "$workspace")
    if [ -n "$concept_path" ]; then
      local impact_file="$concept_path/impact-matrix.md"
      track_prompt_in_matrix "$impact_file" "$focus_group" "$channel_id" "$ts"
    fi
    
    return 0
  else
    local error=$(echo "$response" | jq -r '.error')
    echo "ERROR: Failed to send notification: $error"
    return 1
  fi
}

# Phase 14: Track prompt message in impact-matrix.md
track_prompt_in_matrix() {
  local impact_file="$1"
  local focus_group="$2"
  local channel_id="$3"
  local message_ts="$4"
  
  if [ ! -f "$impact_file" ]; then
    return 1
  fi
  
  python3 -c "
import re
import yaml
from datetime import datetime

with open('$impact_file', 'r') as f:
    content = f.read()

yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    exit(1)

yaml_content = yaml_match.group(1)
data = yaml.safe_load(yaml_content)

# Initialize approval_prompts if not exists
if 'approval_prompts' not in data:
    data['approval_prompts'] = []

# Add prompt tracking
data['approval_prompts'].append({
    'channel_id': '$channel_id',
    'message_ts': '$message_ts',
    'focus_group': '$focus_group',
    'posted_at': datetime.utcnow().isoformat() + 'Z'
})

new_yaml = yaml.dump(data, default_flow_style=False, allow_unicode=True, sort_keys=False)
new_content = content[:yaml_match.start()] + '\`\`\`yaml\n---\n' + new_yaml + '---\n\`\`\`' + content[yaml_match.end():]

with open('$impact_file', 'w') as f:
    f.write(new_content)
" 2>/dev/null || true
  
  return 0
}
```

---

## impact_notify_updated

Send notification when an impact is updated.

```bash
# Usage: impact_notify_updated <workspace> <concept_slug> <focus_group> <changes_json>
# changes_json: {"field": "priority", "old": "P2", "new": "P0"}
impact_notify_updated() {
  local workspace="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local changes_json="$4"
  
  # Get channel info
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  local stub=""
  if [ -f "$config_path" ]; then
    stub=$(grep "^\*\*Stub:\*\*" "$config_path" | awk '{print $2}')
    [ -z "$stub" ] && stub=$(grep "^stub:" "$config_path" | awk '{print $2}')
  fi
  
  local channel_name="${stub}-fg-${focus_group}"
  local channel_line=$(grep "| $channel_name |" "$config_path" 2>/dev/null | head -1)
  local channel_id=$(echo "$channel_line" | awk -F'|' '{print $3}' | tr -d ' ')
  
  if [ -z "$channel_id" ]; then
    echo "⚠️  Channel not found: $channel_name"
    return 0
  fi
  
  # Parse changes
  local change_desc=$(echo "$changes_json" | python3 -c "
import json
import sys
changes = json.load(sys.stdin)
if isinstance(changes, list):
    parts = []
    for c in changes:
        parts.append(f\"{c['field']}: {c['old']} → {c['new']}\")
    print('\n• '.join([''] + parts))
else:
    print(f\"• {changes['field']}: {changes['old']} → {changes['new']}\")
")

  local message="⚠️ *Impact Updated: ${concept_slug}*

The impact declaration for *#${channel_name}* has been modified:

*Changes:*${change_desc}

📋 *Action Required:*
• Review updated impact: \`/wgsd concept ${concept_slug}\`
• Re-approve if needed: \`/wgsd approve ${concept_slug}\`

_Concept: ${concept_slug} | Focus Group: ${focus_group}_"

  # Send notification
  local token=$(jq -r '.channels.slack.botToken // empty' "$HOME/.openclaw/openclaw.json")
  
  local response=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg text "$message" \
      '{channel: $channel, text: $text, mrkdwn: true}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    echo "✅ Update notification sent to #$channel_name"
    return 0
  else
    echo "ERROR: Failed to send update notification"
    return 1
  fi
}
```

---

## impact_notify_removed

Send notification when an impact is removed.

```bash
# Usage: impact_notify_removed <workspace> <concept_slug> <focus_group>
impact_notify_removed() {
  local workspace="$1"
  local concept_slug="$2"
  local focus_group="$3"
  
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  local stub=""
  if [ -f "$config_path" ]; then
    stub=$(grep "^\*\*Stub:\*\*" "$config_path" | awk '{print $2}')
    [ -z "$stub" ] && stub=$(grep "^stub:" "$config_path" | awk '{print $2}')
  fi
  
  local channel_name="${stub}-fg-${focus_group}"
  local channel_line=$(grep "| $channel_name |" "$config_path" 2>/dev/null | head -1)
  local channel_id=$(echo "$channel_line" | awk -F'|' '{print $3}' | tr -d ' ')
  
  if [ -z "$channel_id" ]; then
    echo "⚠️  Channel not found: $channel_name"
    return 0
  fi
  
  local message="✅ *Impact Removed: ${concept_slug}*

This concept no longer declares impact on *#${channel_name}*.

No further review required for this focus group.

_Concept: ${concept_slug}_"

  local token=$(jq -r '.channels.slack.botToken // empty' "$HOME/.openclaw/openclaw.json")
  
  local response=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg text "$message" \
      '{channel: $channel, text: $text, mrkdwn: true}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    echo "✅ Removal notification sent to #$channel_name"
    return 0
  else
    echo "ERROR: Failed to send removal notification"
    return 1
  fi
}
```

---

## impact_notify_batch

Send notifications for multiple impacts at once.

```bash
# Usage: impact_notify_batch <workspace> <concept_slug> <impacts_json>
# Notifies all unnotified impacts and marks them as notified
impact_notify_batch() {
  local workspace="$1"
  local concept_slug="$2"
  local impacts_json="$3"
  
  local sent=0
  local failed=0
  
  # Process each impact
  echo "$impacts_json" | jq -c '.[]' | while read -r impact; do
    local notified=$(echo "$impact" | jq -r '.notified // false')
    
    if [ "$notified" != "true" ]; then
      if impact_notify_new "$workspace" "$concept_slug" "$impact"; then
        ((sent++))
      else
        ((failed++))
      fi
    fi
  done
  
  echo ""
  echo "📊 Notification Summary:"
  echo "   Sent: $sent"
  echo "   Failed: $failed"
  
  return 0
}
```

---

## impact_notify_all_changes

Process an impact diff and send appropriate notifications.

```bash
# Usage: impact_notify_all_changes <workspace> <concept_slug> <diff_json>
# diff_json: Output from impact_diff
impact_notify_all_changes() {
  local workspace="$1"
  local concept_slug="$2"
  local diff_json="$3"
  
  local has_changes=$(echo "$diff_json" | jq -r '.has_changes')
  
  if [ "$has_changes" != "true" ]; then
    echo "ℹ️  No impact changes detected"
    return 0
  fi
  
  echo "📤 Processing impact changes..."
  
  # Notify new impacts
  local added=$(echo "$diff_json" | jq -r '.added[]' 2>/dev/null)
  for fg in $added; do
    echo "  + New impact: $fg"
    # Note: Full impact details needed - caller should provide
  done
  
  # Notify removed impacts
  local removed=$(echo "$diff_json" | jq -r '.removed[]' 2>/dev/null)
  for fg in $removed; do
    echo "  - Removed impact: $fg"
    impact_notify_removed "$workspace" "$concept_slug" "$fg"
  done
  
  # Notify modified impacts
  echo "$diff_json" | jq -c '.modified[]' 2>/dev/null | while read -r mod; do
    local fg=$(echo "$mod" | jq -r '.focus_group')
    local changes=$(echo "$mod" | jq -c '.changes')
    echo "  ~ Modified impact: $fg"
    impact_notify_updated "$workspace" "$concept_slug" "$fg" "$changes"
  done
  
  echo "✅ Change notifications sent"
  return 0
}
```

---

## impact_mark_notified

Mark impacts as notified in the impact-matrix.md file.

```bash
# Usage: impact_mark_notified <impact_matrix_path> <focus_groups>
# focus_groups: Space-separated list of focus group names
impact_mark_notified() {
  local file_path="$1"
  shift
  local focus_groups="$@"
  
  local today=$(date +%Y-%m-%d)
  
  for fg in $focus_groups; do
    python3 -c "
import re
import yaml

with open('$file_path', 'r') as f:
    content = f.read()

yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    exit(1)

yaml_content = yaml_match.group(1)
data = yaml.safe_load(yaml_content)

for impact in data.get('impacts', []):
    if impact.get('focus_group') == '$fg':
        impact['notified'] = True
        impact['notified_date'] = '$today'
        break

new_yaml = yaml.dump(data, default_flow_style=False, allow_unicode=True, sort_keys=False)
new_content = content[:yaml_match.start()] + '\`\`\`yaml\n---\n' + new_yaml + '---\n\`\`\`' + content[yaml_match.end():]

with open('$file_path', 'w') as f:
    f.write(new_content)
"
    if [ $? -eq 0 ]; then
      echo "✅ Marked $fg as notified"
    else
      echo "⚠️  Failed to mark $fg as notified"
    fi
  done
  
  return 0
}
```

---

## impact_get_channel_for_fg

Resolve focus group name to Slack channel ID.

```bash
# Usage: impact_get_channel_for_fg <workspace> <focus_group>
# Returns: CHANNEL_ID or error
impact_get_channel_for_fg() {
  local workspace="$1"
  local focus_group="$2"
  
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  # Get stub
  local stub=$(grep "^\*\*Stub:\*\*" "$config_path" | awk '{print $2}')
  [ -z "$stub" ] && stub=$(grep "^stub:" "$config_path" | awk '{print $2}')
  
  if [ -z "$stub" ]; then
    echo "ERROR: Could not determine project stub"
    return 1
  fi
  
  # Build expected channel name
  local channel_name="${stub}-fg-${focus_group}"
  
  # Look up in registry
  local channel_line=$(grep "| $channel_name |" "$config_path" | head -1)
  
  if [ -z "$channel_line" ]; then
    echo "ERROR: Focus group channel not found: $channel_name"
    return 1
  fi
  
  local channel_id=$(echo "$channel_line" | awk -F'|' '{print $3}' | tr -d ' ')
  
  if [ -z "$channel_id" ]; then
    echo "ERROR: Could not extract channel ID for $channel_name"
    return 1
  fi
  
  echo "$channel_id"
  return 0
}
```

---

## impact_list_focus_group_channels

List all focus group channels for a workspace.

```bash
# Usage: impact_list_focus_group_channels <workspace>
# Returns: List of focus group name → channel ID mappings
impact_list_focus_group_channels() {
  local workspace="$1"
  local config_path="$workspace/.planning/WGSD-CONFIG.md"
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  echo "📋 Focus Group Channels:"
  
  grep "| fg |" "$config_path" | while read -r line; do
    local channel=$(echo "$line" | awk -F'|' '{print $2}' | tr -d ' ')
    local id=$(echo "$line" | awk -F'|' '{print $3}' | tr -d ' ')
    local status=$(echo "$line" | awk -F'|' '{print $5}' | tr -d ' ')
    
    # Extract focus group name from channel name
    local fg_name=$(echo "$channel" | sed 's/.*-fg-//')
    
    local status_icon="✅"
    [ "$status" = "archived" ] && status_icon="📦"
    
    echo "  $status_icon $fg_name → #$channel ($id)"
  done
  
  return 0
}
```

---

## Usage Examples

```bash
# Send notification for a new impact
impact_notify_new "/path/to/workspace" "oauth-integration" \
  '{"focus_group": "security", "priority": "P1", "type": "integration", "description": "Token validation changes"}'

# Send update notification
impact_notify_updated "/path/to/workspace" "oauth-integration" "security" \
  '{"field": "priority", "old": "P2", "new": "P0"}'

# Send removal notification
impact_notify_removed "/path/to/workspace" "oauth-integration" "frontend"

# Process all changes from a diff
impact_notify_all_changes "/path/to/workspace" "oauth-integration" "$diff_json"

# Mark impacts as notified after sending
impact_mark_notified "concepts/oauth-integration/impact-matrix.md" security api

# Get channel ID for a focus group
channel_id=$(impact_get_channel_for_fg "/path/to/workspace" "security")
```

---

*Library created for WGSD Phase 11 - Cross-Cutting Impact System*
