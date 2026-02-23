---
name: wgsd:workflow:conversational-reject
description: Rejection workflow with required feedback and re-review support
triggers:
  - command: "/wgsd reject {concept} {reason}"
  - command: "/wgsd reject {concept} --reason {reason}"
  - command: "/wgsd request-review {concept} --fg {focus_group}"
---

# Conversational Reject Workflow

Enable constructive rejection with required feedback and easy re-review.

---

## Overview

Rejection is a valuable part of the review process, not a dead end:

1. **Required feedback** - Can't reject without explaining why
2. **Author notification** - Concept author learns of rejection immediately
3. **Discussion linkage** - Rejection posted in thread for context
4. **Re-review workflow** - Easy path to address feedback and re-request
5. **History tracking** - Rejection history preserved for learning

---

## Entry Points

### Rejection

```bash
# Reject with inline reason (required)
/wgsd reject oauth-integration "Token refresh needs rate limiting"

# Reject with --reason flag
/wgsd reject oauth-integration --reason "Missing error handling for expired tokens"

# Reject for specific FG (from any channel)
/wgsd reject oauth-integration --fg security --reason "Security review incomplete"
```

### Re-Review Request

```bash
# Request re-review after addressing feedback
/wgsd request-review oauth-integration --fg security

# With description of changes made
/wgsd request-review oauth-integration --fg security --changes "Added rate limiting and token refresh error handling"
```

---

## Workflow Steps

```yaml
workflow:
  name: conversational-reject
  version: "1.0"
  
  inputs:
    concept_name: string          # Required: Concept slug
    reason: string                # Required: Rejection reason
    focus_group: string | null    # Optional: Explicit FG override
    channel_id: string | null     # Optional: Current channel
    user_id: string               # Required: Rejecting user
  
  steps:
    - id: validate_reason
      action: require_feedback
      description: Ensure rejection has meaningful feedback
      
    - id: resolve_context
      action: infer_approval_context
      description: Determine FG from channel or explicit param
      
    - id: validate_user
      action: check_user_authority
      description: Verify user can reject for this FG
      
    - id: find_concept
      action: locate_concept_path
      description: Find concept directory
      
    - id: record_rejection
      action: execute_rejection
      description: Update matrix with rejection and feedback
      
    - id: notify_author
      action: send_author_notification
      description: DM and channel post to concept author
      
    - id: update_channel
      action: post_rejection_notice
      description: Post rejection to FG channel
      
    - id: update_concept_status
      action: mark_needs_revision
      description: Return concept to needs-revision state
      
    - id: track_history
      action: log_rejection
      description: Record in rejection history
```

---

## Core Rejection Handler

### handle_conversational_reject

Main handler for rejection commands.

