---
name: wgsd:canvas-sync
description: Bidirectional sync between Canvas and git planning files
dependencies:
  - wgsd:lib:slack-api
  - wgsd:lib:canvas-api
  - wgsd:lib:canvas-registry
  - wgsd:lib:canvas-templates
  - wgsd:lib:git-ops
---

# Canvas Sync Workflow

Synchronize Canvas content with .planning/ git files bidirectionally.

---

## Sync Architecture

```
                    ┌─────────────┐
                    │   Canvas    │
                    │   (Slack)   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       ┌─────────────┐          ┌─────────────┐
       │ Git → Canvas│          │ Canvas → Git│
       │  (Primary)  │          │ (AI-Only)   │
       └─────────────┘          └─────────────┘
              │                         │
              ▼                         ▼
       ┌─────────────┐          ┌─────────────┐
       │  .planning/ │          │  Validate   │
       │    Files    │◄────────►│  AI Author  │
       └─────────────┘          └─────────────┘
```

**Primary Direction:** Git → Canvas (planning files are source of truth)
**Secondary Direction:** Canvas → Git (only for AI-managed updates)

---

## Sync Triggers

| Trigger | Action | Direction |
|---------|--------|-----------|
| `wgsd sync-canvas` | Full sync all canvases | Git → Canvas |
| `wgsd update-canvas [type]` | Update specific canvas | Git → Canvas |
| Focus group created | Create and sync FG canvas | Git → Canvas |
| Concept state changed | Update FG canvas | Git → Canvas |
| Implementation created | Update dashboards | Git → Canvas |
| Implementation completed | Update all dashboards | Git → Canvas |

---

## Workflow: Sync All Canvases

Synchronize all registered canvases from git state.

```
╔════════════════════════════════════════════════════════════════════════╗
║                       SYNC ALL CANVASES                                 ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                         ║
║  1. Load Registry                                                       ║
║     ├── Read CANVAS-REGISTRY.json                                      ║
║     └── List all registered canvases                                   ║
║                                                                         ║
║  2. For Each Canvas                                                     ║
║     ├── Build current content from .planning/ files                    ║
║     ├── Compare checksum with last sync                                ║
║     ├── Skip if unchanged                                              ║
║     └── Update canvas via API if changed                               ║
║                                                                         ║
║  3. Update Registry                                                     ║
║     ├── Record new checksums                                           ║
║     └── Update last_sync timestamps                                    ║
║                                                                         ║
║  4. Report Results                                                      ║
║     ├── Show updated canvases                                          ║
║     ├── Show skipped (unchanged)                                       ║
║     └── Show errors if any                                             ║
║                                                                         ║
╚════════════════════════════════════════════════════════════════════════╝
```

### Implementation

```bash
sync_all_canvases() {
  local repo_path="$1"
  
  echo "🔄 Syncing All Canvases"
  echo "========================"
  echo ""
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Canvas registry not found"
    echo "Run 'wgsd create-canvas all' first"
    return 1
  fi
  
  local stub=$(jq -r '.project_stub' "$registry_file")
  local canvases=$(jq -r '.canvases | keys[]' "$registry_file")
  
  local updated=0
  local skipped=0
  local errors=0
  
  for key in $canvases; do
    echo "📋 Syncing: $key"
    
    local canvas_info=$(jq -r --arg key "$key" '.canvases[$key]' "$registry_file")
    local canvas_id=$(echo "$canvas_info" | jq -r '.canvas_id')
    local canvas_type=$(echo "$canvas_info" | jq -r '.type')
    local old_checksum=$(echo "$canvas_info" | jq -r '.content_checksum')
    
    # Build new content based on type
    local new_content
    case "$canvas_type" in
      master-dashboard)
        local project_name=$(get_project_name "$repo_path")
        new_content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
        ;;
      implementation-dashboard)
        new_content=$(build_implementation_dashboard_content "$repo_path")
        ;;
      focus-group)
        local fg_name=$(echo "$canvas_info" | jq -r '.focus_group')
        local channel_name=$(echo "$canvas_info" | jq -r '.channel_name')
        new_content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
        ;;
      community-roadmap)
        local project_name=$(get_project_name "$repo_path")
        new_content=$(build_community_content "$repo_path" "$project_name")
        ;;
      *)
        echo "   ⚠️  Unknown type: $canvas_type (skipping)"
        skipped=$((skipped + 1))
        continue
        ;;
    esac
    
    # Check if changed
    local new_checksum=$(canvas_get_checksum "$new_content")
    
    if [ "$new_checksum" = "$old_checksum" ]; then
      echo "   ⏭️  No changes (skipped)"
      skipped=$((skipped + 1))
      continue
    fi
    
    # Update canvas
    if canvas_edit "$canvas_id" "$new_content"; then
      echo "   ✅ Updated"
      registry_update_sync "$repo_path" "$key" "$new_checksum"
      updated=$((updated + 1))
    else
      echo "   ❌ Failed to update"
      errors=$((errors + 1))
    fi
  done
  
  echo ""
  echo "📊 Sync Complete"
  echo "   Updated: $updated"
  echo "   Skipped: $skipped"
  echo "   Errors: $errors"
  
  return $errors
}

# Helper to get project name
get_project_name() {
  local repo_path="$1"
  local project_file="$repo_path/.planning/PROJECT.md"
  
  if [ -f "$project_file" ]; then
    grep -m1 "^#" "$project_file" | sed 's/^#\s*//'
  else
    basename "$repo_path"
  fi
}
```

