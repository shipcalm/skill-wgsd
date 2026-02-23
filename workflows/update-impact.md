---
name: wgsd:update-impact
description: Modify existing impact declarations with change tracking and re-notification
triggers:
  - "/wgsd update-impact"
  - "/wgsd modify-impact"
  - "update impact for concept"
dependencies:
  - wgsd:lib:impact-parser
  - wgsd:lib:impact-notifications
---

# Update Impact Workflow

Modify existing impact declarations for a concept. Tracks changes and sends re-notifications to affected focus groups.

---

## Overview

**Purpose:** Enable modification of impact declarations after initial creation

**Change Types:**
- **Add** - New focus group impacts
- **Remove** - Remove existing impacts  
- **Modify** - Change priority, type, or description

**Re-notification Rules:**
- New impacts → Full notification (NEW_IMPACT)
- Modified impacts → Update notification with changes (UPDATED_IMPACT)
- Removed impacts → Removal notification (REMOVED_IMPACT)

---

## Workflow Entry Points

### Entry Point 1: Direct Command
```
/wgsd update-impact {concept-slug}
```

### Entry Point 2: Interactive
```
"Update impacts for oauth-integration"
```

---

## Workflow Steps

### Step 1: Load Concept and Existing Impacts

```yaml
step: load_existing
inputs:
  - concept_slug: string
  - workspace: string (auto-detected)
outputs:
  - impact_matrix_path: string
  - existing_impacts: object[]
  - impact_snapshot: string (JSON for diff)
```

**Actions:**
1. Locate concept directory
2. Load impact-matrix.md
3. Parse existing impacts
4. Create snapshot for change detection

```bash
concept_slug="${1:-}"
workspace=$(pwd)

# Locate concept
concept_dir=""
for dir in "concepts/$concept_slug" ".planning/concepts/$concept_slug" "$concept_slug"; do
  if [ -d "$workspace/$dir" ]; then
    concept_dir="$workspace/$dir"
    break
  fi
done

if [ -z "$concept_dir" ]; then
  echo "ERROR: Concept '$concept_slug' not found"
  exit 1
fi

impact_matrix="$concept_dir/impact-matrix.md"

if [ ! -f "$impact_matrix" ]; then
  echo "ERROR: impact-matrix.md not found"
  exit 1
fi

# Load existing impacts
existing_impacts=$(impact_parse_file "$impact_matrix")
impact_snapshot="$existing_impacts"

echo "📋 Concept: $concept_slug"
echo "   Location: $concept_dir"
echo ""
echo "Current impacts:"
impact_summary "$impact_matrix"
```

---

### Step 2: Choose Update Action

```yaml
step: choose_action
outputs:
  - action: enum (add|remove|modify|batch)
```

**Interactive Menu:**
```
What would you like to do?
[1] Add new focus group impact
[2] Remove existing impact
[3] Modify existing impact
[4] Batch update (multiple changes)
[5] View current impacts
[6] Done
```

```bash
while true; do
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "Update Impact Menu"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "[1] Add new focus group impact"
  echo "[2] Remove existing impact"
  echo "[3] Modify existing impact"
  echo "[4] View current impacts"
  echo "[5] Done - Apply changes"
  echo ""
  read -p "Select action (1-5): " action
  
  case "$action" in
    1) update_action_add ;;
    2) update_action_remove ;;
    3) update_action_modify ;;
    4) impact_summary "$impact_matrix" ;;
    5) break ;;
    *) echo "Invalid selection" ;;
  esac
done
```

---

### Step 3: Add New Impact

```yaml
step: add_impact
# Same flow as declare-impact for single focus group
```

