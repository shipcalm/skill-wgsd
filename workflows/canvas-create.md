---
name: wgsd:canvas-create
description: Create WGSD canvases for channels
dependencies:
  - wgsd:lib:slack-api
  - wgsd:lib:canvas-api
  - wgsd:lib:canvas-registry
  - wgsd:lib:canvas-templates
---

# Canvas Creation Workflow

Create and initialize WGSD canvases for different channel types.

---

## Trigger Conditions

| Trigger | Canvas Created |
|---------|----------------|
| `wgsd create-canvas master` | Master dashboard in dev channel |
| `wgsd create-canvas implementation` | Implementation dashboard |
| `wgsd create-canvas focus-group [name]` | Focus group canvas |
| `wgsd create-canvas community` | Community roadmap |
| `wgsd setup-canvases` | All core canvases |

---

## Input Requirements

### Required
- **repo_path**: Path to WGSD repository
- **project_stub**: Project stub (e.g., "mvn")

### Optional
- **project_name**: Human-readable project name (defaults from PROJECT.md)
- **focus_group**: Focus group name (for FG canvas)

---

## Workflow: Create Master Dashboard

Create the master dashboard canvas in the dev channel.

```
╔════════════════════════════════════════════════════════════════════════╗
║                     CREATE MASTER DASHBOARD                             ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                         ║
║  1. Load Configuration                                                  ║
║     ├── Read WGSD-CONFIG.md for stub and dev channel                   ║
║     ├── Get project name from PROJECT.md                               ║
║     └── Check if master dashboard already exists in registry           ║
║                                                                         ║
║  2. Build Dashboard Content                                             ║
║     ├── Load master dashboard template                                  ║
║     ├── Aggregate focus group state from .planning/                    ║
║     ├── Aggregate implementation state                                  ║
║     └── Render template with current state                             ║
║                                                                         ║
║  3. Create Canvas                                                       ║
║     ├── Find dev channel ID (from registry or API)                     ║
║     ├── Call conversations.canvases.create                             ║
║     └── Handle existing canvas (update vs create)                      ║
║                                                                         ║
║  4. Register Canvas                                                     ║
║     ├── Add to CANVAS-REGISTRY.json                                    ║
║     ├── Record canvas ID, channel, checksum                            ║
║     └── Commit registry update                                         ║
║                                                                         ║
║  5. Announce Creation                                                   ║
║     └── Post message in dev channel about new dashboard                ║
║                                                                         ║
╚════════════════════════════════════════════════════════════════════════╝
```

### Implementation

```bash
create_master_dashboard() {
  local repo_path="$1"
  local stub="$2"
  local project_name="${3:-}"
  
  echo "🎯 Creating Master Dashboard Canvas"
  echo "===================================="
  
  # 1. Load configuration
  local config_file="$repo_path/.planning/WGSD-CONFIG.md"
  
  if [ ! -f "$config_file" ]; then
    echo "ERROR: WGSD-CONFIG.md not found. Run 'wgsd init' first."
    return 1
  fi
  
  # Get project name if not provided
  if [ -z "$project_name" ]; then
    local project_file="$repo_path/.planning/PROJECT.md"
    if [ -f "$project_file" ]; then
      project_name=$(grep -m1 "^#" "$project_file" | sed 's/^#\s*//')
    fi
    project_name="${project_name:-$stub Project}"
  fi
  
  echo "📁 Repository: $repo_path"
  echo "📝 Project: $project_name ($stub)"
  
  # Check if already exists
  if registry_exists "$repo_path" "master-dashboard"; then
    echo "⚠️  Master dashboard already exists"
    read -p "Update existing canvas? [y/N]: " update
    if [ "$update" != "y" ] && [ "$update" != "Y" ]; then
      echo "Cancelled."
      return 0
    fi
    
    # Update existing
    local canvas_id=$(registry_get_canvas_id "$repo_path" "master-dashboard")
    local content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
    canvas_edit "$canvas_id" "$content"
    registry_update_sync "$repo_path" "master-dashboard" "$(canvas_get_checksum "$content")"
    echo "✅ Master dashboard updated"
    return 0
  fi
  
  # 2. Build content
  echo "📊 Building dashboard content..."
  local content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
  
  # 3. Find or create dev channel
  local dev_channel="${stub}-dev"
  local channel_id=$(slack_find_channel "$dev_channel")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Dev channel #$dev_channel not found"
    echo "Run 'wgsd setup-core-channels' first"
    return 1
  fi
  
  echo "📺 Channel: #$dev_channel ($channel_id)"
  
  # 4. Create canvas
  local title="${project_name} Development Dashboard"
  local result=$(canvas_create_attached "$channel_id" "$title" "$content")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to create canvas"
    echo "$result"
    return 1
  fi
  
  # Extract canvas ID from result
  local canvas_id=$(echo "$result" | grep "CANVAS_ID:" | cut -d':' -f2)
  
  if [ -z "$canvas_id" ]; then
    echo "ERROR: Could not extract canvas ID"
    return 1
  fi
  
  # 5. Register canvas
  registry_add "$repo_path" "master-dashboard" "$canvas_id" "$channel_id" "$dev_channel" "master-dashboard" "$title"
  
  local checksum=$(canvas_get_checksum "$content")
  registry_update_sync "$repo_path" "master-dashboard" "$checksum"
  
  echo ""
  echo "✅ Master Dashboard Created!"
  echo "   Canvas ID: $canvas_id"
  echo "   Channel: #$dev_channel"
  echo ""
  echo "View it by clicking the canvas icon in #$dev_channel"
  
  return 0
}
```