```bash
# Usage: handle_conversational_reject <concept> <reason> [--fg fg] [--user user] [--channel channel]
handle_conversational_reject() {
  local concept_name=""
  local reason=""
  local focus_group=""
  local user_id=""
  local channel_id=""
  local workspace="."
  
  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --fg|--focus-group)
        focus_group="$2"
        shift 2
        ;;
      --reason|-r)
        reason="$2"
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
      --workspace|-w)
        workspace="$2"
        shift 2
        ;;
      *)
        if [ -z "$concept_name" ]; then
          concept_name="$1"
        elif [ -z "$reason" ]; then
          # Remaining args are the reason
          reason="$*"
          break
        fi
        shift
        ;;
    esac
  done
  
  # Validate required inputs
  if [ -z "$concept_name" ]; then
    echo "❌ Usage: /wgsd reject <concept> \"reason for rejection\""
    return 1
  fi
  
  if [ -z "$reason" ]; then
    echo "❌ *Rejection feedback is required*"
    echo ""
    echo "Rejections help improve concepts. Please provide specific feedback:"
    echo ""
    echo "\`/wgsd reject ${concept_name} \"Your feedback here\"\`"
    echo ""
    echo "💡 Good feedback includes:"
    echo "  • What specifically needs to change"
    echo "  • Why the current approach is problematic"
    echo "  • Suggestions for improvement"
    return 1
  fi
  
  if [ -z "$user_id" ]; then
    echo "❌ User context required"
    return 1
  fi
  
  # Validate feedback quality (minimum length)
  local reason_length=$(echo -n "$reason" | wc -c)
  if [ "$reason_length" -lt 10 ]; then
    echo "❌ Please provide more detailed feedback (minimum 10 characters)"
    echo ""
    echo "Current: \"${reason}\""
    return 1
  fi
  
  echo "🔄 Processing rejection..."
  
  # Step 1: Infer focus group context
  local inferred=$(infer_approval_context "$channel_id" "$user_id" "$focus_group" "$workspace")
  
  if echo "$inferred" | grep -q "^ERROR"; then
    echo "$inferred"
    return 1
  fi
  
  focus_group=$(echo "$inferred" | tail -1)
  
  # Step 2: Validate user authority
  if [ "$(approval_matrix_can_approve "$user_id" "$focus_group")" != "true" ]; then
    echo "❌ You don't have authority to reject for *${focus_group}*"
    echo ""
    echo "💡 You can reject for: $(approval_matrix_get_user_fgs "$user_id" "$workspace" | xargs | tr ' ' ', ')"
    return 1
  fi
  
  # Step 3: Find concept
  local concept_path=$(find_concept_path "$concept_name" "$workspace")
  
  if [ -z "$concept_path" ]; then
    echo "❌ Concept not found: *${concept_name}*"
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
  
  if [ "$current_status" = "rejected" ]; then
    echo "⚠️  Already rejected! *${focus_group}* previously rejected this concept."
    echo ""
    echo "Current feedback is recorded. To update feedback, use:"
    echo "\`/wgsd reject ${concept_name} --update \"New feedback\"\`"
    return 0
  fi
  
  # Step 5: Execute rejection
  local today=$(date +%Y-%m-%d)
  
  impact_update "$impact_file" "$focus_group" "status" "rejected"
  impact_update "$impact_file" "$focus_group" "approver" "@$user_id"
  impact_update "$impact_file" "$focus_group" "approved_date" "$today"
  impact_update "$impact_file" "$focus_group" "approval_comment" "$reason"
  
  # Step 6: Log rejection to history
  log_rejection_history "$impact_file" "$focus_group" "$user_id" "$reason" "$today"
  
  # Step 7: Update concept status
  update_concept_status_needs_revision "$concept_path" "$focus_group" "$reason"
  
  # Step 8: Output confirmation
  echo "🚫 *Rejected*"
  echo ""
  echo "📋 *${concept_name}* rejected by @${user_id} for *${focus_group}*"
  echo ""
  echo "📝 *Feedback:*"
  echo "> ${reason}"
  echo ""
  
  # Step 9: Get author and notify
  local author=$(get_concept_author "$concept_path")
  
  if [ -n "$author" ]; then
    echo "📨 Notifying concept author: ${author}"
    notify_author_of_rejection "$author" "$concept_name" "$focus_group" "$user_id" "$reason" "$channel_id"
  fi
  
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "*Next Steps for Author:*"
  echo "1. Review the feedback above"
  echo "2. Update the concept to address concerns"
  echo "3. Request re-review: \`/wgsd request-review ${concept_name} --fg ${focus_group}\`"
  
  # Step 10: Post to FG channel
  if [ -n "$channel_id" ]; then
    post_rejection_notification "$channel_id" "$concept_name" "$focus_group" "$user_id" "$reason"
  fi
  
  return 0
}
```

---

## Rejection Helper Functions

### log_rejection_history

Record rejection in history for tracking.