```bash
update_action_add() {
  echo ""
  echo "━━━ Add New Impact ━━━"
  
  # Get available focus groups
  available_fgs=()
  
  # From focus-groups directory
  if [ -d "$workspace/.planning/focus-groups" ]; then
    for fg_dir in "$workspace/.planning/focus-groups"/*/; do
      if [ -d "$fg_dir" ]; then
        available_fgs+=("$(basename "$fg_dir")")
      fi
    done
  fi
  
  # From WGSD config
  config_path="$workspace/.planning/WGSD-CONFIG.md"
  if [ -f "$config_path" ]; then
    while read -r line; do
      if echo "$line" | grep -q "| fg |"; then
        channel=$(echo "$line" | awk -F'|' '{print $2}' | tr -d ' ')
        fg_name=$(echo "$channel" | sed 's/.*-fg-//')
        if [[ ! " ${available_fgs[@]} " =~ " ${fg_name} " ]]; then
          available_fgs+=("$fg_name")
        fi
      fi
    done < "$config_path"
  fi
  
  # Show options (excluding already impacted)
  existing_fgs=$(echo "$existing_impacts" | jq -r '.[].focus_group' 2>/dev/null)
  
  echo "Available focus groups:"
  idx=1
  selectable_fgs=()
  for fg in "${available_fgs[@]}"; do
    if ! echo "$existing_fgs" | grep -qw "$fg"; then
      echo "  [$idx] $fg"
      selectable_fgs+=("$fg")
      ((idx++))
    fi
  done
  
  if [ ${#selectable_fgs[@]} -eq 0 ]; then
    echo "  All focus groups already have impacts"
    return
  fi
  
  read -p "Select focus group (number): " fg_idx
  fg_idx=$((fg_idx - 1))
  
  if [ $fg_idx -lt 0 ] || [ $fg_idx -ge ${#selectable_fgs[@]} ]; then
    echo "Invalid selection"
    return
  fi
  
  fg="${selectable_fgs[$fg_idx]}"
  
  # Collect impact details (same as declare-impact)
  echo ""
  echo "Priority:"
  echo "  [0] P0 - Critical  [1] P1 - High"
  echo "  [2] P2 - Medium    [3] P3 - Low"
  read -p "Select (0-3): " p_choice
  
  case "$p_choice" in
    0) priority="P0" ;; 1) priority="P1" ;;
    2) priority="P2" ;; 3) priority="P3" ;;
    *) priority="P2" ;;
  esac
  
  echo ""
  echo "Type: [1]breaking [2]api [3]integration [4]docs [5]testing [6]behavior [7]security [8]performance"
  read -p "Select (1-8): " t_choice
  
  case "$t_choice" in
    1) type="breaking-change" ;; 2) type="api-change" ;;
    3) type="integration" ;; 4) type="documentation" ;;
    5) type="testing" ;; 6) type="behavior" ;;
    7) type="security" ;; 8) type="performance" ;;
    *) type="integration" ;;
  esac
  
  read -p "Description: " description
  [ -z "$description" ] && description="Impact on $fg"
  
  # Add impact
  if impact_add "$impact_matrix" "$fg" "$priority" "$type" "$description"; then
    # Track change
    changes+=("ADD:$fg:$priority:$type")
    echo "✅ Impact added: $fg"
    
    # Refresh existing impacts
    existing_impacts=$(impact_parse_file "$impact_matrix")
  fi
}
```

---

### Step 4: Remove Impact

```yaml
step: remove_impact
```

