---
name: wgsd:workflow:matrix-block
description: Block declaration and dependency management for matrix-based approval
triggers:
  - command: "/wgsd block {concept} --fg {focus_group} --blocked-by {blocking_fg}"
  - command: "/wgsd unblock {concept} --fg {focus_group}"
---

# Matrix Block Workflow

Handle blocking dependencies between focus group approvals in the matrix-based approval system.

---

## Overview

Blocking dependencies allow focus groups to declare that they cannot approve until another focus group approves first. This enables:
- Sequential approval requirements (e.g., Security before Frontend)
- Explicit dependency tracking
- Auto-unblocking when dependencies are resolved
- Cascade blocking on rejections

---

## Entry Points

### Block Declaration
```bash
# Declare a blocking dependency
/wgsd block oauth-integration --fg frontend --blocked-by api --reason "Cannot review UI until API endpoints finalized"

# Short form
/wgsd block oauth-integration --fg frontend --blocked-by api
```

### Unblock
```bash
# Manually unblock (admin/override)
/wgsd unblock oauth-integration --fg frontend

# Unblock with reason
/wgsd unblock oauth-integration --fg frontend --reason "API spec shared separately"
```

---

## Workflow: Block Declaration

```yaml
workflow:
  name: matrix-block
  version: "2.2"
  
  inputs:
    concept_name: string          # Required: Concept slug
    focus_group: string           # Required: FG declaring they're blocked
    blocked_by: string            # Required: FG they're waiting on
    reason: string | null         # Optional: Why blocked
    user: string                  # Required: User declaring block
  
  steps:
    - id: validate_inputs
      action: validate_block_inputs
      description: Ensure FGs exist and dependency is valid
      
    - id: check_authority
      action: check_user_authority
      description: Verify user can declare block for focus group
      
    - id: check_circular
      action: detect_circular_dependency
      description: Prevent circular blocking
      
    - id: update_status
      action: set_blocked_status
      description: Update impact-matrix.md with blocked status
      
    - id: notify
      action: send_block_notifications
      description: Notify both focus groups
      
    - id: update_canvas
      action: sync_canvas
      description: Update concept canvas
```

---

## Core Block Functions

### approval_matrix_block

Block a focus group's approval until another FG approves.

```bash
# Usage: approval_matrix_block <concept_path> <focus_group> <blocked_by> <user> [reason]
approval_matrix_block() {
  local concept_path="$1"
  local focus_group="$2"
  local blocked_by="$3"
  local user="$4"
  local reason="${5:-Waiting on $blocked_by}"
  
  local impact_file="$concept_path/impact-matrix.md"
  local today=$(date +%Y-%m-%d)
  
  # Validate impact file exists
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found: $impact_file"
    return 1
  fi
  
  # 1. Validate both focus groups exist in impact matrix
  local impacts=$(impact_parse_file "$impact_file")
  
  local fg_exists=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .focus_group')
  local blocker_exists=$(echo "$impacts" | jq -r --arg fg "$blocked_by" '.[] | select(.focus_group == $fg) | .focus_group')
  
  if [ -z "$fg_exists" ]; then
    echo "ERROR: Focus group $focus_group has no impact on this concept"
    return 1
  fi
  
  if [ -z "$blocker_exists" ]; then
    echo "ERROR: Focus group $blocked_by has no impact on this concept"
    return 1
  fi
  
  # 2. Check if blocker is already approved (no need to block)
  local blocker_status=$(echo "$impacts" | jq -r --arg fg "$blocked_by" '.[] | select(.focus_group == $fg) | .status')
  
  if [ "$blocker_status" = "approved" ]; then
    echo "INFO: $blocked_by has already approved - no need to block"
    return 0
  fi
  
  # 3. Check for circular dependency
  if approval_matrix_would_create_cycle "$impact_file" "$focus_group" "$blocked_by"; then
    echo "ERROR: Would create circular dependency"
    echo "HINT: Check existing block chain to resolve"
    return 1
  fi
  
  # 4. Check current status of blocking FG
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ "$current_status" = "approved" ]; then
    echo "ERROR: $focus_group has already approved - cannot block retroactively"
    return 1
  fi
  
  # 5. Update impact-matrix.md with blocked status
  impact_update "$impact_file" "$focus_group" "status" "blocked"
  impact_update "$impact_file" "$focus_group" "blocked_by" "$blocked_by"
  impact_update "$impact_file" "$focus_group" "blocked_reason" "$reason"
  impact_update "$impact_file" "$focus_group" "blocked_date" "$today"
  
  echo "SUCCESS"
  echo "BLOCKED:$focus_group:by:$blocked_by"
  echo "REASON:$reason"
  
  return 0
}
```

