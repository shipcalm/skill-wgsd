---
name: wgsd:declare-impact
description: Interactive workflow to declare which focus groups are impacted by a concept
triggers:
  - "/wgsd declare-impact"
  - "/wgsd impact"
  - "declare impact for concept"
dependencies:
  - wgsd:lib:impact-parser
  - wgsd:lib:impact-notifications
  - wgsd:lib:channel-registry
---

# Declare Impact Workflow

Interactively declare which focus groups are impacted by a concept. Each impact becomes an approval requirement before implementation.

---

## Overview

**Purpose:** Enable concepts to declare cross-cutting impacts across multiple focus groups

**Example:** An "OAuth Integration" concept might impact:
- 🔐 **Security** (P0) - Token validation flow changes
- 🔌 **API** (P1) - Auth endpoint signature changes  
- 🎨 **Frontend** (P2) - Login UI modifications

---

## Workflow Entry Points

### Entry Point 1: Direct Command
```
/wgsd declare-impact {concept-slug}
```

### Entry Point 2: During Concept Creation
Called automatically from `create-concept.md` when user opts to declare impacts.

### Entry Point 3: Interactive Prompt
```
"Declare impacts for oauth-integration concept"
```

---

## Workflow Steps

### Step 1: Identify Concept

```yaml
step: identify_concept
inputs:
  - concept_slug: string (from command or prompt)
  - workspace: string (auto-detected)
```

**Actions:**
1. Parse concept slug from input
2. Locate concept directory
3. Verify impact-matrix.md exists
4. Load existing impacts

```bash
# Locate concept
concept_slug="${1:-}"
workspace=$(pwd)

# Try to find concept directory
concept_dir=""
for dir in "concepts/$concept_slug" ".planning/concepts/$concept_slug" "$concept_slug"; do
  if [ -d "$workspace/$dir" ]; then
    concept_dir="$workspace/$dir"
    break
  fi
done

if [ -z "$concept_dir" ]; then
  echo "ERROR: Concept '$concept_slug' not found"
  echo ""
  echo "Available concepts:"
  ls -d concepts/*/ .planning/concepts/*/ 2>/dev/null | xargs -n1 basename
  exit 1
fi

impact_matrix="$concept_dir/impact-matrix.md"

if [ ! -f "$impact_matrix" ]; then
  echo "ERROR: impact-matrix.md not found in $concept_dir"
  echo "Run /wgsd create-concept to create a concept with impact matrix"
  exit 1
fi

echo "📋 Found concept: $concept_slug"
echo "   Location: $concept_dir"
```

---

### Step 2: Discover Available Focus Groups

```yaml
step: discover_focus_groups
outputs:
  - available_fgs: string[] (list of focus group names)
  - existing_impacts: object[] (already declared impacts)
```

**Actions:**
1. List focus groups from `.planning/focus-groups/`
2. List focus group channels from WGSD config
3. Load existing impacts from impact-matrix.md

```bash
# Get available focus groups from directory structure
echo ""
echo "🔍 Discovering focus groups..."

available_fgs=()

# From focus-groups directory
if [ -d "$workspace/.planning/focus-groups" ]; then
  for fg_dir in "$workspace/.planning/focus-groups"/*/; do
    if [ -d "$fg_dir" ]; then
      fg_name=$(basename "$fg_dir")
      available_fgs+=("$fg_name")
    fi
  done
fi

# From WGSD config (focus group channels)
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

# Get existing impacts
existing_impacts=$(impact_parse_file "$impact_matrix")
existing_fgs=$(echo "$existing_impacts" | jq -r '.[].focus_group' 2>/dev/null | tr '\n' ' ')

echo ""
echo "Available focus groups:"
for i in "${!available_fgs[@]}"; do
  fg="${available_fgs[$i]}"
  if echo "$existing_fgs" | grep -qw "$fg"; then
    echo "  [$((i+1))] $fg ✓ (already has impact)"
  else
    echo "  [$((i+1))] $fg"
  fi
done
echo ""
```

---

### Step 3: Select Focus Groups

```yaml
step: select_focus_groups
inputs:
  - selection: string (comma-separated numbers or names)
outputs:
  - selected_fgs: string[] (focus groups to add impacts for)
```

**Interactive Prompt:**
```
Which focus groups does this concept impact?
Enter numbers (e.g., 1,3,4) or names (e.g., security,api):
```

**Actions:**
1. Parse user selection
2. Validate against available focus groups
3. Skip already-impacted focus groups (with option to update)