```bash
update_action_remove() {
  echo ""
  echo "━━━ Remove Impact ━━━"
  
  # List current impacts
  echo "Current impacts:"
  idx=1
  fgs=()
  echo "$existing_impacts" | jq -c '.[]' | while read -r impact; do
    fg=$(echo "$impact" | jq -r '.focus_group')
    priority=$(echo "$impact" | jq -r '.priority')
    type=$(echo "$impact" | jq -r '.type')
    echo "  [$idx] $fg ($priority, $type)"
    fgs+=("$fg")
    ((idx++))
  done
  
  # Re-populate fgs array (subshell issue workaround)
  fgs=($(echo "$existing_impacts" | jq -r '.[].focus_group'))
  
  if [ ${#fgs[@]} -eq 0 ]; then
    echo "  No impacts to remove"
    return
  fi
  
  # Check for primary owner
  primary_fg=$(echo "$existing_impacts" | jq -r '.[] | select(.type == "primary-owner") | .focus_group' | head -1)
  
  read -p "Select impact to remove (number): " fg_idx
  fg_idx=$((fg_idx - 1))
  
  if [ $fg_idx -lt 0 ] || [ $fg_idx -ge ${#fgs[@]} ]; then
    echo "Invalid selection"
    return
  fi
  
  fg="${fgs[$fg_idx]}"
  
  # Warn if removing primary owner
  if [ "$fg" = "$primary_fg" ]; then
    echo "⚠️  Warning: This is the primary owner focus group"
    read -p "Are you sure you want to remove it? (y/n): " confirm
    if [ "$confirm" != "y" ]; then
      return
    fi
  fi
  
  # Remove impact
  if impact_remove "$impact_matrix" "$fg"; then
    changes+=("REMOVE:$fg")
    removed_fgs+=("$fg")
    echo "✅ Impact removed: $fg"
    
    # Refresh
    existing_impacts=$(impact_parse_file "$impact_matrix")
  fi
}
```

---

### Step 5: Modify Impact

```yaml
step: modify_impact
```

```bash
update_action_modify() {
  echo ""
  echo "━━━ Modify Impact ━━━"
  
  # List current impacts
  echo "Current impacts:"
  fgs=($(echo "$existing_impacts" | jq -r '.[].focus_group'))
  
  idx=1
  for fg in "${fgs[@]}"; do
    impact=$(echo "$existing_impacts" | jq -c ".[] | select(.focus_group == \"$fg\")")
    priority=$(echo "$impact" | jq -r '.priority')
    type=$(echo "$impact" | jq -r '.type')
    echo "  [$idx] $fg ($priority, $type)"
    ((idx++))
  done
  
  if [ ${#fgs[@]} -eq 0 ]; then
    echo "  No impacts to modify"
    return
  fi
  
  read -p "Select impact to modify (number): " fg_idx
  fg_idx=$((fg_idx - 1))
  
  if [ $fg_idx -lt 0 ] || [ $fg_idx -ge ${#fgs[@]} ]; then
    echo "Invalid selection"
    return
  fi
  
  fg="${fgs[$fg_idx]}"
  impact=$(echo "$existing_impacts" | jq -c ".[] | select(.focus_group == \"$fg\")")
  
  echo ""
  echo "Current values for $fg:"
  echo "  Priority: $(echo "$impact" | jq -r '.priority')"
  echo "  Type: $(echo "$impact" | jq -r '.type')"
  echo "  Description: $(echo "$impact" | jq -r '.description')"
  echo ""
  
  # What to modify
  echo "What to modify?"
  echo "  [1] Priority"
  echo "  [2] Type"
  echo "  [3] Description"
  echo "  [4] All"
  read -p "Select (1-4): " modify_choice
  
  old_priority=$(echo "$impact" | jq -r '.priority')
  old_type=$(echo "$impact" | jq -r '.type')
  old_desc=$(echo "$impact" | jq -r '.description')
  
  new_priority="$old_priority"
  new_type="$old_type"
  new_desc="$old_desc"
  
  field_changes=()
  
  if [ "$modify_choice" = "1" ] || [ "$modify_choice" = "4" ]; then
    echo "New priority [0]P0 [1]P1 [2]P2 [3]P3 (current: $old_priority):"
    read -p "Select: " p_choice
    case "$p_choice" in
      0) new_priority="P0" ;; 1) new_priority="P1" ;;
      2) new_priority="P2" ;; 3) new_priority="P3" ;;
    esac
    if [ "$new_priority" != "$old_priority" ]; then
      impact_update "$impact_matrix" "$fg" "priority" "$new_priority"
      field_changes+=("{\"field\":\"priority\",\"old\":\"$old_priority\",\"new\":\"$new_priority\"}")
    fi
  fi
  
  if [ "$modify_choice" = "2" ] || [ "$modify_choice" = "4" ]; then
    echo "New type [1]breaking [2]api [3]integration [4]docs [5]testing [6]behavior [7]security [8]performance:"
    read -p "Select: " t_choice
    case "$t_choice" in
      1) new_type="breaking-change" ;; 2) new_type="api-change" ;;
      3) new_type="integration" ;; 4) new_type="documentation" ;;
      5) new_type="testing" ;; 6) new_type="behavior" ;;
      7) new_type="security" ;; 8) new_type="performance" ;;
    esac
    if [ "$new_type" != "$old_type" ]; then
      impact_update "$impact_matrix" "$fg" "type" "$new_type"
      field_changes+=("{\"field\":\"type\",\"old\":\"$old_type\",\"new\":\"$new_type\"}")
    fi
  fi
  
  if [ "$modify_choice" = "3" ] || [ "$modify_choice" = "4" ]; then
    read -p "New description: " new_desc
    if [ -n "$new_desc" ] && [ "$new_desc" != "$old_desc" ]; then
      impact_update "$impact_matrix" "$fg" "description" "$new_desc"
      field_changes+=("{\"field\":\"description\",\"old\":\"$old_desc\",\"new\":\"$new_desc\"}")
    fi
  fi
  
  if [ ${#field_changes[@]} -gt 0 ]; then
    changes+=("MODIFY:$fg")
    modified_impacts+=("$fg:[${field_changes[*]}]")
    echo "✅ Impact modified: $fg"
    
    # Reset approval status (modified impacts need re-approval)
    impact_update "$impact_matrix" "$fg" "status" "pending"
    impact_update "$impact_matrix" "$fg" "approver" "null"
    impact_update "$impact_matrix" "$fg" "approved_date" "null"
    
    # Refresh
    existing_impacts=$(impact_parse_file "$impact_matrix")
  else
    echo "No changes made"
  fi
}
```