---

## Workflow: Sync Focus Group

Sync a specific focus group canvas.

```bash
sync_focus_group() {
  local repo_path="$1"
  local fg_name="$2"
  
  echo "🎯 Syncing Focus Group: $fg_name"
  
  local registry_key="fg-$fg_name"
  
  if ! registry_exists "$repo_path" "$registry_key"; then
    echo "ERROR: Canvas not found for focus group: $fg_name"
    echo "Create it with: wgsd create-canvas focus-group $fg_name"
    return 1
  fi
  
  local canvas_id=$(registry_get_canvas_id "$repo_path" "$registry_key")
  local canvas_info=$(registry_get "$repo_path" "$registry_key")
  local channel_name=$(echo "$canvas_info" | jq -r '.channel_name')
  
  # Build content
  local content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
  local checksum=$(canvas_get_checksum "$content")
  
  # Update
  if canvas_edit "$canvas_id" "$content"; then
    registry_update_sync "$repo_path" "$registry_key" "$checksum"
    echo "✅ Focus group canvas synced"
    return 0
  else
    echo "❌ Failed to sync"
    return 1
  fi
}
```

---

## Workflow: Sync on State Change

Auto-sync canvases when project state changes.

```bash
# Called after state-changing operations
sync_on_state_change() {
  local repo_path="$1"
  local change_type="$2"  # "concept", "implementation", "focus-group"
  local change_target="$3"  # specific item that changed
  
  echo "🔄 Auto-syncing after $change_type change..."
  
  case "$change_type" in
    concept)
      # Sync the focus group canvas that contains this concept
      local fg_name="$change_target"
      sync_focus_group "$repo_path" "$fg_name"
      # Also update master dashboard
      sync_canvas_by_key "$repo_path" "master-dashboard"
      ;;
      
    implementation)
      # Sync implementation dashboard
      sync_canvas_by_key "$repo_path" "implementation-dashboard"
      # Also update master dashboard
      sync_canvas_by_key "$repo_path" "master-dashboard"
      ;;
      
    focus-group)
      # Sync master dashboard
      sync_canvas_by_key "$repo_path" "master-dashboard"
      # Sync community roadmap (if exists)
      if registry_exists "$repo_path" "community-roadmap"; then
        sync_canvas_by_key "$repo_path" "community-roadmap"
      fi
      ;;
      
    *)
      # Full sync for unknown change types
      sync_all_canvases "$repo_path"
      ;;
  esac
}

# Sync a specific canvas by registry key
sync_canvas_by_key() {
  local repo_path="$1"
  local key="$2"
  
  if ! registry_exists "$repo_path" "$key"; then
    return 0  # Canvas doesn't exist, nothing to sync
  fi
  
  local canvas_info=$(registry_get "$repo_path" "$key")
  local canvas_id=$(echo "$canvas_info" | jq -r '.canvas_id')
  local canvas_type=$(echo "$canvas_info" | jq -r '.type')
  local stub=$(jq -r '.project_stub' "$repo_path/.planning/CANVAS-REGISTRY.json")
  
  local content
  case "$canvas_type" in
    master-dashboard)
      local project_name=$(get_project_name "$repo_path")
      content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
      ;;
    implementation-dashboard)
      content=$(build_implementation_dashboard_content "$repo_path")
      ;;
    community-roadmap)
      local project_name=$(get_project_name "$repo_path")
      content=$(build_community_content "$repo_path" "$project_name")
      ;;
    *)
      echo "Unknown type for auto-sync: $canvas_type"
      return 1
      ;;
  esac
  
  local checksum=$(canvas_get_checksum "$content")
  
  if canvas_edit "$canvas_id" "$content"; then
    registry_update_sync "$repo_path" "$key" "$checksum"
    echo "   ✅ Synced: $key"
  fi
}
```