```bash
echo "Select focus groups to add impacts for:"
echo "(Enter numbers like '1,3' or names like 'security,api')"
echo ""
read -p "Selection: " selection

selected_fgs=()

# Parse selection
IFS=',' read -ra parts <<< "$selection"
for part in "${parts[@]}"; do
  part=$(echo "$part" | tr -d ' ')
  
  # Check if number
  if [[ "$part" =~ ^[0-9]+$ ]]; then
    idx=$((part - 1))
    if [ $idx -ge 0 ] && [ $idx -lt ${#available_fgs[@]} ]; then
      selected_fgs+=("${available_fgs[$idx]}")
    else
      echo "⚠️  Invalid number: $part"
    fi
  else
    # Check if name matches
    for fg in "${available_fgs[@]}"; do
      if [ "$fg" = "$part" ]; then
        selected_fgs+=("$fg")
        break
      fi
    done
  fi
done

if [ ${#selected_fgs[@]} -eq 0 ]; then
  echo "❌ No valid focus groups selected"
  exit 1
fi

echo ""
echo "Selected focus groups: ${selected_fgs[*]}"
```

---

### Step 4: Collect Impact Details

```yaml
step: collect_impact_details
inputs:
  - selected_fgs: string[]
outputs:
  - impacts: object[] (fully specified impacts)
```

**For each focus group, prompt:**
1. Priority (P0/P1/P2/P3)
2. Impact Type (from enum)
3. Description

```bash
impacts=()

echo ""
echo "📝 Collecting impact details..."

for fg in "${selected_fgs[@]}"; do
  echo ""
  echo "━━━ $fg ━━━"
  
  # Priority
  echo "Priority:"
  echo "  [0] P0 - Critical (4h SLA)"
  echo "  [1] P1 - High (24h SLA)"
  echo "  [2] P2 - Medium (72h SLA)"
  echo "  [3] P3 - Low (7d SLA)"
  read -p "Select (0-3): " p_choice
  
  case "$p_choice" in
    0) priority="P0" ;;
    1) priority="P1" ;;
    2) priority="P2" ;;
    3) priority="P3" ;;
    *) priority="P2"; echo "  (defaulting to P2)" ;;
  esac
  
  # Type
  echo ""
  echo "Impact Type:"
  echo "  [1] breaking-change - API/behavior breaking changes"
  echo "  [2] api-change - Non-breaking API modifications"
  echo "  [3] integration - Integration touchpoints affected"
  echo "  [4] documentation - Docs must be updated"
  echo "  [5] testing - Test suite changes needed"
  echo "  [6] behavior - Behavior changes (non-breaking)"
  echo "  [7] security - Security implications"
  echo "  [8] performance - Performance considerations"
  read -p "Select (1-8): " t_choice
  
  case "$t_choice" in
    1) type="breaking-change" ;;
    2) type="api-change" ;;
    3) type="integration" ;;
    4) type="documentation" ;;
    5) type="testing" ;;
    6) type="behavior" ;;
    7) type="security" ;;
    8) type="performance" ;;
    *) type="integration"; echo "  (defaulting to integration)" ;;
  esac
  
  # Description
  echo ""
  read -p "Description (how does this concept impact $fg?): " description
  
  if [ -z "$description" ]; then
    description="Impact on $fg focus group"
  fi
  
  # Store impact
  impact_json=$(jq -n \
    --arg fg "$fg" \
    --arg priority "$priority" \
    --arg type "$type" \
    --arg desc "$description" \
    '{focus_group: $fg, priority: $priority, type: $type, description: $desc}')
  
  impacts+=("$impact_json")
  
  echo "  ✅ Impact recorded: $fg ($priority, $type)"
done
```

---

### Step 5: Confirm and Apply

```yaml
step: confirm_and_apply
inputs:
  - impacts: object[]
  - impact_matrix: string (file path)
outputs:
  - applied_impacts: object[]
```

**Actions:**
1. Display summary for confirmation
2. Add impacts to impact-matrix.md
3. Update markdown tables for human readability

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Impact Summary"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

for impact in "${impacts[@]}"; do
  fg=$(echo "$impact" | jq -r '.focus_group')
  priority=$(echo "$impact" | jq -r '.priority')
  type=$(echo "$impact" | jq -r '.type')
  desc=$(echo "$impact" | jq -r '.description')
  
  echo ""
  echo "• $fg"
  echo "  Priority: $priority"
  echo "  Type: $type"
  echo "  Description: $desc"
done

echo ""
read -p "Apply these impacts? (y/n): " confirm