---

## Workflow: Create Implementation Dashboard

Create implementation tracking dashboard in dev channel.

```bash
create_implementation_dashboard() {
  local repo_path="$1"
  local stub="$2"
  local project_name="${3:-}"
  
  echo "🚀 Creating Implementation Dashboard"
  echo "====================================="
  
  # Get project name
  if [ -z "$project_name" ]; then
    local project_file="$repo_path/.planning/PROJECT.md"
    if [ -f "$project_file" ]; then
      project_name=$(grep -m1 "^#" "$project_file" | sed 's/^#\s*//')
    fi
    project_name="${project_name:-$stub Project}"
  fi
  
  # Check if already exists
  if registry_exists "$repo_path" "implementation-dashboard"; then
    echo "⚠️  Implementation dashboard already exists"
    local canvas_id=$(registry_get_canvas_id "$repo_path" "implementation-dashboard")
    local content=$(build_implementation_dashboard_content "$repo_path")
    canvas_edit "$canvas_id" "$content"
    registry_update_sync "$repo_path" "implementation-dashboard"
    echo "✅ Implementation dashboard updated"
    return 0
  fi
  
  # Build content
  echo "📊 Building implementation dashboard..."
  local content=$(build_implementation_dashboard_content "$repo_path")
  
  # Find dev channel
  local dev_channel="${stub}-dev"
  local channel_id=$(slack_find_channel "$dev_channel")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Dev channel not found"
    return 1
  fi
  
  # Create as standalone canvas (since channel can only have one attached canvas)
  local title="${project_name} Implementation Dashboard"
  local result=$(canvas_create_standalone "$title" "$content")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to create canvas"
    return 1
  fi
  
  local canvas_id=$(echo "$result" | grep "CANVAS_ID:" | cut -d':' -f2)
  
  # Share to dev channel
  canvas_share_to_channel "$canvas_id" "$channel_id" "read"
  
  # Register
  registry_add "$repo_path" "implementation-dashboard" "$canvas_id" "$channel_id" "$dev_channel" "implementation-dashboard" "$title"
  
  echo "✅ Implementation Dashboard Created!"
  echo "   Canvas ID: $canvas_id"
  
  return 0
}

# Build implementation dashboard content
build_implementation_dashboard_content() {
  local repo_path="$1"
  
  local template=$(template_implementation_dashboard)
  
  # Count implementations
  local impl_dir="$repo_path/.planning/active-implementations"
  local active_count=0
  local max_count=4
  
  if [ -d "$impl_dir" ]; then
    active_count=$(ls -1d "$impl_dir"/*/ 2>/dev/null | wc -l)
  fi
  
  local implementations=$(format_implementations_list "$repo_path")
  local queue=$(format_queue_list "$repo_path")
  local completed="*No completions yet*"
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  
  local vars=$(jq -n \
    --arg active_count "$active_count" \
    --arg max_count "$max_count" \
    --arg active "$implementations" \
    --arg queue "$queue" \
    --arg completed "$completed" \
    --arg timestamp "$timestamp" \
    '{
      ACTIVE_COUNT: $active_count,
      MAX_COUNT: $max_count,
      ACTIVE_IMPLEMENTATIONS: $active,
      QUEUED_IMPLEMENTATIONS: $queue,
      COMPLETED_IMPLEMENTATIONS: $completed,
      AVG_TIME: "~2 days",
      WEEKLY_COUNT: "0",
      QUEUE_COUNT: "0",
      TIMESTAMP: $timestamp
    }')
  
  template_render "$template" "$vars"
}
```

---

## Workflow: Create Focus Group Canvas

Create canvas for a specific focus group.