---

### approval_matrix_unblock

Manually unblock a focus group (admin action or when blocker resolves).

```bash
# Usage: approval_matrix_unblock <concept_path> <focus_group> [reason]
approval_matrix_unblock() {
  local concept_path="$1"
  local focus_group="$2"
  local reason="${3:-Manually unblocked}"
  
  local impact_file="$concept_path/impact-matrix.md"
  
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found"
    return 1
  fi
  
  # Check current status
  local impacts=$(impact_parse_file "$impact_file")
  local current_status=$(echo "$impacts" | jq -r --arg fg "$focus_group" '.[] | select(.focus_group == $fg) | .status')
  
  if [ "$current_status" != "blocked" ]; then
    echo "INFO: $focus_group is not blocked (status: $current_status)"
    return 0
  fi
  
  # Update status back to pending
  impact_update "$impact_file" "$focus_group" "status" "pending"
  impact_update "$impact_file" "$focus_group" "blocked_by" "null"
  impact_update "$impact_file" "$focus_group" "blocked_reason" "null"
  impact_update "$impact_file" "$focus_group" "blocked_date" "null"
  
  # Reset SLA since it's now unblocked
  approval_matrix_set_sla "$impact_file" "$focus_group"
  
  echo "SUCCESS"
  echo "UNBLOCKED:$focus_group"
  echo "REASON:$reason"
  
  return 0
}
```

---

### approval_matrix_would_create_cycle

Check if blocking would create a circular dependency.

```bash
# Usage: approval_matrix_would_create_cycle <impact_file> <focus_group> <blocked_by>
# Returns: "true" if cycle would be created, "false" otherwise
approval_matrix_would_create_cycle() {
  local impact_file="$1"
  local focus_group="$2"
  local blocked_by="$3"
  
  local impacts=$(impact_parse_file "$impact_file")
  
  python3 -c "
import json

impacts = json.loads('''$impacts''')
new_fg = '$focus_group'
new_blocked_by = '$blocked_by'

# Build current dependency graph
blocked_by_map = {}
for impact in impacts:
    fg = impact.get('focus_group')
    blocker = impact.get('blocked_by')
    if blocker:
        blocked_by_map[fg] = blocker

# Add the proposed new dependency
blocked_by_map[new_fg] = new_blocked_by

# DFS to detect cycle
def has_cycle(start, visited=None, path=None):
    if visited is None:
        visited = set()
    if path is None:
        path = set()
    
    if start in path:
        return True  # Cycle detected
    
    if start in visited:
        return False
    
    visited.add(start)
    path.add(start)
    
    # Follow the blocked_by chain
    if start in blocked_by_map:
        if has_cycle(blocked_by_map[start], visited, path):
            return True
    
    path.remove(start)
    return False

# Check for cycle starting from the new blocked_by
# (if blocked_by eventually leads back to focus_group)
if has_cycle(new_fg):
    print('true')
else:
    print('false')
"
}
```

---

### approval_matrix_get_blocked

Get all blocked focus groups and their dependencies.

```bash
# Usage: approval_matrix_get_blocked <impact_file>
# Returns: JSON array of blocked items
approval_matrix_get_blocked() {
  local impact_file="$1"
  
  local impacts=$(impact_parse_file "$impact_file")
  
  python3 -c "
import json

impacts = json.loads('''$impacts''')

blocked = []
for impact in impacts:
    if impact.get('status') == 'blocked':
        blocked.append({
            'focus_group': impact.get('focus_group'),
            'blocked_by': impact.get('blocked_by'),
            'blocked_reason': impact.get('blocked_reason'),
            'blocked_date': impact.get('blocked_date')
        })

print(json.dumps(blocked, indent=2))
"
}
```