---

### Step 6: Apply Changes and Notify

```yaml
step: apply_and_notify
inputs:
  - changes: string[]
  - impact_snapshot: string (original state)
  - existing_impacts: string (new state)
```

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📊 Change Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

# Calculate diff
diff_result=$(impact_diff "$impact_snapshot" "$existing_impacts")
has_changes=$(echo "$diff_result" | jq -r '.has_changes')

if [ "$has_changes" != "true" ]; then
  echo "No changes made"
  exit 0
fi

# Display changes
added=$(echo "$diff_result" | jq -r '.added[]' 2>/dev/null)
removed=$(echo "$diff_result" | jq -r '.removed[]' 2>/dev/null)
modified=$(echo "$diff_result" | jq -c '.modified[]' 2>/dev/null)

[ -n "$added" ] && echo "➕ Added: $added"
[ -n "$removed" ] && echo "➖ Removed: $removed"

echo "$modified" | while read -r mod; do
  [ -z "$mod" ] && continue
  fg=$(echo "$mod" | jq -r '.focus_group')
  echo "📝 Modified: $fg"
  echo "$mod" | jq -r '.changes[] | "   - \(.field): \(.old) → \(.new)"'
done

echo ""
read -p "Send notifications for changes? (y/n): " notify

if [ "$notify" = "y" ] || [ "$notify" = "Y" ]; then
  echo ""
  echo "📤 Sending notifications..."
  
  # Notify removed impacts
  for fg in $removed; do
    impact_notify_removed "$workspace" "$concept_slug" "$fg"
  done
  
  # Notify modified impacts
  echo "$modified" | while read -r mod; do
    [ -z "$mod" ] && continue
    fg=$(echo "$mod" | jq -r '.focus_group')
    changes_json=$(echo "$mod" | jq -c '.changes')
    impact_notify_updated "$workspace" "$concept_slug" "$fg" "$changes_json"
  done
  
  # Notify new impacts
  for fg in $added; do
    impact=$(echo "$existing_impacts" | jq -c ".[] | select(.focus_group == \"$fg\")")
    impact_notify_new "$workspace" "$concept_slug" "$impact"
    impact_mark_notified "$impact_matrix" "$fg"
  done
  
  echo "✅ Notifications sent"
