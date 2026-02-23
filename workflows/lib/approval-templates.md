---
name: wgsd:lib:approval-templates
description: Rich Slack message templates for conversational approval workflow
---

# Approval Templates Library

Rich Slack Block Kit templates for approval prompts, status messages, and discussion threads.

---

## Overview

Approval templates enable conversational Slack-native approval workflows:
- Rich approval prompts with context and actions
- Status summaries with visual progress indicators
- Discussion thread headers and context
- Rejection feedback formatting

---

## Template Functions

### template_approval_prompt

Generate a rich approval prompt message for a focus group channel.

```bash
# Usage: template_approval_prompt <workspace> <concept_slug> <focus_group> <impact_json>
# Returns: Slack Block Kit JSON for approval prompt
template_approval_prompt() {
  local workspace="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local impact_json="$4"
  
  # Parse impact details
  local priority=$(echo "$impact_json" | jq -r '.priority // "P2"')
  local type=$(echo "$impact_json" | jq -r '.type // "integration"')
  local description=$(echo "$impact_json" | jq -r '.description // "No description"')
  
  # Get matrix state for context
  local concept_path=$(find_concept_path "$concept_slug" "$workspace")
  local matrix_state=""
  if [ -n "$concept_path" ]; then
    matrix_state=$(approval_matrix_get "$concept_path/impact-matrix.md" 2>/dev/null || echo '{}')
  fi
  
  local approved=$(echo "$matrix_state" | jq -r '.approved // 0')
  local total=$(echo "$matrix_state" | jq -r '.total_approvals // 1')
  local completion=$(echo "$matrix_state" | jq -r '.completion_percent // 0')
  
  # Priority badge and color
  local priority_badge=""
  local priority_color=""
  case "$priority" in
    P0) priority_badge="🔴 P0 - Critical"; priority_color="#dc3545" ;;
    P1) priority_badge="🟠 P1 - High"; priority_color="#fd7e14" ;;
    P2) priority_badge="🟡 P2 - Medium"; priority_color="#ffc107" ;;
    P3) priority_badge="🟢 P3 - Low"; priority_color="#28a745" ;;
  esac
  
  # Type label
  local type_label=""
  case "$type" in
    primary-owner) type_label="Primary Owner 👑" ;;
    breaking-change) type_label="⚠️ Breaking Change" ;;
    api-change) type_label="API Change 🔌" ;;
    documentation) type_label="Documentation 📚" ;;
    integration) type_label="Integration 🔗" ;;
    testing) type_label="Testing 🧪" ;;
    behavior) type_label="Behavior 🔄" ;;
    security) type_label="Security 🔒" ;;
    performance) type_label="Performance ⚡" ;;
    *) type_label="$type" ;;
  esac
  
  # Progress bar (visual)
  local progress_bar=$(generate_progress_bar "$completion")
  
  # Generate Block Kit JSON
  cat <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "📋 Approval Request: ${concept_slug}",
        "emoji": true
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Your review is requested for this concept*\n\n${description}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Priority:*\n${priority_badge}"
        },
        {
          "type": "mrkdwn",
          "text": "*Impact Type:*\n${type_label}"
        }
      ]
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Approval Progress:*\n${approved}/${total} (${completion}%)"
        },
        {
          "type": "mrkdwn",
          "text": "*Your Focus Group:*\n${focus_group}"
        }
      ]
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "${progress_bar}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📌 Quick Actions:*"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "✅ \`/wgsd approve ${concept_slug}\` — Approve this concept\n🚫 \`/wgsd reject ${concept_slug} \"reason\"\` — Reject with feedback\n⏸️ \`/wgsd block ${concept_slug} --until <fg>\` — Block pending dependency\n💬 Reply in thread to discuss"
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "💡 *Tip:* React with ✅ to quick-approve or reply \"LGTM\" in thread"
        }
      ]
    }
  ],
  "text": "📋 Approval Request: ${concept_slug} — Your review is requested (${priority})"
}
EOF
}
```

