---
name: wgsd:workflow:matrix-approve
description: Per-focus-group approval workflow for matrix-based concept approval
triggers:
  - command: "/wgsd approve {concept} --fg {focus_group}"
  - command: "/wgsd approve {concept}"
  - slack_reaction: thumbsup on concept channel
  - slack_thread_reply: "approved" in concept discussion
---

# Matrix Approve Workflow

Enable independent approval from each impacted focus group for matrix-based concept approval.

---

## Overview

The matrix approval system enables parallel approvals across multiple stakeholders:
- Each focus group independently reviews and approves their impact
- Partial approval state is valid and tracked
- Full approval requires ALL impacted focus groups to approve
- SLA tracking ensures timely reviews
- Blocked dependencies are handled transparently

---

## Entry Points

### Explicit Approval
```bash
# Approve for specific focus group
/wgsd approve oauth-integration --fg security

# Approve for your default focus group (inferred from user)
/wgsd approve oauth-integration

# Approve with comment
/wgsd approve oauth-integration --fg security --comment "Token flow reviewed, LGTM"
```

### Slack Reactions
- 👍 or ✅ on concept announcement → triggers approval for user's focus group

### Thread Reply
- "approved", "lgtm", "+1" in concept thread → triggers approval

---

## Workflow Steps

```yaml
workflow:
  name: matrix-approve
  version: "2.2"
  
  inputs:
    concept_name: string          # Required: Concept slug
    focus_group: string | null    # Optional: Target focus group (inferred if null)
    user: string                  # Required: User requesting approval
    comment: string | null        # Optional: Approval comment
  
  steps:
    - id: validate_authority
      action: approval_matrix_can_approve
      description: Verify user can approve for focus group
      
    - id: load_concept
      action: load_concept_path
      description: Find concept directory and impact-matrix.md
      
    - id: check_current_status
      action: check_impact_status
      description: Verify impact is pending/blocked (not already approved)
      
    - id: record_approval
      action: update_impact_status
      description: Update status to approved with metadata
      
    - id: clear_blocked
      action: auto_unblock_dependents
      description: Unblock any FGs that were waiting on this one
      
    - id: check_completion
      action: check_full_approval
      description: Check if all approvals are now complete
      
    - id: notify
      action: send_notifications
      description: Notify concept channel and potentially trigger next phase
      
    - id: sync_canvas
      action: update_canvas
      description: Update concept canvas with new approval state
```

---

## Authority Validation

### approval_matrix_can_approve

Check if user has authority to approve for a focus group.

```bash
# Usage: approval_matrix_can_approve <user> <focus_group>
# Returns: "true" or "false"
approval_matrix_can_approve() {
  local user="$1"
  local focus_group="$2"
  
  # Get user's focus group affiliations
  local user_fgs=$(approval_matrix_get_user_fgs "$user")
  
  if [ $? -ne 0 ]; then
    echo "false"
    return 1
  fi
  
  # Check if user belongs to the focus group
  if echo "$user_fgs" | grep -qw "$focus_group"; then
    echo "true"
    return 0
  fi
  
  # Check if user is admin (can approve any)
  if approval_matrix_is_admin "$user"; then
    echo "true"
    return 0
  fi
  
  echo "false"
  return 1
}
```

### approval_matrix_get_user_fgs

Get user's focus group affiliations.

```bash
# Usage: approval_matrix_get_user_fgs <user>
# Returns: Space-separated list of focus group slugs
approval_matrix_get_user_fgs() {
  local user="$1"
  local repo_path="${2:-.}"
  
  local fg_dir="$repo_path/.planning/focus-groups"
  local result=""
  
  if [ ! -d "$fg_dir" ]; then
    return 1
  fi
  
  # Check each focus group's MEMBERS.md or STATE.md
  for fg in "$fg_dir"/*/; do
    [ -d "$fg" ] || continue
    
    local fg_name=$(basename "$fg")
    
    # Check MEMBERS.md
    if [ -f "$fg/MEMBERS.md" ]; then
      if grep -qi "@$user" "$fg/MEMBERS.md" 2>/dev/null; then
        result="$result $fg_name"
        continue
      fi
    fi
    
    # Check STATE.md members section
    if [ -f "$fg/STATE.md" ]; then
      if grep -qi "@$user" "$fg/STATE.md" 2>/dev/null; then
        result="$result $fg_name"
      fi
    fi
  done
  
  echo "$result" | xargs  # Trim whitespace
  return 0
}
```

### approval_matrix_is_admin