fi
```

---

### Step 7: Update Change Log

```yaml
step: update_changelog
```

```bash
# Append to change log in impact-matrix.md
today=$(date +%Y-%m-%d)
author="${USER:-unknown}"

# Build change description
change_desc=""
[ -n "$added" ] && change_desc="Added impacts: $added"
[ -n "$removed" ] && change_desc="$change_desc; Removed impacts: $removed"
[ -n "$modified" ] && {
  mod_fgs=$(echo "$diff_result" | jq -r '.modified[].focus_group' | tr '\n' ', ' | sed 's/,$//')
  change_desc="$change_desc; Modified impacts: $mod_fgs"
}
change_desc=$(echo "$change_desc" | sed 's/^; //')

# Append to markdown table
if grep -q "^## Change Log" "$impact_matrix"; then
  # Insert new row after table header
  sed -i "/^## Change Log/,/^---$/{
    /^|.*|.*|.*|$/a | $today | $change_desc | $author |
  }" "$impact_matrix" 2>/dev/null || {
    # Fallback: append to end
    echo "| $today | $change_desc | $author |" >> "$impact_matrix"
  }
fi

echo ""
echo "📝 Change log updated"
```

---

### Step 8: Display Final State

```yaml
step: display_final
```

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Updated Impact Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

impact_summary "$impact_matrix"

echo ""
echo "🎯 Next Steps:"
echo "• Modified impacts require re-approval from focus groups"
echo "• View status: /wgsd concept $concept_slug"
echo "• Sync to Canvas: /wgsd canvas-sync $concept_slug"
```

---

## Non-Interactive Mode

```bash
# Add impact
/wgsd update-impact oauth-integration --add security P0 breaking-change "Token changes"

# Remove impact
/wgsd update-impact oauth-integration --remove frontend

# Modify impact
/wgsd update-impact oauth-integration --modify api priority P0
```

---

## Error Handling

| Error | Resolution |
|-------|------------|
| Cannot remove primary owner | Warn user, require confirmation |
| Impact not found | List available impacts |
| Validation failed | Show schema requirements |
| Notification failed | Log and continue |

---

## Change Detection Logic

```python
# Pseudo-code for change detection
def detect_changes(old_impacts, new_impacts):
    old_by_fg = {i['focus_group']: i for i in old_impacts}
    new_by_fg = {i['focus_group']: i for i in new_impacts}
    
    added = set(new_by_fg.keys()) - set(old_by_fg.keys())
    removed = set(old_by_fg.keys()) - set(new_by_fg.keys())
    
    modified = []
    for fg in old_by_fg.keys() & new_by_fg.keys():
        old = old_by_fg[fg]
        new = new_by_fg[fg]
        changes = []
        for field in ['priority', 'type', 'description']:
            if old.get(field) != new.get(field):
                changes.append({
                    'field': field,
                    'old': old.get(field),
                    'new': new.get(field)
                })
        if changes:
            modified.append({'focus_group': fg, 'changes': changes})
    
    return {'added': added, 'removed': removed, 'modified': modified}
```

---

## Re-Approval Rules

When impacts are modified:
1. Status resets to `pending`
2. Approver cleared
3. Approved date cleared
4. Focus group notified of changes
5. Concept cannot proceed to implementation until re-approved

---

*Workflow created for WGSD Phase 11 - Cross-Cutting Impact System*