---

### approval_matrix_get_blocking_chain

Get the complete blocking chain for visualization.

```bash
# Usage: approval_matrix_get_blocking_chain <impact_file>
# Returns: Formatted blocking chain
approval_matrix_get_blocking_chain() {
  local impact_file="$1"
  
  local impacts=$(impact_parse_file "$impact_file")
  
  python3 -c "
import json

impacts = json.loads('''$impacts''')

# Build blocked_by map
blocked_by = {}
status_map = {}
for impact in impacts:
    fg = impact.get('focus_group')
    status_map[fg] = impact.get('status')
    if impact.get('blocked_by'):
        blocked_by[fg] = impact.get('blocked_by')

if not blocked_by:
    print('No blocking dependencies')
    exit(0)

# Build chains
chains = []
visited = set()

def build_chain(fg, chain=[]):
    chain.append(fg)
    if fg in blocked_by:
        blocker = blocked_by[fg]
        if blocker not in chain:  # Avoid cycles in output
            build_chain(blocker, chain)
    return chain

for fg in blocked_by:
    if fg not in visited:
        chain = build_chain(fg, [])
        chains.append(chain)
        visited.update(chain)

print('Blocking Dependencies:')
print()
for chain in chains:
    status_icons = {
        'pending': '⏳',
        'approved': '✅',
        'blocked': '⏸️',
        'rejected': '🚫'
    }
    
    chain_str = ''
    for i, fg in enumerate(chain):
        icon = status_icons.get(status_map.get(fg, 'pending'), '❓')
        chain_str += f'{icon} {fg}'
        if i < len(chain) - 1:
            chain_str += ' → '
    
    print(f'  {chain_str}')
"
}
```

---

## Cascade Blocking

### approval_matrix_cascade_block

When a focus group rejects, optionally cascade block dependent FGs.

```bash
# Usage: approval_matrix_cascade_block <concept_path> <rejecting_fg>
# Called when a FG rejects to optionally block dependents
approval_matrix_cascade_block() {
  local concept_path="$1"
  local rejecting_fg="$2"
  
  local impact_file="$concept_path/impact-matrix.md"
  local impacts=$(impact_parse_file "$impact_file")
  
  # Find FGs that had this FG as a dependency (were waiting on it)
  local dependents=$(echo "$impacts" | jq -r --arg fg "$rejecting_fg" \
    '.[] | select(.blocked_by == $fg) | .focus_group')
  
  if [ -z "$dependents" ]; then
    return 0
  fi
  
  echo "CASCADE: $rejecting_fg rejection affects:"
  
  for fg in $dependents; do
    # Already blocked by the rejecting FG, update reason
    impact_update "$impact_file" "$fg" "blocked_reason" "Blocked: $rejecting_fg rejected"
    echo "  - $fg (reason updated)"
  done
  
  return 0
}
```

---

## Auto-Unblock on Approval

### approval_matrix_auto_unblock_on_approve

Automatically unblock FGs when their blocker approves.

```bash
# Usage: approval_matrix_auto_unblock_on_approve <concept_path> <approved_fg>
# Called automatically when a FG approves
approval_matrix_auto_unblock_on_approve() {
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
  
  echo "AUTO-UNBLOCK: $approved_fg approval unblocks:"
  
  for fg in $blocked_fgs; do
    approval_matrix_unblock "$concept_path" "$fg" "Auto-unblocked: $approved_fg approved"
    echo "  ✓ $fg is now pending"
  done
  
  return 0
}
```

---

## Notifications

### send_block_notification

Send notification about blocking dependency.

```bash
# Usage: send_block_notification <concept_path> <focus_group> <blocked_by> <reason>
send_block_notification() {
  local concept_path="$1"
  local focus_group="$2"
  local blocked_by="$3"
  local reason="$4"
  
  local concept_name=$(basename "$concept_path")
  
  cat <<EOF
⏸️ **Approval Blocked: $concept_name**

The **$focus_group** focus group has declared they are blocked waiting on **$blocked_by**.

**Reason:** $reason

**What this means:**
- $focus_group cannot complete their review yet
- Once $blocked_by approves, $focus_group will be automatically unblocked
- Use \`/wgsd approval-status $concept_name\` to see the full dependency chain

**To $blocked_by team:**
Please prioritize your review to unblock downstream approvals.

_Chain: $focus_group ← $blocked_by_
EOF
}
```

