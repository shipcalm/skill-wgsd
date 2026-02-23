---
name: wgsd:workflow:concept-status
description: Approval status summary command for Slack-native status display
triggers:
  - command: "/wgsd status {concept}"
  - command: "/wgsd approval-status {concept}"
  - command: "/wgsd show-approvals {concept}"
  - command: "/wgsd status"
---

# Concept Status Workflow

Display approval status summaries with visual progress indicators in Slack.

---

## Overview

The status workflow provides at-a-glance approval visibility:

1. **Per-concept status** - Full approval matrix for a specific concept
2. **Channel status** - All pending approvals for current focus group
3. **Daily digest** - Summary of what needs attention
4. **Progress visualization** - Color-coded status with progress bars

---

## Entry Points

### Concept Status

```bash
# Full status for a specific concept
/wgsd status oauth-integration

# Aliases
/wgsd approval-status oauth-integration
/wgsd show-approvals oauth-integration
```

### Channel Status

```bash
# All pending approvals in current FG channel
/wgsd status

# Explicit focus group
/wgsd status --fg security
```

### Daily Digest

```bash
# Generate and post daily summary to FG channel
/wgsd status --digest

# Post to specific channel
/wgsd status --digest --channel C0123456789
```

---

## Workflow Steps

```yaml
workflow:
  name: concept-status
  version: "1.0"
  
  inputs:
    concept_name: string | null   # Optional: Specific concept
    focus_group: string | null    # Optional: Focus group filter
    channel_id: string | null     # Optional: Current channel context
    digest_mode: boolean          # Whether to generate digest
    user_id: string               # Requesting user
  
  steps:
    - id: determine_mode
      action: check_status_type
      description: Concept-specific vs channel-wide vs digest
      
    - id: resolve_context
      action: infer_fg_context
      description: Get focus group from channel if needed
      
    - id: gather_status
      action: collect_approval_data
      description: Load relevant approval matrices
      
    - id: format_output
      action: generate_status_message
      description: Create rich Slack message
      
    - id: deliver
      action: post_or_return
      description: Post to channel or return ephemeral
```

---

## Core Status Functions

### handle_concept_status

Main handler for status commands.

```bash
# Usage: handle_concept_status [concept] [--fg fg] [--digest] [--channel channel]
handle_concept_status() {
  local concept_name=""
  local focus_group=""
  local channel_id=""
  local digest_mode="false"
  local user_id=""
  local workspace="."
  
  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --fg|--focus-group)
        focus_group="$2"
        shift 2
        ;;
      --channel)
        channel_id="$2"
        shift 2
        ;;
      --digest)
        digest_mode="true"
        shift
        ;;
      --user|-u)
        user_id="$2"
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
  
  # Determine mode and execute
  if [ "$digest_mode" = "true" ]; then
    # Channel digest mode
    generate_channel_digest "$focus_group" "$channel_id" "$workspace"
  elif [ -n "$concept_name" ]; then
    # Specific concept status
    show_concept_status "$concept_name" "$channel_id" "$workspace"
  else
    # Channel-wide pending status
    show_channel_pending "$focus_group" "$channel_id" "$workspace"
  fi
}
```

---

### show_concept_status

Display detailed status for a specific concept.