if [ "$confirm" != "y" ] && [ "$confirm" != "Y" ]; then
  echo "❌ Cancelled"
  exit 0
fi

# Apply impacts
echo ""
echo "📝 Applying impacts..."

for impact in "${impacts[@]}"; do
  fg=$(echo "$impact" | jq -r '.focus_group')
  priority=$(echo "$impact" | jq -r '.priority')
  type=$(echo "$impact" | jq -r '.type')
  desc=$(echo "$impact" | jq -r '.description')
  
  # Add to impact matrix
  impact_add "$impact_matrix" "$fg" "$priority" "$type" "$desc"
done

echo ""
echo "✅ Impacts applied to $impact_matrix"
```

---

### Step 6: Send Notifications

```yaml
step: send_notifications
inputs:
  - impacts: object[]
  - workspace: string
  - concept_slug: string
```

**Actions:**
1. Send notification to each impacted focus group channel
2. Mark impacts as notified in impact-matrix.md

```bash
echo ""
echo "📤 Sending notifications to focus groups..."

notified_fgs=""

for impact in "${impacts[@]}"; do
  fg=$(echo "$impact" | jq -r '.focus_group')
  
  # Add status field for notification
  full_impact=$(echo "$impact" | jq '. + {status: "pending"}')
  
  if impact_notify_new "$workspace" "$concept_slug" "$full_impact"; then
    notified_fgs="$notified_fgs $fg"
  fi
done

# Mark as notified
if [ -n "$notified_fgs" ]; then
  impact_mark_notified "$impact_matrix" $notified_fgs
fi

echo ""
echo "✅ Notifications sent"
```

---

### Step 7: Display Next Steps

```yaml
step: display_next_steps
```

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🎯 Next Steps"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "1. Focus groups have been notified of this concept's impact"
echo "2. Each focus group should review and approve:"
echo "   /wgsd approve $concept_slug"
echo ""
echo "3. View impact status:"
echo "   /wgsd concept $concept_slug"
echo ""
echo "4. Update impacts later:"
echo "   /wgsd update-impact $concept_slug"
echo ""
echo "Impact matrix saved to: $impact_matrix"
```

---

## Non-Interactive Mode

For scripted usage, pass all parameters:

```bash
/wgsd declare-impact oauth-integration \
  --focus-group security --priority P0 --type breaking-change --description "Token flow changes" \
  --focus-group api --priority P1 --type api-change --description "New auth endpoints"
```

**Parser:**
```bash
# Parse flags for non-interactive mode
impacts=()
current_fg=""
current_priority=""
current_type=""
current_desc=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --focus-group)
      # Save previous impact if exists
      if [ -n "$current_fg" ]; then
        impacts+=("{\"focus_group\":\"$current_fg\",\"priority\":\"$current_priority\",\"type\":\"$current_type\",\"description\":\"$current_desc\"}")
      fi
      current_fg="$2"
      current_priority="P2"
      current_type="integration"
      current_desc=""
      shift 2
      ;;
    --priority)
      current_priority="$2"
      shift 2
      ;;
    --type)
      current_type="$2"
      shift 2
      ;;
    --description)
      current_desc="$2"
      shift 2
      ;;
    *)
      shift
      ;;
  esac
done

# Save last impact
if [ -n "$current_fg" ]; then
  impacts+=("{\"focus_group\":\"$current_fg\",\"priority\":\"$current_priority\",\"type\":\"$current_type\",\"description\":\"$current_desc\"}")
fi
```

---

## Integration with create-concept

Add option in `create-concept.md` Step 6:

```bash
echo ""
read -p "Declare cross-cutting impacts now? (y/n): " declare_impacts

if [ "$declare_impacts" = "y" ] || [ "$declare_impacts" = "Y" ]; then
  # Call declare-impact workflow
  source workflows/declare-impact.md
  # Pass concept_slug and workspace
fi
```

---

## Error Handling

| Error | Resolution |
|-------|------------|
| Concept not found | List available concepts, check path |
| No focus groups | Run /wgsd create-focus-group first |
| Impact already exists | Offer to update instead |
| Channel not found | Create focus group channel first |
| Notification failed | Log error, continue with others |

---

## Output Artifacts

1. **Updated impact-matrix.md** - New impacts in YAML frontmatter
2. **Slack notifications** - Sent to each impacted focus group channel
3. **Change log entry** - Appended to impact matrix

---

*Workflow created for WGSD Phase 11 - Cross-Cutting Impact System*