### send_unblock_notification

Send notification when a FG is unblocked.

```bash
# Usage: send_unblock_notification <concept_path> <focus_group> <reason>
send_unblock_notification() {
  local concept_path="$1"
  local focus_group="$2"
  local reason="$3"
  
  local concept_name=$(basename "$concept_path")
  
  cat <<EOF
✅ **Unblocked: $concept_name**

The **$focus_group** focus group is no longer blocked!

**Reason:** $reason

**Action Required:**
Please review and approve: \`/wgsd approve $concept_name --fg $focus_group\`

_Your SLA timer has been reset._
EOF
}
```

---

## Command Handlers

### handle_block_command

Handle the /wgsd block command.

```bash
# Usage: handle_block_command <args>
handle_block_command() {
  local concept_name=""
  local focus_group=""
  local blocked_by=""
  local reason=""
  local user=""
  
  # Parse arguments
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --fg|--focus-group)
        focus_group="$2"
        shift 2
        ;;
      --blocked-by|--by)
        blocked_by="$2"
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
      *)
        if [ -z "$concept_name" ]; then
          concept_name="$1"
        fi
        shift
        ;;
    esac
  done
  
  # Validate required arguments
  if [ -z "$concept_name" ] || [ -z "$focus_group" ] || [ -z "$blocked_by" ]; then
    echo "Usage: /wgsd block <concept> --fg <focus_group> --blocked-by <blocking_fg> [--reason <reason>]"
    return 1
  fi
  
  # Find concept path
  local concept_path=$(find_concept_path "$concept_name")
  if [ -z "$concept_path" ]; then
    echo "ERROR: Concept not found: $concept_name"
    return 1
  fi
  
  # Execute block
  local result=$(approval_matrix_block "$concept_path" "$focus_group" "$blocked_by" "$user" "$reason")
  
  if echo "$result" | grep -q "^SUCCESS"; then
    echo "$result"
    
    # Send notification
    send_block_notification "$concept_path" "$focus_group" "$blocked_by" "${reason:-Waiting on $blocked_by}"
    
    return 0
  else
    echo "$result"
    return 1
  fi
}
```

### handle_unblock_command

Handle the /wgsd unblock command.

```bash
# Usage: handle_unblock_command <args>
handle_unblock_command() {
  local concept_name=""
  local focus_group=""
  local reason=""
  
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
    echo "Usage: /wgsd unblock <concept> --fg <focus_group> [--reason <reason>]"
    return 1
  fi
  
  # Find concept path
  local concept_path=$(find_concept_path "$concept_name")
  if [ -z "$concept_path" ]; then
    echo "ERROR: Concept not found: $concept_name"
    return 1
  fi
  
  # Execute unblock
  local result=$(approval_matrix_unblock "$concept_path" "$focus_group" "$reason")
  
  if echo "$result" | grep -q "^SUCCESS"; then
    echo "$result"
    
    # Send notification
    send_unblock_notification "$concept_path" "$focus_group" "${reason:-Manually unblocked}"
    
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
# Block frontend waiting on API
/wgsd block oauth-integration --fg frontend --blocked-by api --reason "Need API spec"

# Unblock manually (admin)
/wgsd unblock oauth-integration --fg frontend --reason "Spec shared offline"

# View blocking chain
/wgsd blocking-chain oauth-integration

# Example output:
# Blocking Dependencies:
#   ⏸️ frontend → ⏳ api → ✅ security
#   ⏸️ docs → ⏳ api
```

---

## Error Handling

| Error | Resolution |
|-------|------------|
| "Would create circular dependency" | Review blocking chain, manually unblock one FG |
| "Focus group has no impact" | Use `/wgsd declare-impact` to add FG first |
| "Already approved" | Cannot block after approval |
| "Concept not found" | Check concept name spelling |

---

*Workflow created for WGSD Phase 12 - Matrix-Based Approval System*