```bash
# Usage: show_concept_status <concept_name> [channel_id] [workspace]
show_concept_status() {
  local concept_name="$1"
  local channel_id="$2"
  local workspace="${3:-.}"
  
  # Find concept
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
  
  # Get matrix state
  local matrix=$(approval_matrix_get "$impact_file")
  
  if echo "$matrix" | grep -q "error"; then
    echo "❌ Could not load approval matrix"
    return 1
  fi
  
  # Extract summary data
  local total=$(echo "$matrix" | jq -r '.total_approvals')
  local approved=$(echo "$matrix" | jq -r '.approved')
  local pending=$(echo "$matrix" | jq -r '.pending')
  local rejected=$(echo "$matrix" | jq -r '.rejected')
  local blocked=$(echo "$matrix" | jq -r '.blocked')
  local completion=$(echo "$matrix" | jq -r '.completion_percent')
  local fully_approved=$(echo "$matrix" | jq -r '.fully_approved')
  
  # Determine overall status
  local status_icon="⏳"
  local status_text="In Review"
  
  if [ "$fully_approved" = "true" ]; then
    status_icon="✅"
    status_text="Fully Approved"
  elif [ "$rejected" -gt 0 ]; then
    status_icon="🚫"
    status_text="Needs Revision"
  elif [ "$blocked" -gt 0 ]; then
    status_icon="⏸️"
    status_text="Blocked"
  fi
  
  # Generate progress bar
  local progress_bar=$(generate_progress_bar "$completion")
  
  # Header
  echo "${status_icon} *Approval Status: ${concept_name}*"
  echo ""
  echo "*Status:* ${status_text}"
  echo "*Progress:* ${approved}/${total} approvals (${completion}%)"
  echo ""
  echo "$progress_bar"
  echo ""
  
  # Per-FG breakdown
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "*Focus Group Breakdown:*"
  echo ""
  
  # Parse and display each FG
  echo "$matrix" | jq -c '.details[]' | while IFS= read -r detail; do
    local fg=$(echo "$detail" | jq -r '.focus_group')
    local fg_status=$(echo "$detail" | jq -r '.status')
    local priority=$(echo "$detail" | jq -r '.priority // "?"')
    local approver=$(echo "$detail" | jq -r '.approver // ""')
    local approved_date=$(echo "$detail" | jq -r '.approved_date // ""')
    local blocked_by=$(echo "$detail" | jq -r '.blocked_by // ""')
    local blocked_reason=$(echo "$detail" | jq -r '.blocked_reason // ""')
    local approval_comment=$(echo "$detail" | jq -r '.approval_comment // ""')
    
    # Status icon
    local fg_icon=""
    case "$fg_status" in
      approved) fg_icon="✅" ;;
      pending) fg_icon="⏳" ;;
      rejected) fg_icon="🚫" ;;
      blocked) fg_icon="⏸️" ;;
      *) fg_icon="❓" ;;
    esac
    
    # Priority badge
    local p_icon=""
    case "$priority" in
      P0) p_icon="🔴" ;;
      P1) p_icon="🟠" ;;
      P2) p_icon="🟡" ;;
      P3) p_icon="🟢" ;;
    esac
    
    echo "${fg_icon} *${fg}* (${p_icon} ${priority})"
    
    case "$fg_status" in
      approved)
        echo "    └─ Approved by ${approver} on ${approved_date}"
        if [ -n "$approval_comment" ] && [ "$approval_comment" != "null" ]; then
          echo "    └─ 💬 ${approval_comment}"
        fi
        ;;
      pending)
        echo "    └─ Awaiting review"
        ;;
      rejected)
        echo "    └─ Rejected by ${approver}"
        if [ -n "$approval_comment" ] && [ "$approval_comment" != "null" ]; then
          echo "    └─ 📝 Feedback: ${approval_comment}"
        fi
        ;;
      blocked)
        echo "    └─ Blocked by ${blocked_by}"
        if [ -n "$blocked_reason" ] && [ "$blocked_reason" != "null" ]; then
          echo "    └─ 📝 Reason: ${blocked_reason}"
        fi
        ;;
    esac
    echo ""
  done
  
  # Summary footer
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "📊 *Summary:* ✅ ${approved} | ⏳ ${pending} | 🚫 ${rejected} | ⏸️ ${blocked}"
  
  # Next action suggestion
  if [ "$fully_approved" = "true" ]; then
    echo ""
    echo "🚀 *Ready for implementation!*"
    echo "   \`/wgsd create-implementation ${concept_name}\`"
  elif [ "$rejected" -gt 0 ]; then
    echo ""
    echo "📝 *Action:* Address rejection feedback and request re-review"
  elif [ "$pending" -gt 0 ]; then
    local pending_fgs=$(echo "$matrix" | jq -r '.details[] | select(.status == "pending") | .focus_group' | head -3 | tr '\n' ', ' | sed 's/,$//')
    echo ""
    echo "⏳ *Pending review from:* ${pending_fgs}"
  fi
  
  return 0
}
```

---

### show_channel_pending

Show all pending approvals for a focus group channel.