```bash
create_focus_group_canvas() {
  local repo_path="$1"
  local fg_name="$2"
  local stub="$3"
  
  echo "🎯 Creating Focus Group Canvas: $fg_name"
  echo "========================================="
  
  local fg_dir="$repo_path/.planning/focus-groups/$fg_name"
  
  if [ ! -d "$fg_dir" ]; then
    echo "ERROR: Focus group '$fg_name' not found"
    echo "Path: $fg_dir"
    return 1
  fi
  
  local registry_key="fg-$fg_name"
  
  # Check if already exists
  if registry_exists "$repo_path" "$registry_key"; then
    echo "⚠️  Focus group canvas already exists"
    local canvas_id=$(registry_get_canvas_id "$repo_path" "$registry_key")
    local channel_name="${stub}-fg-${fg_name}"
    local content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
    canvas_edit "$canvas_id" "$content"
    registry_update_sync "$repo_path" "$registry_key"
    echo "✅ Focus group canvas updated"
    return 0
  fi
  
  # Find channel
  local channel_name="${stub}-fg-${fg_name}"
  local channel_id=$(slack_find_channel "$channel_name")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Focus group channel #$channel_name not found"
    return 1
  fi
  
  # Build content
  local content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
  
  # Create attached canvas
  local title="${fg_name} Focus Group"
  local result=$(canvas_create_attached "$channel_id" "$title" "$content")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to create canvas"
    return 1
  fi
  
  local canvas_id=$(echo "$result" | grep "CANVAS_ID:" | cut -d':' -f2)
  
  # Register with focus_group metadata
  local extra=$(jq -n --arg fg "$fg_name" '{focus_group: $fg}')
  registry_add "$repo_path" "$registry_key" "$canvas_id" "$channel_id" "$channel_name" "focus-group" "$title" "$extra"
  
  echo "✅ Focus Group Canvas Created!"
  echo "   Focus Group: $fg_name"
  echo "   Canvas ID: $canvas_id"
  echo "   Channel: #$channel_name"
  
  return 0
}
```

---

## Workflow: Create Community Canvas

Create public community roadmap canvas.

```bash
create_community_canvas() {
  local repo_path="$1"
  local stub="$2"
  local project_name="${3:-}"
  
  echo "💬 Creating Community Roadmap Canvas"
  echo "====================================="
  
  # Get project name
  if [ -z "$project_name" ]; then
    local project_file="$repo_path/.planning/PROJECT.md"
    if [ -f "$project_file" ]; then
      project_name=$(grep -m1 "^#" "$project_file" | sed 's/^#\s*//')
    fi
    project_name="${project_name:-$stub Project}"
  fi
  
  # Check if exists
  if registry_exists "$repo_path" "community-roadmap"; then
    echo "⚠️  Community canvas already exists"
    local canvas_id=$(registry_get_canvas_id "$repo_path" "community-roadmap")
    local content=$(build_community_content "$repo_path" "$project_name")
    canvas_edit "$canvas_id" "$content"
    registry_update_sync "$repo_path" "community-roadmap"
    echo "✅ Community canvas updated"
    return 0
  fi
  
  # Find community channel
  local channel_name="${stub}-community"
  local channel_id=$(slack_find_channel "$channel_name")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Community channel #$channel_name not found"
    return 1
  fi
  
  # Build sanitized content
  local content=$(build_community_content "$repo_path" "$project_name")
  
  # Create canvas
  local title="${project_name} Public Roadmap"
  local result=$(canvas_create_attached "$channel_id" "$title" "$content")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to create canvas"
    return 1
  fi
  
  local canvas_id=$(echo "$result" | grep "CANVAS_ID:" | cut -d':' -f2)
  
  # Register
  registry_add "$repo_path" "community-roadmap" "$canvas_id" "$channel_id" "$channel_name" "community-roadmap" "$title"
  
  echo "✅ Community Roadmap Created!"
  echo "   Canvas ID: $canvas_id"
  echo "   Channel: #$channel_name"
  
  return 0
}

# Build sanitized community content
build_community_content() {
  local repo_path="$1"
  local project_name="$2"
  
  local template=$(template_community_roadmap)
  
  # Get public-safe focus areas (just names, no internal details)
  local fg_dir="$repo_path/.planning/focus-groups"
  local focus_areas=""
  
  if [ -d "$fg_dir" ]; then
    for fg in "$fg_dir"/*/; do
      [ -d "$fg" ] || continue
      local name=$(basename "$fg")
      # Format nicely
      local display_name=$(echo "$name" | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
      focus_areas="${focus_areas}- **$display_name**\n"
    done
  fi
  
  [ -z "$focus_areas" ] && focus_areas="*Coming soon*"
  
  # Get project summary if exists
  local summary="Building amazing features for our users!"
  local project_file="$repo_path/.planning/PROJECT.md"
  if [ -f "$project_file" ]; then
    summary=$(sed -n '/^##.*Overview/,/^##/p' "$project_file" | head -10 | tail -n +2 | xargs)
    [ -z "$summary" ] && summary="Building amazing features for our users!"
  fi
  
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  
  local vars=$(jq -n \
    --arg project "$project_name" \
    --arg summary "$summary" \
    --arg focus "$focus_areas" \
    --arg timestamp "$timestamp" \
    '{
      PROJECT: $project,
      PROJECT_SUMMARY: $summary,
      PUBLIC_FOCUS_AREAS: $focus,
      IN_PROGRESS: "*Updates coming soon*",
      RECENTLY_SHIPPED: "*Check back for updates*",
      FOCUS_COUNT: "0",
      ACTIVE_COUNT: "0",
      MONTHLY_COUNT: "0",
      TIMESTAMP: $timestamp
    }')
  
  template_render "$template" "$vars"
}
```

