---
name: wgsd:workflow:approval-override
description: Admin override for stuck or urgent approvals in matrix-based system
triggers:
  - command: "/wgsd override {concept} --fg {focus_group} --reason {reason}"
  - command: "/wgsd force-approve {concept} --fg {focus_group}"
---

# Approval Override Workflow

Emergency admin bypass for matrix-based approvals with full audit trail.

---

## Overview

The override workflow allows administrators to force-approve stuck approvals for urgent situations:
- User must have admin role
- Override requires mandatory reason
- Full audit trail maintained
- Original focus group is notified
- Original status preserved for audit

**Use Cases:**
- Focus group member unavailable (vacation, emergency)
- Critical security fix that can't wait
- Time-sensitive release deadline
- Abandoned/dormant focus group

---

## Entry Points

### Override Command
```bash
# Force-approve a stuck approval
/wgsd override oauth-integration --fg api --reason "Critical security fix, API team unavailable"

# Alternative syntax
/wgsd force-approve oauth-integration --fg api --reason "Emergency release deadline"
```

### With Confirmation
```bash
# Interactive mode with confirmation
/wgsd override oauth-integration --fg api --confirm
```

---

## Workflow Steps

```yaml
workflow:
  name: approval-override
  version: "2.2"
  
  inputs:
    concept_name: string          # Required: Concept slug
    focus_group: string           # Required: FG to override
    reason: string                # Required: Why override is needed
    user: string                  # Required: Admin performing override
    skip_confirm: boolean         # Optional: Skip confirmation prompt
  
  steps:
    - id: validate_admin
      action: check_admin_role
      description: Verify user has admin override authority
      
    - id: validate_concept
      action: find_concept_path
      description: Find concept and verify FG has pending approval
      
    - id: confirm_override
      action: prompt_confirmation
      description: Confirm destructive action
      condition: "!skip_confirm"
      
    - id: preserve_original
      action: save_original_state
      description: Save original status for audit
      
    - id: execute_override
      action: force_approve
      description: Update status with override flags
      
    - id: notify_fg
      action: notify_original_fg
      description: Inform focus group of override
      
    - id: check_completion
      action: check_full_approval
      description: Check if concept is now fully approved
      
    - id: audit_log
      action: write_audit_entry
      description: Record override in audit log
```

---

## Admin Authority

### approval_matrix_is_admin

Check if user has admin override authority.

```bash
# Usage: approval_matrix_is_admin <user>
# Returns: "true" or "false"
approval_matrix_is_admin() {
  local user="$1"
  local repo_path="${2:-.}"
  
  # 1. Check WGSD.md for admin list
  local config_file="$repo_path/.planning/WGSD.md"
  
  if [ -f "$config_file" ]; then
    # Look for admins section
    if grep -qiE "^(Admin|Owner|Maintainer).*@$user" "$config_file" 2>/dev/null; then
      echo "true"
      return 0
    fi
    
    # Check admins list
    if sed -n '/## Admins/,/##/p' "$config_file" 2>/dev/null | grep -qi "@$user"; then
      echo "true"
      return 0
    fi
  fi
  
  # 2. Check .planning/config.yaml if exists
  local yaml_config="$repo_path/.planning/config.yaml"
  if [ -f "$yaml_config" ]; then
    if grep -qE "admins:.*$user" "$yaml_config" 2>/dev/null; then
      echo "true"
      return 0
    fi
  fi
  
  # 3. Fallback: Check if user is repo owner (via git config)
  local repo_owner=$(cd "$repo_path" 2>/dev/null && git config --get user.name | tr '[:upper:]' '[:lower:]' 2>/dev/null)
  local user_lower=$(echo "$user" | tr '[:upper:]' '[:lower:]')
  if [ "$repo_owner" = "$user_lower" ]; then
    echo "true"
    return 0
  fi
  
  echo "false"
  return 1
}
```

### approval_matrix_get_admins

Get list of users with admin authority.

```bash
# Usage: approval_matrix_get_admins
# Returns: List of admin usernames
approval_matrix_get_admins() {
  local repo_path="${1:-.}"
  local config_file="$repo_path/.planning/WGSD.md"
  
  if [ ! -f "$config_file" ]; then
    echo "No admin list configured"
    return 1
  fi
  
  # Extract admin usernames from WGSD.md
  grep -oE "@[a-zA-Z0-9_-]+" "$config_file" | \
    grep -iE "admin|owner|maintainer" -B1 | \
    grep -oE "@[a-zA-Z0-9_-]+" | \
    sort -u
  
  return 0
}
```