```bash
# Usage: show_channel_pending <focus_group> [channel_id] [workspace]
show_channel_pending() {
  local focus_group="$1"
  local channel_id="$2"
  local workspace="${3:-.}"
  
  # Infer FG from channel if not provided
  if [ -z "$focus_group" ] && [ -n "$channel_id" ]; then
    focus_group=$(get_channel_focus_group "$channel_id" "$workspace")
  fi
  
  if [ -z "$focus_group" ]; then
    echo "❌ Could not determine focus group"
    echo "💡 Use \`/wgsd status --fg <focus_group>\` to specify"
    return 1
  fi
  
  echo "📋 *Pending Approvals for ${focus_group}*"
  echo ""
  
  # Find all concepts with pending impacts for this FG
  local found_any="false"
  
  # Search through all focus groups for concepts
  for fg_dir in "$workspace/.planning/focus-groups"/*/; do
    [ -d "$fg_dir" ] || continue
    
    local concepts_dir="$fg_dir/concepts"
    [ -d "$concepts_dir" ] || continue
    
    for concept_dir in "$concepts_dir"/*/; do
      [ -d "$concept_dir" ] || continue
      
      local impact_file="$concept_dir/impact-matrix.md"
      [ -f "$impact_file" ] || continue
      
      local concept_name=$(basename "$concept_dir")
      
      # Check if this FG has pending impact
      local impacts=$(impact_parse_file "$impact_file" 2>/dev/null)
      [ -z "$impacts" ] && continue
      
      local fg_impact=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg)')
      [ -z "$fg_impact" ] && continue
      
      local status=$(echo "$fg_impact" | jq -r '.status')
      
      if [ "$status" = "pending" ]; then
        found_any="true"
        
        local priority=$(echo "$fg_impact" | jq -r '.priority // "P2"')
        local type=$(echo "$fg_impact" | jq -r '.type // "integration"')
        local description=$(echo "$fg_impact" | jq -r '.description // "No description"')
        
        # Priority icon
        local p_icon=""
        case "$priority" in
          P0) p_icon="🔴" ;;
          P1) p_icon="🟠" ;;
          P2) p_icon="🟡" ;;
          P3) p_icon="🟢" ;;
        esac
        
        echo "${p_icon} *${concept_name}* (${priority})"
        echo "    └─ Type: ${type}"
        echo "    └─ ${description}"
        echo "    └─ \`/wgsd approve ${concept_name}\`"
        echo ""
      fi
    done
  done
  
  if [ "$found_any" = "false" ]; then
    echo "✨ No pending approvals for *${focus_group}*!"
    echo ""
    echo "💡 All caught up. Check \`/wgsd status --all\` for other focus groups."
  fi
  
  # Show blocked items too
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  show_blocked_items "$focus_group" "$workspace"
  
  return 0
}
```

---

### show_blocked_items

Show items blocked by or waiting on this focus group.

```bash
# Usage: show_blocked_items <focus_group> [workspace]
show_blocked_items() {
  local focus_group="$1"
  local workspace="${2:-.}"
  
  echo "*⏸️ Blocked Items:*"
  echo ""
  
  local found_blocked="false"
  local found_blocking="false"
  
  for fg_dir in "$workspace/.planning/focus-groups"/*/; do
    [ -d "$fg_dir" ] || continue
    
    local concepts_dir="$fg_dir/concepts"
    [ -d "$concepts_dir" ] || continue
    
    for concept_dir in "$concepts_dir"/*/; do
      [ -d "$concept_dir" ] || continue
      
      local impact_file="$concept_dir/impact-matrix.md"
      [ -f "$impact_file" ] || continue
      
      local concept_name=$(basename "$concept_dir")
      local impacts=$(impact_parse_file "$impact_file" 2>/dev/null)
      [ -z "$impacts" ] && continue
      
      # Check if this FG is blocked
      local blocked_impact=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
        '.[] | select(.focus_group == $fg and .status == "blocked")')
      
      if [ -n "$blocked_impact" ]; then
        found_blocked="true"
        local blocked_by=$(echo "$blocked_impact" | jq -r '.blocked_by')
        echo "⏸️ *${concept_name}* - waiting on ${blocked_by}"
      fi
      
      # Check if this FG is blocking others
      local blocking=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
        '.[] | select(.blocked_by == $fg) | .focus_group')
      
      if [ -n "$blocking" ]; then
        found_blocking="true"
        echo "⚡ *${concept_name}* - blocking ${blocking} (approve to unblock)"
      fi
    done
  done
  
  if [ "$found_blocked" = "false" ] && [ "$found_blocking" = "false" ]; then
    echo "✅ No blocked items"
  fi
}
```

---

### generate_channel_digest

Generate daily digest of approval status.