---

## Workflow: Setup All Canvases

Create all core canvases for a project.

```bash
setup_all_canvases() {
  local repo_path="$1"
  local stub="$2"
  local project_name="$3"
  
  echo "📋 Setting Up All WGSD Canvases"
  echo "================================"
  echo ""
  
  # Initialize registry if needed
  registry_init "$repo_path" "$stub"
  
  # 1. Master Dashboard
  echo "1/4 Creating Master Dashboard..."
  create_master_dashboard "$repo_path" "$stub" "$project_name"
  echo ""
  
  # 2. Implementation Dashboard
  echo "2/4 Creating Implementation Dashboard..."
  create_implementation_dashboard "$repo_path" "$stub" "$project_name"
  echo ""
  
  # 3. Community Roadmap
  echo "3/4 Creating Community Roadmap..."
  create_community_canvas "$repo_path" "$stub" "$project_name"
  echo ""
  
  # 4. Focus Group Canvases
  echo "4/4 Creating Focus Group Canvases..."
  local fg_dir="$repo_path/.planning/focus-groups"
  
  if [ -d "$fg_dir" ]; then
    for fg in "$fg_dir"/*/; do
      [ -d "$fg" ] || continue
      local fg_name=$(basename "$fg")
      echo "     Creating canvas for: $fg_name"
      create_focus_group_canvas "$repo_path" "$fg_name" "$stub"
    done
  else
    echo "     No focus groups found"
  fi
  
  echo ""
  echo "✅ All Canvases Created!"
  echo ""
  registry_list "$repo_path"
}
```

---

## Entry Point

Route canvas creation based on command.

```bash
# Parse command and route
main() {
  local command="$1"
  shift
  
  # Get repo path from workspace or argument
  local repo_path="${WGSD_REPO_PATH:-$(pwd)}"
  
  # Get stub from WGSD-CONFIG.md
  local stub=$(grep "^stub:" "$repo_path/.planning/WGSD-CONFIG.md" 2>/dev/null | awk '{print $2}')
  
  case "$command" in
    master|master-dashboard)
      create_master_dashboard "$repo_path" "$stub" "$@"
      ;;
    implementation|impl-dashboard)
      create_implementation_dashboard "$repo_path" "$stub" "$@"
      ;;
    focus-group|fg)
      local fg_name="$1"
      if [ -z "$fg_name" ]; then
        echo "ERROR: Focus group name required"
        echo "Usage: wgsd create-canvas focus-group <name>"
        return 1
      fi
      create_focus_group_canvas "$repo_path" "$fg_name" "$stub"
      ;;
    community)
      create_community_canvas "$repo_path" "$stub" "$@"
      ;;
    all|setup)
      setup_all_canvases "$repo_path" "$stub" "$@"
      ;;
    *)
      echo "Unknown canvas type: $command"
      echo ""
      echo "Available types:"
      echo "  master        - Master development dashboard"
      echo "  implementation - Implementation tracking dashboard"
      echo "  focus-group   - Focus group canvas"
      echo "  community     - Public community roadmap"
      echo "  all           - All core canvases"
      return 1
      ;;
  esac
}
```

---

## Error Handling

| Error | Recovery |
|-------|----------|
| Channel not found | Run setup-core-channels first |
| Canvas already exists | Prompt for update or skip |
| API failure | Retry with exponential backoff |
| Registry missing | Initialize registry automatically |

---

*Workflow created for WGSD Phase 4*