---

### generate_progress_bar

Create a visual progress bar using emoji.

```bash
# Usage: generate_progress_bar <percentage>
# Returns: Visual progress bar string
generate_progress_bar() {
  local percent="${1:-0}"
  local filled=$(( percent / 10 ))
  local empty=$(( 10 - filled ))
  
  local bar=""
  for ((i=0; i<filled; i++)); do
    bar="${bar}🟩"
  done
  for ((i=0; i<empty; i++)); do
    bar="${bar}⬜"
  done
  
  echo "${bar} ${percent}%"
}
```

---

### template_approval_status

Generate approval status summary for a concept.

```bash
# Usage: template_approval_status <concept_path>
# Returns: Slack Block Kit JSON for status summary
template_approval_status() {
  local concept_path="$1"
  local concept_slug=$(basename "$concept_path")
  
  local matrix=$(approval_matrix_get "$concept_path/impact-matrix.md" 2>/dev/null)
  if [ -z "$matrix" ] || echo "$matrix" | grep -q "error"; then
    echo '{"error": "Could not load approval matrix"}'
    return 1
  fi
  
  local total=$(echo "$matrix" | jq -r '.total_approvals')
  local approved=$(echo "$matrix" | jq -r '.approved')
  local pending=$(echo "$matrix" | jq -r '.pending')
  local rejected=$(echo "$matrix" | jq -r '.rejected')
  local blocked=$(echo "$matrix" | jq -r '.blocked')
  local completion=$(echo "$matrix" | jq -r '.completion_percent')
  local fully_approved=$(echo "$matrix" | jq -r '.fully_approved')
  
  # Status emoji and text
  local status_emoji="⏳"
  local status_text="In Review"
  if [ "$fully_approved" = "true" ]; then
    status_emoji="✅"
    status_text="Fully Approved"
  elif [ "$rejected" -gt 0 ]; then
    status_emoji="🚫"
    status_text="Needs Revision"
  elif [ "$blocked" -gt 0 ]; then
    status_emoji="⏸️"
    status_text="Blocked"
  fi
  
  # Build per-FG status
  local fg_status=""
  local details=$(echo "$matrix" | jq -c '.details[]')
  while IFS= read -r detail; do
    local fg=$(echo "$detail" | jq -r '.focus_group')
    local fg_status_val=$(echo "$detail" | jq -r '.status')
    local approver=$(echo "$detail" | jq -r '.approver // ""')
    
    local icon=""
    case "$fg_status_val" in
      approved) icon="✅" ;;
      pending) icon="⏳" ;;
      rejected) icon="🚫" ;;
      blocked) icon="⏸️" ;;
    esac
    
    if [ -n "$approver" ] && [ "$approver" != "null" ]; then
      fg_status="${fg_status}\n${icon} *${fg}*: ${fg_status_val} (${approver})"
    else
      fg_status="${fg_status}\n${icon} *${fg}*: ${fg_status_val}"
    fi
  done <<< "$details"
  
  local progress_bar=$(generate_progress_bar "$completion")
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "${status_emoji} Approval Status: ${concept_slug}",
        "emoji": true
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Overall Status:* ${status_text}\n*Progress:* ${approved}/${total} approvals (${completion}%)\n\n${progress_bar}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Focus Group Status:*${fg_status}"
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "📊 Approved: ${approved} | ⏳ Pending: ${pending} | 🚫 Rejected: ${rejected} | ⏸️ Blocked: ${blocked}"
        }
      ]
    }
  ],
  "text": "${status_emoji} Approval Status for ${concept_slug}: ${status_text} (${completion}%)"
}
EOF
}
```

---

### template_discussion_header

Generate discussion thread header with context.