---

## Override Functions

### approval_matrix_override

Execute admin override on a pending/blocked/rejected approval.

```bash
# Usage: approval_matrix_override <concept_path> <focus_group> <admin_user> <reason>
approval_matrix_override() {
  local concept_path="$1"
  local focus_group="$2"
  local admin_user="$3"
  local reason="$4"
  
  local impact_file="$concept_path/impact-matrix.md"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  local today=$(date +%Y-%m-%d)
  
  # Validate impact file exists
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found: $impact_file"
    return 1
  fi
  
  # 1. Validate admin authority
  if [ "$(approval_matrix_is_admin "$admin_user")" != "true" ]; then
    echo "ERROR: User @$admin_user does not have admin override authority"
    echo "HINT: Contact a project admin or add yourself to .planning/WGSD.md"
    return 1
  fi
  
  # 2. Get current state for preservation
  local impacts=$(impact_parse_file "$impact_file")
  local current=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
    '.[] | select(.focus_group == $fg)')
  
  if [ -z "$current" ] || [ "$current" = "null" ]; then
    echo "ERROR: Focus group $focus_group has no impact on this concept"
    return 1
  fi
  
  local original_status=$(echo "$current" | jq -r '.status')
  local original_approver=$(echo "$current" | jq -r '.approver // empty')
  
  # 3. Check if already approved
  if [ "$original_status" = "approved" ]; then
    local was_overridden=$(echo "$current" | jq -r '.overridden // false')
    if [ "$was_overridden" != "true" ]; then
      echo "INFO: $focus_group already legitimately approved"
      return 0
    fi
  fi
  
  # 4. Execute override - preserve original and mark as overridden
  impact_update "$impact_file" "$focus_group" "original_status" "$original_status"
  impact_update "$impact_file" "$focus_group" "status" "approved"
  impact_update "$impact_file" "$focus_group" "overridden" "true"
  impact_update "$impact_file" "$focus_group" "override_by" "@$admin_user"
  impact_update "$impact_file" "$focus_group" "override_date" "$today"
  impact_update "$impact_file" "$focus_group" "override_reason" "$reason"
  impact_update "$impact_file" "$focus_group" "approver" "@$admin_user (override)"
  impact_update "$impact_file" "$focus_group" "approved_date" "$today"
  
  # 5. Clear blocked status if it was blocked
  if [ "$original_status" = "blocked" ]; then
    impact_update "$impact_file" "$focus_group" "blocked_by" "null"
    impact_update "$impact_file" "$focus_group" "blocked_reason" "null"
  fi
  
  # 6. Auto-unblock any FGs that were waiting on this one
  approval_matrix_auto_unblock_on_approve "$concept_path" "$focus_group"
  
  echo "SUCCESS"
  echo "OVERRIDE:$focus_group:by:$admin_user"
  echo "ORIGINAL_STATUS:$original_status"
  echo "REASON:$reason"
  
  return 0
}
```

---

### approval_matrix_revoke_override

Revoke an override and restore original status (for correction).

```bash
# Usage: approval_matrix_revoke_override <concept_path> <focus_group> <admin_user>
approval_matrix_revoke_override() {
  local concept_path="$1"
  local focus_group="$2"
  local admin_user="$3"
  
  local impact_file="$concept_path/impact-matrix.md"
  
  # Validate admin
  if [ "$(approval_matrix_is_admin "$admin_user")" != "true" ]; then
    echo "ERROR: Admin authority required to revoke override"
    return 1
  fi
  
  # Get current state
  local impacts=$(impact_parse_file "$impact_file")
  local was_overridden=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
    '.[] | select(.focus_group == $fg) | .overridden // false')
  
  if [ "$was_overridden" != "true" ]; then
    echo "ERROR: $focus_group was not overridden"
    return 1
  fi
  
  # Restore original status
  local original_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
    '.[] | select(.focus_group == $fg) | .original_status // "pending"')
  
  impact_update "$impact_file" "$focus_group" "status" "$original_status"
  impact_update "$impact_file" "$focus_group" "overridden" "false"
  impact_update "$impact_file" "$focus_group" "override_by" "null"
  impact_update "$impact_file" "$focus_group" "override_date" "null"
  impact_update "$impact_file" "$focus_group" "override_reason" "null"
  impact_update "$impact_file" "$focus_group" "original_status" "null"
  impact_update "$impact_file" "$focus_group" "approver" "null"
  impact_update "$impact_file" "$focus_group" "approved_date" "null"
  
  echo "SUCCESS"
  echo "REVOKED:$focus_group:restored:$original_status"
  
  return 0
}
```