```bash
# Usage: log_rejection_history <impact_file> <focus_group> <user> <reason> <date>
log_rejection_history() {
  local impact_file="$1"
  local focus_group="$2"
  local user="$3"
  local reason="$4"
  local date="$5"
  
  # Add to rejection_history array in impact-matrix.md
  python3 -c "
import re
import yaml
from datetime import datetime

with open('$impact_file', 'r') as f:
    content = f.read()

yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    print('ERROR: Could not find YAML frontmatter')
    exit(1)

yaml_content = yaml_match.group(1)
data = yaml.safe_load(yaml_content)

# Initialize rejection_history if not exists
if 'rejection_history' not in data:
    data['rejection_history'] = []

# Add new rejection entry
data['rejection_history'].append({
    'focus_group': '$focus_group',
    'rejected_by': '@$user',
    'date': '$date',
    'reason': '''$reason''',
    'timestamp': datetime.utcnow().isoformat() + 'Z'
})

# Rebuild YAML
new_yaml = yaml.dump(data, default_flow_style=False, allow_unicode=True, sort_keys=False)
new_content = content[:yaml_match.start()] + '\`\`\`yaml\n---\n' + new_yaml + '---\n\`\`\`' + content[yaml_match.end():]

with open('$impact_file', 'w') as f:
    f.write(new_content)

print('HISTORY_LOGGED')
"
  return 0
}
```

### update_concept_status_needs_revision

Update concept CONCEPT.md to reflect needs-revision state.

```bash
# Usage: update_concept_status_needs_revision <concept_path> <focus_group> <reason>
update_concept_status_needs_revision() {
  local concept_path="$1"
  local focus_group="$2"
  local reason="$3"
  
  local concept_file="$concept_path/CONCEPT.md"
  
  if [ ! -f "$concept_file" ]; then
    return 1
  fi
  
  # Update status
  sed -i 's/\*\*Status:\*\* .*/\*\*Status:\*\* 📝 Needs Revision/' "$concept_file" 2>/dev/null || true
  
  # Add revision note if there's a notes section
  local today=$(date +%Y-%m-%d)
  local note="**${today}**: Rejected by ${focus_group} - ${reason}"
  
  # Append to revision notes section if it exists, or add one
  if grep -q "## Revision Notes" "$concept_file"; then
    # Append to existing section
    sed -i "/## Revision Notes/a - ${note}" "$concept_file" 2>/dev/null || true
  fi
  
  echo "STATUS_UPDATED:needs_revision"
  return 0
}
```

### get_concept_author

Get the author of a concept from CONCEPT.md.

```bash
# Usage: get_concept_author <concept_path>
get_concept_author() {
  local concept_path="$1"
  local concept_file="$concept_path/CONCEPT.md"
  
  if [ ! -f "$concept_file" ]; then
    return 1
  fi
  
  # Try to extract author from frontmatter or metadata
  local author=$(grep -m1 "^\*\*Author:\*\*\|^author:\|^\*\*Creator:\*\*" "$concept_file" | \
    sed 's/.*@//' | sed 's/[[:space:]]*$//' | head -1)
  
  if [ -z "$author" ]; then
    # Try YAML frontmatter
    author=$(sed -n '/^---$/,/^---$/p' "$concept_file" | grep "author:" | awk '{print $2}' | tr -d '@')
  fi
  
  echo "$author"
}
```

### notify_author_of_rejection

Send rejection notification to concept author.

```bash
# Usage: notify_author_of_rejection <author> <concept> <focus_group> <rejector> <reason> [channel_id]
notify_author_of_rejection() {
  local author="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local rejector="$4"
  local reason="$5"
  local channel_id="${6:-}"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    echo "WARNING: Could not get Slack token for author notification"
    return 0
  fi
  
  # Build DM message
  local message="🚫 *Your Concept Was Rejected*

*Concept:* ${concept_slug}
*Focus Group:* ${focus_group}
*Rejected by:* @${rejector}

*📝 Feedback:*
>${reason}

*Next Steps:*
1. Review the feedback above
2. Update your concept to address concerns
3. Request re-review: \`/wgsd request-review ${concept_slug} --fg ${focus_group}\`

💬 Feel free to discuss the feedback in the focus group channel or reach out to @${rejector} directly."

  # Get author's user ID (might need to look up from username)
  local author_id=$(lookup_user_id "$author")
  
  if [ -n "$author_id" ]; then
    # Send DM
    curl -s -X POST "https://slack.com/api/chat.postMessage" \
      -H "Authorization: Bearer $token" \
      -H "Content-Type: application/json" \
      -d "$(jq -n \
        --arg channel "$author_id" \
        --arg text "$message" \
        '{channel: $channel, text: $text, mrkdwn: true}')" >/dev/null
    
    echo "AUTHOR_DM_SENT:$author"
  fi
  
  return 0
}
```