---

## Live Roadmap Visualization

Build cross-focus-group roadmap view.

```bash
build_live_roadmap() {
  local repo_path="$1"
  
  local fg_dir="$repo_path/.planning/focus-groups"
  local output=""
  
  output+="# 🗺️ Live Development Roadmap\n\n"
  output+="---\n\n"
  
  # Active implementations first
  output+="## 🚀 In Flight\n\n"
  local impl_dir="$repo_path/.planning/active-implementations"
  
  if [ -d "$impl_dir" ]; then
    for impl in "$impl_dir"/*/; do
      [ -d "$impl" ] || continue
      local name=$(basename "$impl")
      local state_file="$impl/STATE.md"
      local status="🔄"
      local owner=""
      
      if [ -f "$state_file" ]; then
        local status_line=$(grep -m1 "^Status:" "$state_file" 2>/dev/null || true)
        local owner_line=$(grep -m1 "^Owner:" "$state_file" 2>/dev/null || true)
        [ -n "$owner_line" ] && owner=" ($(echo "$owner_line" | cut -d':' -f2 | xargs))"
      fi
      
      output+="- $status **$name**$owner\n"
    done
  else
    output+="*No active implementations*\n"
  fi
  
  output+="\n---\n\n"
  
  # Focus groups with concepts
  output+="## 🎯 Focus Areas\n\n"
  
  if [ -d "$fg_dir" ]; then
    for fg in "$fg_dir"/*/; do
      [ -d "$fg" ] || continue
      local fg_name=$(basename "$fg")
      local display_name=$(echo "$fg_name" | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
      
      output+="### $display_name\n\n"
      
      # List concepts with status
      local concepts_dir="$fg/concepts"
      if [ -d "$concepts_dir" ]; then
        for concept_file in "$concepts_dir"/*.md; do
          [ -f "$concept_file" ] || continue
          local concept_name=$(basename "$concept_file" .md)
          local status=$(grep -m1 "^Status:" "$concept_file" 2>/dev/null | cut -d':' -f2 | xargs || echo "proposed")
          
          local emoji="📝"
          case "$status" in
            *ready*|*Ready*) emoji="✅" ;;
            *active*|*Active*|*development*) emoji="🔄" ;;
            *proposed*|*draft*) emoji="📝" ;;
          esac
          
          output+="- $emoji $concept_name\n"
        done
      else
        output+="*No concepts yet*\n"
      fi
      
      output+="\n"
    done
  else
    output+="*No focus groups defined*\n"
  fi
  
  output+="\n---\n\n"
  output+="*Live roadmap - updates automatically with project state*\n"
  output+="*Last updated: $(date -u +"%Y-%m-%d %H:%M UTC")*\n"
  
  echo -e "$output"
}
```

---

## Bidirectional Sync: Canvas → Git (AI-Only)

This direction is restricted to AI-managed updates only.