---

## Notifications

### notify_override

Send notification to original focus group about override.

```bash
# Usage: notify_override <concept_path> <focus_group> <admin_user> <reason>
notify_override() {
  local concept_path="$1"
  local focus_group="$2"
  local admin_user="$3"
  local reason="$4"
  
  local concept_name=$(basename "$concept_path")
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  
  cat <<EOF
⚠️ **Approval Override: $concept_name**

The **$focus_group** focus group approval has been overridden by @$admin_user.

**Reason:** $reason

**What this means:**
- This concept will now proceed in the approval process
- The override is recorded in the audit trail
- Your focus group can still review and provide feedback

**Actions Available:**
- Review the concept: \`/wgsd show $concept_name\`
- Add feedback: Post in the concept channel
- View override details: \`/wgsd audit $concept_name\`

_Override timestamp: $timestamp_

---

*If you believe this override was made in error, please contact @$admin_user or another admin.*
EOF
}
```

### notify_override_to_channel

Send override notification to concept channel.

```bash
# Usage: notify_override_to_channel <concept_path> <focus_group> <admin_user>
notify_override_to_channel() {
  local concept_path="$1"
  local focus_group="$2"
  local admin_user="$3"
  
  local concept_name=$(basename "$concept_path")
  local matrix=$(approval_matrix_get "$concept_path/impact-matrix.md")
  local approved=$(echo "$matrix" | jq '.approved')
  local total=$(echo "$matrix" | jq '.total_approvals')
  
  cat <<EOF
⚡ **Admin Override Applied**

@$admin_user has overridden the **$focus_group** approval.

**Progress:** $approved/$total approvals complete

$(if [ "$(echo "$matrix" | jq '.fully_approved')" = "true" ]; then
  echo "🎉 **Concept is now fully approved!**"
else
  echo "Remaining: $(approval_matrix_get_blockers "$concept_path/impact-matrix.md" | jq -r '[.[].focus_group] | join(", ")')"
fi)
EOF
}
```

---

## Audit Trail

### write_override_audit

Write override to audit log.

```bash
# Usage: write_override_audit <concept_path> <focus_group> <admin_user> <reason> <original_status>
write_override_audit() {
  local concept_path="$1"
  local focus_group="$2"
  local admin_user="$3"
  local reason="$4"
  local original_status="$5"
  
  local audit_file="$concept_path/.audit-log"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  cat >> "$audit_file" <<EOF
---
type: approval_override
timestamp: $timestamp
focus_group: $focus_group
admin: @$admin_user
original_status: $original_status
reason: "$reason"
EOF

  echo "AUDIT:written:$audit_file"
  return 0
}
```

### get_override_history

Get history of overrides for a concept.

```bash
# Usage: get_override_history <concept_path>
get_override_history() {
  local concept_path="$1"
  local audit_file="$concept_path/.audit-log"
  
  if [ ! -f "$audit_file" ]; then
    echo "No override history"
    return 0
  fi
  
  echo "## Override History"
  echo ""
  
  grep -A5 "type: approval_override" "$audit_file" | \
    awk '/timestamp:/ {ts=$2} /focus_group:/ {fg=$2} /admin:/ {admin=$2} /reason:/ {r=$0; sub(/reason: /, "", r)} 
         /---/ {if (ts) print "- " ts ": " fg " overridden by " admin; if (r) print "  Reason: " r; ts=""; r=""}'
}
```

---

## Command Handler

### handle_override_command

Handle the /wgsd override command.