Check if user has admin override authority.

```bash
# Usage: approval_matrix_is_admin <user>
# Returns: "true" or "false" with exit code
approval_matrix_is_admin() {
  local user="$1"
  local repo_path="${2:-.}"
  
  # Admin list from config or WGSD.md
  local config_file="$repo_path/.planning/WGSD.md"
  
  if [ -f "$config_file" ]; then
    # Look for admins section
    if grep -qi "Admin.*@$user" "$config_file" 2>/dev/null || \
       grep -qi "Owner.*@$user" "$config_file" 2>/dev/null; then
      echo "true"
      return 0
    fi
  fi
  
  # Fallback: Check if user is repo owner (via git config)
  local repo_owner=$(cd "$repo_path" && git remote get-url origin 2>/dev/null | sed 's/.*[:/]\([^/]*\)\/.*/\1/')
  if [ "$repo_owner" = "$user" ]; then
    echo "true"
    return 0
  fi
  
  echo "false"
  return 1
}
```

---

## Core Approval Functions

### approval_matrix_approve

Execute approval for a focus group.

```bash
# Usage: approval_matrix_approve <concept_path> <focus_group> <user> [comment]
# Returns: SUCCESS or ERROR with details
approval_matrix_approve() {
  local concept_path="$1"
  local focus_group="$2"
  local user="$3"
  local comment="${4:-}"
  
  local impact_file="$concept_path/impact-matrix.md"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  local today=$(date +%Y-%m-%d)
  
  # Validate impact file exists
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found: $impact_file"
    return 1
  fi
  
  # 1. Validate authority
  if [ "$(approval_matrix_can_approve "$user" "$focus_group")" != "true" ]; then
    echo "ERROR: User @$user does not have authority to approve for $focus_group"
    return 1
  fi
  
  # 2. Check current status (can't approve if already approved)
  local impacts=$(impact_parse_file "$impact_file")
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ "$current_status" = "approved" ]; then
    echo "INFO: $focus_group already approved this concept"
    return 0
  fi
  
  if [ -z "$current_status" ]; then
    echo "ERROR: Focus group $focus_group has no impact declared on this concept"
    return 1
  fi
  
  # 3. Update impact-matrix.md with approval
  impact_update "$impact_file" "$focus_group" "status" "approved"
  impact_update "$impact_file" "$focus_group" "approver" "@$user"
  impact_update "$impact_file" "$focus_group" "approved_date" "$today"
  
  if [ -n "$comment" ]; then
    impact_update "$impact_file" "$focus_group" "approval_comment" "$comment"
  fi
  
  # 4. Clear blocked_by if it was set
  impact_update "$impact_file" "$focus_group" "blocked_by" "null"
  impact_update "$impact_file" "$focus_group" "blocked_reason" "null"
  impact_update "$impact_file" "$focus_group" "blocked_date" "null"
  
  # 5. Auto-unblock any FGs that were waiting on this one
  approval_matrix_auto_unblock "$concept_path" "$focus_group"
  
  # 6. Check for full approval completion
  local is_complete=$(approval_matrix_is_complete "$impact_file")
  
  echo "SUCCESS"
  echo "APPROVAL:$focus_group:$user:$today"
  
  if [ "$is_complete" = "true" ]; then
    echo "FULLY_APPROVED:$concept_path"
  fi
  
  return 0
}
```

### approval_matrix_reject

Reject approval for a focus group.

```bash
# Usage: approval_matrix_reject <concept_path> <focus_group> <user> <reason>
# Returns: SUCCESS or ERROR with details
approval_matrix_reject() {
  local concept_path="$1"
  local focus_group="$2"
  local user="$3"
  local reason="$4"
  
  local impact_file="$concept_path/impact-matrix.md"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  local today=$(date +%Y-%m-%d)
  
  # Validate impact file exists
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found: $impact_file"
    return 1
  fi
  
  # Validate authority
  if [ "$(approval_matrix_can_approve "$user" "$focus_group")" != "true" ]; then
    echo "ERROR: User @$user does not have authority to reject for $focus_group"
    return 1
  fi
  
  # Check current status
  local impacts=$(impact_parse_file "$impact_file")
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ -z "$current_status" ]; then
    echo "ERROR: Focus group $focus_group has no impact declared on this concept"
    return 1
  fi
  
  # Update with rejection
  impact_update "$impact_file" "$focus_group" "status" "rejected"
  impact_update "$impact_file" "$focus_group" "approver" "@$user"
  impact_update "$impact_file" "$focus_group" "approved_date" "$today"
  impact_update "$impact_file" "$focus_group" "approval_comment" "$reason"
  
  echo "SUCCESS"
  echo "REJECTION:$focus_group:$user:$reason"
  
  return 0
}
```