```bash
# This function is called when AI processes a conversation that should update planning
sync_canvas_to_git() {
  local repo_path="$1"
  local canvas_key="$2"
  local new_content="$3"
  local change_description="$4"
  
  echo "📝 Syncing Canvas → Git"
  echo "Key: $canvas_key"
  echo "Change: $change_description"
  
  # IMPORTANT: This should only be called by AI processing, never by direct canvas edits
  # The AI reads conversations, decides what to update, and calls this function
  
  case "$canvas_key" in
    fg-*)
      local fg_name=$(echo "$canvas_key" | sed 's/^fg-//')
      sync_fg_canvas_to_git "$repo_path" "$fg_name" "$new_content" "$change_description"
      ;;
    *)
      echo "WARNING: Canvas → Git sync not supported for type: $canvas_key"
      echo "Only focus group canvases sync back to git"
      return 1
      ;;
  esac
}

# Sync focus group canvas changes to git
sync_fg_canvas_to_git() {
  local repo_path="$1"
  local fg_name="$2"
  local new_content="$3"
  local description="$4"
  
  local fg_dir="$repo_path/.planning/focus-groups/$fg_name"
  
  if [ ! -d "$fg_dir" ]; then
    echo "ERROR: Focus group directory not found"
    return 1
  fi
  
  # Parse content to extract changes
  # This is where AI-parsed content gets written to git files
  
  # For now, update the STATE.md with a note about the change
  local state_file="$fg_dir/STATE.md"
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  
  # Append change note
  echo "" >> "$state_file"
  echo "## Recent Update ($timestamp)" >> "$state_file"
  echo "$description" >> "$state_file"
  
  # Commit the change
  cd "$repo_path"
  git add ".planning/focus-groups/$fg_name/"
  git commit -m "Update $fg_name focus group: $description" || true
  
  echo "✅ Git files updated"
  
  # Now sync back to canvas to ensure consistency
  sync_focus_group "$repo_path" "$fg_name"
}
```

---

## Integration with Other Workflows

### Hook into create-focus-group

After focus group creation, auto-create and sync canvas:

```bash
# Add to end of create-focus-group workflow
post_create_focus_group() {
  local repo_path="$1"
  local fg_name="$2"
  local stub="$3"
  
  # Create focus group canvas
  create_focus_group_canvas "$repo_path" "$fg_name" "$stub"
  
  # Trigger state change sync for master dashboard
  sync_on_state_change "$repo_path" "focus-group" "$fg_name"
}
```

### Hook into promote-concept

After concept promotion, update dashboards:

```bash
# Add to end of promote-concept workflow
post_promote_concept() {
  local repo_path="$1"
  local concept_name="$2"
  local fg_name="$3"
  
  # Update focus group canvas
  sync_focus_group "$repo_path" "$fg_name"
  
  # Update implementation dashboard
  sync_on_state_change "$repo_path" "implementation" "$concept_name"
}
```

### Hook into complete-implementation

After implementation completion, update all dashboards:

```bash
# Add to end of complete-implementation workflow
post_complete_implementation() {
  local repo_path="$1"
  local impl_name="$2"
  
  # Full dashboard update
  sync_on_state_change "$repo_path" "implementation" "$impl_name"
  
  # Update community roadmap if exists
  if registry_exists "$repo_path" "community-roadmap"; then
    sync_canvas_by_key "$repo_path" "community-roadmap"
  fi
}
```

---

## Entry Point

```bash
main() {
  local command="$1"
  shift
  
  local repo_path="${WGSD_REPO_PATH:-$(pwd)}"
  
  case "$command" in
    all|sync)
      sync_all_canvases "$repo_path"
      ;;
    focus-group|fg)
      local fg_name="$1"
      [ -z "$fg_name" ] && { echo "ERROR: Focus group name required"; return 1; }
      sync_focus_group "$repo_path" "$fg_name"
      ;;
    roadmap)
      build_live_roadmap "$repo_path"
      ;;
    master|dashboard)
      sync_canvas_by_key "$repo_path" "master-dashboard"
      ;;
    implementation|impl)
      sync_canvas_by_key "$repo_path" "implementation-dashboard"
      ;;
    community)
      sync_canvas_by_key "$repo_path" "community-roadmap"
      ;;
    *)
      echo "Unknown sync command: $command"
      echo ""
      echo "Usage:"
      echo "  wgsd sync-canvas all         - Sync all canvases"
      echo "  wgsd sync-canvas fg <name>   - Sync specific focus group"
      echo "  wgsd sync-canvas master      - Sync master dashboard"
      echo "  wgsd sync-canvas impl        - Sync implementation dashboard"
      echo "  wgsd sync-canvas community   - Sync community roadmap"
      echo "  wgsd sync-canvas roadmap     - Generate live roadmap"
      return 1
      ;;
  esac
}
```

---

## Automatic Sync Schedule

For production use, sync can be triggered automatically:

| Trigger | Frequency | Canvases |
|---------|-----------|----------|
| Git push to .planning/ | On event | All changed |
| Heartbeat check | Every 30 min | Check for drift |
| Manual sync | On demand | Selected or all |
| State change | Immediate | Related canvases |

---

*Workflow created for WGSD Phase 4*