```bash
# Usage: handle_override_command <args>
handle_override_command() {
  local concept_name=""
  local focus_group=""
  local reason=""
  local user=""
  local skip_confirm=false
  
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
        user="$2"
        shift 2
        ;;
      --confirm|-y)
        skip_confirm=true
        shift
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
  if [ -z "$concept_name" ] || [ -z "$focus_group" ]; then
    echo "Usage: /wgsd override <concept> --fg <focus_group> --reason <reason>"
    echo ""
    echo "Options:"
    echo "  --fg, --focus-group   Focus group to override"
    echo "  --reason, -r          Reason for override (required)"
    echo "  --confirm, -y         Skip confirmation prompt"
    return 1
  fi
  
  if [ -z "$reason" ]; then
    echo "ERROR: Reason is required for override"
    echo "Use: /wgsd override $concept_name --fg $focus_group --reason \"your reason\""
    return 1
  fi
  
  if [ -z "$user" ]; then
    echo "ERROR: User context required"
    return 1
  fi
  
  # Verify admin status first
  if [ "$(approval_matrix_is_admin "$user")" != "true" ]; then
    echo "❌ **Permission Denied**"
    echo ""
    echo "You do not have admin override authority."
    echo ""
    echo "**Current admins:**"
    approval_matrix_get_admins
    echo ""
    echo "Contact an admin to perform this override, or add yourself to \`.planning/WGSD.md\`."
    return 1
  fi
  
  # Find concept path
  local concept_path=$(find_concept_path "$concept_name")
  if [ -z "$concept_path" ]; then
    echo "ERROR: Concept not found: $concept_name"
    return 1
  fi
  
  # Confirmation (if not skipped)
  if [ "$skip_confirm" = false ]; then
    echo "⚠️ **Override Confirmation**"
    echo ""
    echo "You are about to override the **$focus_group** approval for **$concept_name**."
    echo ""
    echo "**Reason:** $reason"
    echo ""
    echo "This will:"
    echo "- Force-approve the $focus_group impact"
    echo "- Notify the $focus_group team"
    echo "- Create an audit trail entry"
    echo "- Potentially trigger full approval completion"
    echo ""
    echo "To confirm, run with --confirm flag:"
    echo "\`/wgsd override $concept_name --fg $focus_group --reason \"$reason\" --confirm\`"
    return 0
  fi
  
  # Get original status for audit
  local impacts=$(impact_parse_file "$concept_path/impact-matrix.md")
  local original_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" \
    '.[] | select(.focus_group == $fg) | .status')
  
  # Execute override
  local result=$(approval_matrix_override "$concept_path" "$focus_group" "$user" "$reason")
  
  if echo "$result" | grep -q "^SUCCESS"; then
    echo "$result"
    
    # Write audit log
    write_override_audit "$concept_path" "$focus_group" "$user" "$reason" "$original_status"
    
    # Send notifications
    notify_override "$concept_path" "$focus_group" "$user" "$reason"
    notify_override_to_channel "$concept_path" "$focus_group" "$user"
    
    # Check for full approval
    local is_complete=$(approval_matrix_is_complete "$concept_path/impact-matrix.md")
    if [ "$is_complete" = "true" ]; then
      echo ""
      echo "🎉 **Concept Fully Approved!**"
      approval_matrix_trigger_on_complete "$concept_path"
    fi
    
    return 0
  else
    echo "$result"
    return 1
  fi
}
```

---

## Usage Examples

```bash
# Override a stuck approval (with confirmation)
/wgsd override oauth-integration --fg api --reason "Team on vacation, urgent release"

# Override with auto-confirm
/wgsd override oauth-integration --fg api --reason "Critical security fix" --confirm

# Revoke an override (admin)
/wgsd revoke-override oauth-integration --fg api

# View override history
/wgsd audit oauth-integration

# List admins
/wgsd list-admins
```

---

## Error Handling

| Error | Resolution |
|-------|------------|
| "Permission denied" | User not admin. Contact admin or update WGSD.md |
| "Reason required" | Must provide --reason argument |
| "Focus group has no impact" | FG not in impact matrix. Use /wgsd declare-impact |
| "Already legitimately approved" | FG approved normally, no override needed |

---

## Security Considerations

1. **Audit Trail**: All overrides are logged with timestamp, user, and reason
2. **Notifications**: Original FG is always notified
3. **Reversibility**: Overrides can be revoked by admins
4. **Visibility**: Override status visible in approval matrix widget
5. **Rate Limiting**: Consider implementing cooldown between overrides

---

*Workflow created for WGSD Phase 12 - Matrix-Based Approval System*