```bash
# Usage: template_discussion_header <concept_slug> <focus_group> <author>
# Returns: Slack message for thread header
template_discussion_header() {
  local concept_slug="$1"
  local focus_group="$2"
  local author="${3:-unknown}"
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "💬 *Discussion Thread: ${concept_slug}*\n\nUse this thread to discuss concerns, ask questions, or provide feedback before approving.\n\n*Concept Author:* ${author}\n*Focus Group:* ${focus_group}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "💡 The concept author has been notified of this discussion. Reply here to continue the conversation."
        }
      ]
    }
  ],
  "text": "💬 Discussion thread for ${concept_slug} - ${focus_group} review"
}
EOF
}
```

---

### template_approval_confirmed

Generate confirmation message after approval.

```bash
# Usage: template_approval_confirmed <concept_slug> <focus_group> <approver> <comment>
# Returns: Slack message for confirmation
template_approval_confirmed() {
  local concept_slug="$1"
  local focus_group="$2"
  local approver="$3"
  local comment="${4:-}"
  
  local comment_section=""
  if [ -n "$comment" ]; then
    comment_section="
    {
      \"type\": \"section\",
      \"text\": {
        \"type\": \"mrkdwn\",
        \"text\": \"*Comment:* ${comment}\"
      }
    },"
  fi
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "✅ *Approved by ${approver}*\n\nThe *${focus_group}* focus group has approved *${concept_slug}*."
      }
    },${comment_section}
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "📋 Use \`/wgsd status ${concept_slug}\` to check full approval progress"
        }
      ]
    }
  ],
  "text": "✅ ${approver} approved ${concept_slug} for ${focus_group}"
}
EOF
}
```

---

### template_rejection_message

Generate rejection notification with feedback.

```bash
# Usage: template_rejection_message <concept_slug> <focus_group> <rejector> <reason>
# Returns: Slack Block Kit JSON for rejection
template_rejection_message() {
  local concept_slug="$1"
  local focus_group="$2"
  local rejector="$3"
  local reason="$4"
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🚫 Concept Rejected: ${concept_slug}",
        "emoji": true
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "The *${focus_group}* focus group has rejected this concept.\n\n*Rejected by:* ${rejector}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📝 Feedback:*\n>${reason}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📋 Next Steps:*\n1. Review the feedback above\n2. Update the concept to address concerns\n3. Request re-review: \`/wgsd request-review ${concept_slug} --fg ${focus_group}\`"
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "💬 Reply in thread to discuss the feedback or ask clarifying questions"
        }
      ]
    }
  ],
  "text": "🚫 ${concept_slug} rejected by ${rejector} (${focus_group}): ${reason}"
}
EOF
}
```

---

### template_re_review_request

Generate re-review request notification.

```bash
# Usage: template_re_review_request <concept_slug> <focus_group> <requester> <changes>
# Returns: Slack message for re-review request
template_re_review_request() {
  local concept_slug="$1"
  local focus_group="$2"
  local requester="$3"
  local changes="${4:-Concept updated to address feedback}"
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🔄 Re-Review Requested: ${concept_slug}",
        "emoji": true
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "${requester} has requested that *${focus_group}* re-review this concept after addressing previous feedback."
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📝 Changes Made:*\n${changes}"
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📌 Actions:*\n✅ \`/wgsd approve ${concept_slug}\` — Approve the updated concept\n🚫 \`/wgsd reject ${concept_slug} \"reason\"\` — Reject again with new feedback\n💬 Reply in thread to continue discussion"
      }
    }
  ],
  "text": "🔄 Re-review requested for ${concept_slug} by ${requester}"
}
EOF
}
```

---

### template_full_approval_celebration

Generate celebration message when all approvals complete.

```bash
# Usage: template_full_approval_celebration <concept_slug> <approver_count>
# Returns: Slack message for full approval
template_full_approval_celebration() {
  local concept_slug="$1"
  local approver_count="${2:-1}"
  
  cat <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🎉 Concept Fully Approved!",
        "emoji": true
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*${concept_slug}* has received approval from all ${approver_count} required focus groups!\n\n✨ This concept is now ready for the roadmap."
      }
    },
    {
      "type": "divider"
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*📋 Next Steps:*\n• Concept will auto-merge to \`roadmap\` branch\n• Create implementation: \`/wgsd create-implementation ${concept_slug}\`\n• Or add to sprint planning queue"
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "🚀 Congratulations team! The approval process is complete."
        }
      ]
    }
  ],
  "text": "🎉 ${concept_slug} is fully approved and ready for implementation!"
}
EOF
}
```