### approval_matrix_auto_unblock

Automatically unblock focus groups that were waiting on the approving FG.

```bash
# Usage: approval_matrix_auto_unblock <concept_path> <approved_fg>
# Returns: List of unblocked focus groups
approval_matrix_auto_unblock() {
  local concept_path="$1"
  local approved_fg="$2"
  
  local impact_file="$concept_path/impact-matrix.md"
  local impacts=$(impact_parse_file "$impact_file")
  
  # Find FGs blocked by the just-approved FG
  local blocked_fgs=$(echo "$impacts" | jq -r --arg fg "$approved_fg" \
    '.[] | select(.blocked_by == $fg) | .focus_group')
  
  if [ -z "$blocked_fgs" ]; then
    return 0
  fi
  
  local unblocked=""
  
  for fg in $blocked_fgs; do
    # Unblock: set status to pending, clear blocked_by
    impact_update "$impact_file" "$fg" "status" "pending"
    impact_update "$impact_file" "$fg" "blocked_by" "null"
    impact_update "$impact_file" "$fg" "blocked_reason" "null"
    impact_update "$impact_file" "$fg" "blocked_date" "null"
    
    # Reset SLA now that it's unblocked
    approval_matrix_set_sla "$impact_file" "$fg"
    
    unblocked="$unblocked $fg"
    echo "UNBLOCKED:$fg:was_waiting_on:$approved_fg"
  done
  
  return 0
}
```

---

## Completion Detection

### check_full_approval

Check if concept is ready for implementation (all approvals complete).

```bash
# Usage: check_full_approval <concept_path>
check_full_approval() {
  local concept_path="$1"
  local impact_file="$concept_path/impact-matrix.md"
  
  if [ "$(approval_matrix_is_complete "$impact_file")" = "true" ]; then
    # Trigger completion actions
    approval_matrix_on_complete "$concept_path"
    return 0
  fi
  
  return 1
}
```

### approval_matrix_on_complete

Actions to take when a concept receives full approval.

```bash
# Usage: approval_matrix_on_complete <concept_path>
approval_matrix_on_complete() {
  local concept_path="$1"
  local concept_name=$(basename "$concept_path")
  local concept_file="$concept_path/CONCEPT.md"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  echo "🎉 CONCEPT FULLY APPROVED: $concept_name"
  
  # 1. Update concept status to "Ready for Implementation"
  if [ -f "$concept_file" ]; then
    sed -i 's/\*\*Status:\*\* .*/\*\*Status:\*\* 🚀 Ready for Implementation/' "$concept_file"
  fi
  
  # 2. Create completion marker
  echo "approved_at: $timestamp" > "$concept_path/.approved"
  
  # 3. Generate notification message
  cat <<EOF
✅ **Concept Approved: $concept_name**

All required focus groups have approved this concept!

**Status:** 🚀 Ready for Implementation

**Next steps:**
1. Assign an owner: \`/wgsd assign $concept_name @user\`
2. Create implementation: \`/wgsd create-implementation $concept_name\`
3. Or claim it: \`/wgsd claim $concept_name\`

*Congratulations team on completing the approval process!*
EOF
  
  # 4. Queue for roadmap merge (Phase 13)
  echo "READY_FOR_ROADMAP:$concept_name"
  
  return 0
}
```

---

## Notifications

### send_approval_notification

Send notification to concept channel about approval.

```bash
# Usage: send_approval_notification <concept_path> <focus_group> <user> <status>
send_approval_notification() {
  local concept_path="$1"
  local focus_group="$2"
  local user="$3"
  local status="$4"
  
  local concept_name=$(basename "$concept_path")
  
  # Get approval matrix state
  local matrix=$(approval_matrix_get "$concept_path/impact-matrix.md")
  local approved=$(echo "$matrix" | jq '.approved')
  local total=$(echo "$matrix" | jq '.total_approvals')
  local completion=$(echo "$matrix" | jq '.completion_percent')%
  
  case "$status" in
    approved)
      cat <<EOF
✅ **Approval: $focus_group**

@$user has approved this concept for the **$focus_group** focus group.

**Progress:** $approved/$total ($completion)

$(approval_matrix_get_blockers "$concept_path/impact-matrix.md" | jq -r 'if length > 0 then "**Remaining:** " + ([.[].focus_group] | join(", ")) else "🎉 All approvals complete!" end')
EOF
      ;;
    rejected)
      cat <<EOF
🚫 **Rejection: $focus_group**

@$user has rejected this concept for the **$focus_group** focus group.

**Feedback:**
$(impact_parse_file "$concept_path/impact-matrix.md" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .approval_comment // "No specific feedback provided"')

**Action Required:** Address the concerns and re-request approval.

Use \`/wgsd request-review $concept_name --fg $focus_group\` after addressing feedback.
EOF
      ;;
  esac
}
```