### post_rejection_notification

Post rejection notice to focus group channel.

```bash
# Usage: post_rejection_notification <channel_id> <concept> <focus_group> <rejector> <reason>
post_rejection_notification() {
  local channel_id="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local rejector="$4"
  local reason="$5"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  # Generate rejection message
  local message_json=$(template_rejection_message "$concept_slug" "$focus_group" "$rejector" "$reason")
  local text=$(echo "$message_json" | jq -r '.text')
  local blocks=$(echo "$message_json" | jq -c '.blocks')
  
  # Post to channel
  curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg text "$text" \
      --argjson blocks "$blocks" \
      '{channel: $channel, text: $text, blocks: $blocks}')" >/dev/null
  
  echo "REJECTION_POSTED:$channel_id"
  return 0
}
```

### lookup_user_id

Look up Slack user ID from username.

```bash
# Usage: lookup_user_id <username>
lookup_user_id() {
  local username="$1"
  
  # Remove @ prefix if present
  username=$(echo "$username" | sed 's/^@//')
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  # Use users.list to find user
  local response=$(curl -s "https://slack.com/api/users.list?limit=1000" \
    -H "Authorization: Bearer $token")
  
  local user_id=$(echo "$response" | jq -r --arg name "$username" \
    '.members[] | select(.name == $name or .profile.display_name == $name) | .id' | head -1)
  
  echo "$user_id"
}
```

---

## Re-Review Workflow

### handle_request_review

Handle re-review requests after addressing rejection feedback.

```bash
# Usage: handle_request_review <concept> --fg <focus_group> [--changes "description"]
handle_request_review() {
  local concept_name=""
  local focus_group=""
  local changes=""
  local user_id=""
  local channel_id=""
  local workspace="."
  
  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --fg|--focus-group)
        focus_group="$2"
        shift 2
        ;;
      --changes|-c)
        changes="$2"
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
    echo "❌ Usage: /wgsd request-review <concept> --fg <focus_group>"
    return 1
  fi
  
  if [ -z "$focus_group" ]; then
    echo "❌ Focus group is required for re-review request"
    echo ""
    echo "Usage: \`/wgsd request-review ${concept_name} --fg <focus_group>\`"
    return 1
  fi
  
  # Find concept
  local concept_path=$(find_concept_path "$concept_name" "$workspace")
  
  if [ -z "$concept_path" ]; then
    echo "❌ Concept not found: *${concept_name}*"
    return 1
  fi
  
  local impact_file="$concept_path/impact-matrix.md"
  
  # Check current status
  local impacts=$(impact_parse_file "$impact_file")
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ "$current_status" != "rejected" ]; then
    echo "⚠️  *${focus_group}* hasn't rejected this concept"
    echo ""
    if [ "$current_status" = "approved" ]; then
      echo "✅ Already approved!"
    elif [ "$current_status" = "pending" ]; then
      echo "⏳ Still pending review"
    fi
    return 0
  fi
  
  # Reset status to pending
  local today=$(date +%Y-%m-%d)
  
  impact_update "$impact_file" "$focus_group" "status" "pending"
  impact_update "$impact_file" "$focus_group" "approver" "null"
  impact_update "$impact_file" "$focus_group" "approved_date" "null"
  # Keep the rejection comment for history
  # impact_update "$impact_file" "$focus_group" "approval_comment" "null"
  
  # Reset SLA
  approval_matrix_set_sla "$impact_file" "$focus_group"
  
  # Update concept status
  local concept_file="$concept_path/CONCEPT.md"
  if [ -f "$concept_file" ]; then
    sed -i 's/\*\*Status:\*\* .*/\*\*Status:\*\* 🔄 Under Review/' "$concept_file" 2>/dev/null || true
  fi
  
  echo "🔄 *Re-Review Requested*"
  echo ""
  echo "📋 *${concept_name}* re-review requested for *${focus_group}*"
  
  if [ -n "$changes" ]; then
    echo ""
    echo "📝 *Changes Made:*"
    echo "> ${changes}"
  fi
  
  echo ""
  echo "📨 Notifying the ${focus_group} focus group..."
  
  # Get FG channel and send notification
  local fg_channel=$(impact_get_channel_for_fg "$workspace" "$focus_group")
  
  if [ -n "$fg_channel" ]; then
    local requester="${user_id:-the author}"
    send_re_review_notification "$fg_channel" "$concept_name" "$focus_group" "$requester" "$changes"
    echo "✅ Notification sent to #${focus_group} channel"
  fi
  
  return 0
}
```