---

### template_channel_approval_summary

Generate channel-specific approval summary (daily digest).

```bash
# Usage: template_channel_approval_summary <workspace> <focus_group> <concepts_json>
# Returns: Slack message for channel summary
template_channel_approval_summary() {
  local workspace="$1"
  local focus_group="$2"
  local concepts_json="$3"
  
  python3 -c "
import json
from datetime import date

concepts = json.loads('''$concepts_json''')
fg = '$focus_group'
today = date.today().isoformat()

pending = [c for c in concepts if c.get('status') == 'pending']
recently_approved = [c for c in concepts if c.get('status') == 'approved' and c.get('approved_date', '').startswith(today[:7])]
blocked = [c for c in concepts if c.get('status') == 'blocked']

# Build summary sections
pending_text = ''
if pending:
    pending_text = '*⏳ Pending Your Review ({0}):*\n'.format(len(pending))
    for c in pending[:5]:  # Limit to 5
        pending_text += '• \`{0}\` ({1})\n'.format(c.get('concept', '?'), c.get('priority', '?'))
    if len(pending) > 5:
        pending_text += '• _...and {0} more_\n'.format(len(pending) - 5)

approved_text = ''
if recently_approved:
    approved_text = '\n*✅ Recently Approved ({0}):*\n'.format(len(recently_approved))
    for c in recently_approved[:3]:
        approved_text += '• \`{0}\` by {1}\n'.format(c.get('concept', '?'), c.get('approver', '?'))

blocked_text = ''
if blocked:
    blocked_text = '\n*⏸️ Blocked ({0}):*\n'.format(len(blocked))
    for c in blocked[:3]:
        blocked_text += '• \`{0}\` (waiting on {1})\n'.format(c.get('concept', '?'), c.get('blocked_by', '?'))

summary = {
    'blocks': [
        {
            'type': 'header',
            'text': {
                'type': 'plain_text',
                'text': '📊 Approval Summary for ' + fg,
                'emoji': True
            }
        },
        {
            'type': 'section',
            'text': {
                'type': 'mrkdwn',
                'text': pending_text + approved_text + blocked_text if (pending_text or approved_text or blocked_text) else '✨ No pending approvals!'
            }
        },
        {
            'type': 'context',
            'elements': [
                {
                    'type': 'mrkdwn',
                    'text': '💡 Use \`/wgsd status <concept>\` for detailed status | Generated ' + today
                }
            ]
        }
    ],
    'text': 'Approval summary for ' + fg + ': ' + str(len(pending)) + ' pending, ' + str(len(recently_approved)) + ' approved this month'
}

print(json.dumps(summary, indent=2))
"
}
```

---

## Thread Management

### create_discussion_thread

Create a discussion thread from an approval prompt.

```bash
# Usage: create_discussion_thread <channel_id> <prompt_message_ts> <concept_slug> <focus_group> <author>
# Returns: Thread ts or error
create_discussion_thread() {
  local channel_id="$1"
  local prompt_ts="$2"
  local concept_slug="$3"
  local focus_group="$4"
  local author="${5:-the author}"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    echo "ERROR: Could not get Slack token"
    return 1
  fi
  
  # Generate thread header
  local thread_content=$(template_discussion_header "$concept_slug" "$focus_group" "$author")
  local text=$(echo "$thread_content" | jq -r '.text')
  local blocks=$(echo "$thread_content" | jq -c '.blocks')
  
  # Post as thread reply to approval prompt
  local response=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg text "$text" \
      --arg thread_ts "$prompt_ts" \
      --argjson blocks "$blocks" \
      '{channel: $channel, text: $text, thread_ts: $thread_ts, blocks: $blocks}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    local thread_ts=$(echo "$response" | jq -r '.ts')
    echo "THREAD_CREATED:$thread_ts"
    echo "CHANNEL:$channel_id"
    return 0
  else
    local error=$(echo "$response" | jq -r '.error')
    echo "ERROR: Failed to create discussion thread: $error"
    return 1
  fi
}
```