---

## Canvas Integration

### update_approval_canvas

Update concept canvas with current approval state.

```bash
# Usage: update_approval_canvas <concept_path>
update_approval_canvas() {
  local concept_path="$1"
  local concept_name=$(basename "$concept_path")
  
  # Generate approval matrix widget
  local widget=$(template_approval_matrix "$concept_path")
  
  echo "CANVAS_UPDATE:$concept_name"
  echo "$widget"
  
  # Actual canvas update would call canvas-sync workflow
  # /wgsd sync-canvas would pick up the changes
}
```

---

## Command Handler

### Main approve command handler

```bash
# Usage: handle_approve_command <args>
handle_approve_command() {
  local concept_name=""
  local focus_group=""
  local user=""
  local comment=""
  
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
        user="$2"
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
  
  # Validate required arguments
  if [ -z "$concept_name" ]; then
    echo "Usage: /wgsd approve <concept> [--fg <focus_group>] [--comment <comment>]"
    return 1
  fi
  
  if [ -z "$user" ]; then
    echo "ERROR: User context required"
    return 1
  fi
  
  # If no focus group specified, infer from user
  if [ -z "$focus_group" ]; then
    focus_group=$(approval_matrix_get_user_fgs "$user" | awk '{print $1}')
    if [ -z "$focus_group" ]; then
      echo "ERROR: Could not infer focus group for user @$user"
      echo "Use --fg to specify which focus group you're approving for"
      return 1
    fi
    echo "INFO: Approving for inferred focus group: $focus_group"
  fi
  
  # Find concept path
  local concept_path=$(find_concept_path "$concept_name")
  if [ -z "$concept_path" ]; then
    echo "ERROR: Concept not found: $concept_name"
    return 1
  fi
  
  # Execute approval
  local result=$(approval_matrix_approve "$concept_path" "$focus_group" "$user" "$comment")
  
  if echo "$result" | grep -q "^SUCCESS"; then
    echo "$result"
    
    # Send notification
    send_approval_notification "$concept_path" "$focus_group" "$user" "approved"
    
    # Update canvas
    update_approval_canvas "$concept_path"
    
    # Check for full approval
    if echo "$result" | grep -q "FULLY_APPROVED"; then
      echo ""
      approval_matrix_on_complete "$concept_path"
    fi
    
    return 0
  else
    echo "$result"
    return 1
  fi
}
```

---

## Helper Functions

### find_concept_path

Find concept directory by name.

```bash
# Usage: find_concept_path <concept_name>
find_concept_path() {
  local concept_name="$1"
  local repo_path="${2:-.}"
  
  # Search in all focus groups
  for fg_dir in "$repo_path/.planning/focus-groups"/*/; do
    [ -d "$fg_dir" ] || continue
    
    local concept_dir="$fg_dir/concepts/$concept_name"
    
    if [ -d "$concept_dir" ]; then
      echo "$concept_dir"
      return 0
    fi
  done
  
  return 1
}
```

---

## Usage Examples

```bash
# Simple approval (focus group inferred from user)
/wgsd approve oauth-integration

# Approve for specific focus group
/wgsd approve oauth-integration --fg security

# Approve with comment
/wgsd approve oauth-integration --fg api --comment "Endpoints look good, versioning correct"

# Reject with reason
/wgsd reject oauth-integration --fg security --reason "Token refresh flow needs rate limiting"

# Check approval status
/wgsd approval-status oauth-integration

# View approval matrix
/wgsd show-approvals oauth-integration
```

---

## Error Handling

| Error | Resolution |
|-------|------------|
| "No authority" | User not member of focus group. Use `--fg` with correct FG or request membership |
| "Already approved" | Focus group already approved. No action needed |
| "No impact declared" | Focus group has no impact. Use `/wgsd declare-impact` first |
| "Concept not found" | Check concept name spelling or use `/wgsd list-concepts` |

---

*Workflow created for WGSD Phase 12 - Matrix-Based Approval System*