### send_re_review_notification

Send re-review request to focus group channel.

```bash
# Usage: send_re_review_notification <channel_id> <concept> <focus_group> <requester> [changes]
send_re_review_notification() {
  local channel_id="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local requester="$4"
  local changes="${5:-Concept updated to address feedback}"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  # Generate re-review request message
  local message_json=$(template_re_review_request "$concept_slug" "$focus_group" "$requester" "$changes")
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
    local ts=$(echo "$response" | jq -r '.ts')
    # Track this as a new approval prompt
    track_approval_prompt "$channel_id" "$ts" "$concept_slug" "$focus_group"
    echo "RE_REVIEW_SENT:$ts"
    return 0
  else
    echo "WARNING: Could not send re-review notification"
    return 1
  fi
}
```

---

## Rejection History

### show_rejection_history

Display rejection history for a concept.

```bash
# Usage: show_rejection_history <concept_name> [workspace]
show_rejection_history() {
  local concept_name="$1"
  local workspace="${2:-.}"
  
  local concept_path=$(find_concept_path "$concept_name" "$workspace")
  
  if [ -z "$concept_path" ]; then
    echo "❌ Concept not found: ${concept_name}"
    return 1
  fi
  
  local impact_file="$concept_path/impact-matrix.md"
  
  # Extract rejection history
  local history=$(python3 -c "
import re
import yaml

with open('$impact_file', 'r') as f:
    content = f.read()

yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    exit(1)

data = yaml.safe_load(yaml_match.group(1))
history = data.get('rejection_history', [])

if not history:
    print('NO_HISTORY')
else:
    for i, entry in enumerate(history, 1):
        print(f'{i}. {entry.get(\"date\", \"?\")} - {entry.get(\"focus_group\", \"?\")}')
        print(f'   Rejected by: {entry.get(\"rejected_by\", \"?\")}')
        print(f'   Reason: {entry.get(\"reason\", \"No reason\")}')
        print()
")
  
  echo "📜 *Rejection History: ${concept_name}*"
  echo ""
  
  if [ "$history" = "NO_HISTORY" ]; then
    echo "✅ No previous rejections"
  else
    echo "$history"
  fi
}
```

---

## Command Routing

```yaml
# Rejection commands
/wgsd reject <concept> "reason" → handle_conversational_reject
/wgsd reject <concept> --reason "reason" → handle_conversational_reject
/wgsd reject <concept> --fg security --reason "reason" → handle_conversational_reject

# Re-review commands
/wgsd request-review <concept> --fg security → handle_request_review
/wgsd re-review <concept> --fg security → handle_request_review

# History
/wgsd rejection-history <concept> → show_rejection_history
```

---

## Usage Examples

```bash
# Reject with feedback (from FG channel)
/wgsd reject oauth-integration "Token refresh needs rate limiting to prevent abuse"

# Reject for specific FG (from any channel)
/wgsd reject oauth-integration --fg security --reason "Missing validation for refresh tokens"

# Request re-review after fixing
/wgsd request-review oauth-integration --fg security --changes "Added rate limiting and token validation"

# View rejection history
/wgsd rejection-history oauth-integration
```

---

## Error Handling

| Situation | Message |
|-----------|---------|
| No reason provided | "Rejection feedback is required. Please provide specific feedback." |
| Reason too short | "Please provide more detailed feedback (minimum 10 characters)" |
| No authority | "You don't have authority to reject for {fg}" |
| Already rejected | "Already rejected! Current feedback is recorded." |
| No impact declared | "{fg} has no declared impact on this concept" |

---

*Workflow created for WGSD Phase 14 - Slack-Native Approval Workflow*