### notify_author_of_discussion

Notify concept author that a discussion started.

```bash
# Usage: notify_author_of_discussion <author_user_id> <concept_slug> <focus_group> <channel_id> <thread_ts>
# Returns: Success or error
notify_author_of_discussion() {
  local author_id="$1"
  local concept_slug="$2"
  local focus_group="$3"
  local channel_id="$4"
  local thread_ts="$5"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  local thread_link="<slack://channel?team=T123&id=${channel_id}&message=${thread_ts}|View Discussion>"
  
  local message="💬 *Discussion Started*\n\nThe *${focus_group}* focus group has started a discussion about your concept *${concept_slug}*.\n\n${thread_link}\n\n_Join the thread to address questions or concerns._"
  
  # Send DM to author
  local response=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$author_id" \
      --arg text "$message" \
      '{channel: $channel, text: $text, mrkdwn: true}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    echo "AUTHOR_NOTIFIED:$author_id"
    return 0
  else
    echo "WARNING: Could not notify author: $(echo "$response" | jq -r '.error')"
    return 0  # Non-fatal
  fi
}
```

---

## Message Update Functions

### update_approval_prompt

Update the original approval prompt with new status.

```bash
# Usage: update_approval_prompt <channel_id> <message_ts> <new_status> <concept_slug>
# Returns: Success or error
update_approval_prompt() {
  local channel_id="$1"
  local message_ts="$2"
  local new_status="$3"
  local concept_slug="$4"
  
  local token=$(slack_get_token)
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  # Generate updated blocks based on status
  local status_text=""
  local status_color=""
  
  case "$new_status" in
    approved)
      status_text="✅ *APPROVED* - This focus group has approved the concept"
      ;;
    rejected)
      status_text="🚫 *REJECTED* - See thread for feedback"
      ;;
    blocked)
      status_text="⏸️ *BLOCKED* - Waiting on dependency"
      ;;
  esac
  
  # Add status banner to existing message
  # In a real implementation, we'd reconstruct the full blocks
  # For now, we'll use a simpler approach with attachments
  
  local response=$(curl -s -X POST "https://slack.com/api/chat.update" \
    -H "Authorization: Bearer $token" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg channel "$channel_id" \
      --arg ts "$message_ts" \
      --arg text "📋 Approval Request: ${concept_slug} - ${new_status}" \
      '{channel: $channel, ts: $ts, text: $text}')")
  
  local ok=$(echo "$response" | jq -r '.ok')
  
  if [ "$ok" = "true" ]; then
    echo "PROMPT_UPDATED:$message_ts"
    return 0
  else
    echo "WARNING: Could not update prompt: $(echo "$response" | jq -r '.error')"
    return 0  # Non-fatal
  fi
}
```

---

## Usage Examples

```bash
# Generate approval prompt for a focus group
prompt=$(template_approval_prompt "/path/to/workspace" "oauth-integration" "security" \
  '{"priority": "P1", "type": "integration", "description": "Token validation changes"}')

# Generate status summary
status=$(template_approval_status "/path/to/concepts/oauth-integration")

# Generate rejection message
rejection=$(template_rejection_message "oauth-integration" "security" "@alice" \
  "The token refresh logic needs rate limiting to prevent abuse")

# Create discussion thread
create_discussion_thread "C0123456789" "1234567890.123456" "oauth-integration" "security" "@bob"

# Generate celebration message
celebration=$(template_full_approval_celebration "oauth-integration" 3)
```

---

*Library created for WGSD Phase 14 - Slack-Native Approval Workflow*