```bash
# Usage: generate_channel_digest <focus_group> [channel_id] [workspace]
generate_channel_digest() {
  local focus_group="$1"
  local channel_id="$2"
  local workspace="${3:-.}"
  
  # Infer FG from channel if not provided
  if [ -z "$focus_group" ] && [ -n "$channel_id" ]; then
    focus_group=$(get_channel_focus_group "$channel_id" "$workspace")
  fi
  
  if [ -z "$focus_group" ]; then
    echo "❌ Could not determine focus group for digest"
    return 1
  fi
  
  local today=$(date +%Y-%m-%d)
  
  # Collect all concepts with impacts for this FG
  local concepts_data='[]'
  
  for fg_dir in "$workspace/.planning/focus-groups"/*/; do
    [ -d "$fg_dir" ] || continue
    
    local concepts_dir="$fg_dir/concepts"
    [ -d "$concepts_dir" ] || continue
    
    for concept_dir in "$concepts_dir"/*/; do
      [ -d "$concept_dir" ] || continue
      
      local impact_file="$concept_dir/impact-matrix.md"
      [ -f "$impact_file" ] || continue
      
      local concept_name=$(basename "$concept_dir")
      local impacts=$(impact_parse_file "$impact_file" 2>/dev/null)
      [ -z "$impacts" ] && continue
      
      # Get this FG's impact
      local fg_impact=$(echo "$impacts" | jq --arg fg "$focus_group" \
        '.[] | select(.focus_group == $fg)')
      
      if [ -n "$fg_impact" ] && [ "$fg_impact" != "null" ]; then
        # Add concept name to the impact data
        local enriched=$(echo "$fg_impact" | jq --arg name "$concept_name" '. + {concept: $name}')
        concepts_data=$(echo "$concepts_data" | jq --argjson item "$enriched" '. + [$item]')
      fi
    done
  done
  
  # Generate digest message
  local digest=$(template_channel_approval_summary "$workspace" "$focus_group" "$concepts_data")
  
  # If channel_id provided, post it
  if [ -n "$channel_id" ]; then
    local token=$(slack_get_token)
    if [ $? -eq 0 ]; then
      local text=$(echo "$digest" | jq -r '.text')
      local blocks=$(echo "$digest" | jq -c '.blocks')
      
      curl -s -X POST "https://slack.com/api/chat.postMessage" \
        -H "Authorization: Bearer $token" \
        -H "Content-Type: application/json" \
        -d "$(jq -n \
          --arg channel "$channel_id" \
          --arg text "$text" \
          --argjson blocks "$blocks" \
          '{channel: $channel, text: $text, blocks: $blocks}')" >/dev/null
      
      echo "📋 Digest posted to channel"
      return 0
    fi
  fi
  
  # Otherwise return the digest content
  echo "$digest" | jq -r '.blocks[].text.text // empty' | while read -r line; do
    echo "$line"
  done
}
```

---

## Rich Status Message

### generate_status_blocks

Generate Slack Block Kit JSON for status display.

```bash
# Usage: generate_status_blocks <concept_path>
generate_status_blocks() {
  local concept_path="$1"
  
  # This calls the template function from approval-templates.md
  template_approval_status "$concept_path"
}
```

---

## SLA Status

### show_sla_status

Show SLA status for pending approvals.

```bash
# Usage: show_sla_status <concept_name> [workspace]
show_sla_status() {
  local concept_name="$1"
  local workspace="${2:-.}"
  
  local concept_path=$(find_concept_path "$concept_name" "$workspace")
  
  if [ -z "$concept_path" ]; then
    echo "❌ Concept not found: ${concept_name}"
    return 1
  fi
  
  local sla_status=$(approval_matrix_check_sla "$concept_path/impact-matrix.md")
  
  echo "⏰ *SLA Status: ${concept_name}*"
  echo ""
  
  # Overdue
  local overdue=$(echo "$sla_status" | jq -r '.overdue[]')
  if [ -n "$overdue" ]; then
    echo "🚨 *OVERDUE:*"
    echo "$sla_status" | jq -r '.overdue[] | "   🔴 \(.focus_group) - deadline passed by \(.overdue_by)"'
    echo ""
  fi
  
  # Approaching deadline
  local approaching=$(echo "$sla_status" | jq -r '.approaching[]')
  if [ -n "$approaching" ]; then
    echo "⚠️ *APPROACHING DEADLINE:*"
    echo "$sla_status" | jq -r '.approaching[] | "   🟠 \(.focus_group) - \(.remaining_hours)h remaining"'
    echo ""
  fi
  
  # OK
  local ok=$(echo "$sla_status" | jq -r '.ok[]')
  if [ -n "$ok" ]; then
    echo "✅ *On Track:*"
    echo "$sla_status" | jq -r '.ok[]' | while read fg; do
      echo "   🟢 $fg"
    done
    echo ""
  fi
  
  # Approved
  local approved=$(echo "$sla_status" | jq -r '.approved[]')
  if [ -n "$approved" ]; then
    echo "✅ *Approved:*"
    echo "$sla_status" | jq -r '.approved[]' | while read fg; do
      echo "   ✅ $fg"
    done
  fi
}
```

---

## Command Aliases

```bash
# All these invoke the same status workflow
/wgsd status oauth-integration
/wgsd approval-status oauth-integration
/wgsd show-approvals oauth-integration
/wgsd matrix oauth-integration

# Channel status
/wgsd status
/wgsd pending

# Digest
/wgsd status --digest
/wgsd daily-digest
```

---

## Usage Examples

```bash
# Show status for specific concept
/wgsd status oauth-integration

# Show all pending in current FG channel
/wgsd status

# Show pending for specific FG
/wgsd status --fg security

# Generate daily digest
/wgsd status --digest

# Show SLA status
/wgsd status oauth-integration --sla
```

---

*Workflow created for WGSD Phase 14 - Slack-Native Approval Workflow*
