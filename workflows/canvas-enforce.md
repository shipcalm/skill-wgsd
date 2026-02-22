---
name: wgsd:canvas-enforce
description: AI-only Canvas enforcement - prevent direct human edits
dependencies:
  - wgsd:lib:slack-api
  - wgsd:lib:canvas-api
  - wgsd:lib:canvas-registry
---

# Canvas Enforcement Workflow

Ensure WGSD canvases are AI-managed only. Detect and handle direct human edits.

---

## Enforcement Philosophy

WGSD canvases serve as **live dashboards** that reflect the source of truth in `.planning/` files. Allowing direct canvas edits would:

1. **Break bidirectional sync** - Changes would conflict with git files
2. **Cause data loss** - Next sync would overwrite manual edits
3. **Create confusion** - Team wouldn't know what's authoritative
4. **Bypass workflows** - Changes would skip concept/implementation processes

**Instead:** Users discuss in channels → AI processes conversations → AI updates canvases

---

## Enforcement Strategies

### Strategy 1: Education (Primary)

When a human attempts to edit a WGSD canvas:
1. Detect the edit attempt
2. Post a friendly message explaining the policy
3. Provide guidance on the correct workflow
4. Revert to the last known good state

### Strategy 2: Documentation (Always)

Every canvas includes footer text:
```
*This canvas is AI-managed. Discuss changes in the channel, not by editing here.*
```

### Strategy 3: Restoration (On Detection)

If changes are detected during sync:
1. Compare current canvas content with last sync checksum
2. If different and not from AI: restore from git state
3. Notify channel about the restoration

---

## Edit Detection

Detect canvas edits by comparing checksums.

```bash
# Check if canvas was edited since last sync
detect_canvas_edit() {
  local repo_path="$1"
  local canvas_key="$2"
  
  if ! registry_exists "$repo_path" "$canvas_key"; then
    return 1
  fi
  
  local canvas_info=$(registry_get "$repo_path" "$canvas_key")
  local canvas_id=$(echo "$canvas_info" | jq -r '.canvas_id')
  local expected_checksum=$(echo "$canvas_info" | jq -r '.content_checksum')
  
  # Get current canvas content via API
  # Note: Slack doesn't have a direct "get canvas content" endpoint
  # Detection relies on sync comparison
  
  echo "DETECTION: Comparing checksums for $canvas_key"
  echo "Expected: $expected_checksum"
  
  # During normal sync, if we rebuild content and it matches expected checksum
  # but the canvas was manually edited, we'd detect this during edit
  
  # For now, enforcement happens at sync time
  return 0
}
```

---

## Education Messages

Messages posted when explaining canvas policy.

```bash
# Post education message about canvas policy
post_canvas_policy_message() {
  local channel_id="$1"
  local canvas_name="$2"
  local user_id="${3:-}"
  
  local user_mention=""
  if [ -n "$user_id" ]; then
    user_mention="Hey <@$user_id>! "
  fi
  
  local message="${user_mention}📋 **About WGSD Canvases**

This canvas (\`$canvas_name\`) is **AI-managed** and automatically synced with our planning files in git.

**Why can't I edit it directly?**
- Changes would be overwritten on the next sync
- It could conflict with what's in git
- It bypasses our structured development workflow

**How to make changes:**
1. **Discuss in this channel** - Share your ideas, questions, or suggestions
2. **I'll process the conversation** - I pick up important decisions and updates
3. **The canvas updates automatically** - Changes flow through git to canvas

**Quick commands:**
- \`/wgsd create-concept <name>\` - Add a new concept idea
- \`/wgsd promote-concept <name>\` - Ready for implementation
- \`/wgsd sync-canvas\` - Force a sync if needed

Think of the canvas as a **dashboard**, not a document. The real work happens in conversations and git! 💬"
  
  # Post message to channel
  # Note: Use your Slack posting mechanism here
  echo "$message"
}
```

---

## Restoration Workflow

Restore canvas to last known good state.

```bash
# Restore canvas from git state
restore_canvas() {
  local repo_path="$1"
  local canvas_key="$2"
  local notify_channel="${3:-true}"
  
  echo "🔧 Restoring canvas: $canvas_key"
  
  if ! registry_exists "$repo_path" "$canvas_key"; then
    echo "ERROR: Canvas not found in registry"
    return 1
  fi
  
  local canvas_info=$(registry_get "$repo_path" "$canvas_key")
  local canvas_id=$(echo "$canvas_info" | jq -r '.canvas_id')
  local canvas_type=$(echo "$canvas_info" | jq -r '.type')
  local channel_id=$(echo "$canvas_info" | jq -r '.channel_id')
  local channel_name=$(echo "$canvas_info" | jq -r '.channel_name')
  
  local stub=$(jq -r '.project_stub' "$repo_path/.planning/CANVAS-REGISTRY.json")
  
  # Rebuild content from git state
  local content
  case "$canvas_type" in
    master-dashboard)
      local project_name=$(get_project_name "$repo_path")
      content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
      ;;
    implementation-dashboard)
      content=$(build_implementation_dashboard_content "$repo_path")
      ;;
    focus-group)
      local fg_name=$(echo "$canvas_info" | jq -r '.focus_group')
      content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
      ;;
    community-roadmap)
      local project_name=$(get_project_name "$repo_path")
      content=$(build_community_content "$repo_path" "$project_name")
      ;;
    *)
      echo "ERROR: Unknown canvas type: $canvas_type"
      return 1
      ;;
  esac
  
  # Update canvas
  if canvas_edit "$canvas_id" "$content"; then
    local checksum=$(canvas_get_checksum "$content")
    registry_update_sync "$repo_path" "$canvas_key" "$checksum"
    echo "✅ Canvas restored from git state"
    
    # Notify channel if requested
    if [ "$notify_channel" = "true" ]; then
      local notify_msg="🔄 Canvas restored to match git state. Remember: discuss changes in the channel, and I'll update the canvas!"
      # Post to channel
      echo "NOTIFY: $channel_id - $notify_msg"
    fi
    
    return 0
  else
    echo "ERROR: Failed to restore canvas"
    return 1
  fi
}
```

---

## Integrity Check

Periodic check for canvas drift.

```bash
# Check all canvases for drift from expected state
check_canvas_integrity() {
  local repo_path="$1"
  
  echo "🔍 Checking Canvas Integrity"
  echo "============================"
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "No canvas registry found"
    return 0
  fi
  
  local canvases=$(jq -r '.canvases | keys[]' "$registry_file")
  local issues=0
  
  for key in $canvases; do
    local canvas_info=$(jq -r --arg key "$key" '.canvases[$key]' "$registry_file")
    local canvas_type=$(echo "$canvas_info" | jq -r '.type')
    local expected_checksum=$(echo "$canvas_info" | jq -r '.content_checksum')
    local channel_name=$(echo "$canvas_info" | jq -r '.channel_name')
    local stub=$(jq -r '.project_stub' "$registry_file")
    
    # Rebuild expected content
    local expected_content
    case "$canvas_type" in
      master-dashboard)
        local project_name=$(get_project_name "$repo_path")
        expected_content=$(build_master_dashboard_content "$repo_path" "$project_name" "$stub")
        ;;
      implementation-dashboard)
        expected_content=$(build_implementation_dashboard_content "$repo_path")
        ;;
      focus-group)
        local fg_name=$(echo "$canvas_info" | jq -r '.focus_group')
        expected_content=$(build_focus_group_content "$repo_path" "$fg_name" "$channel_name")
        ;;
      community-roadmap)
        local project_name=$(get_project_name "$repo_path")
        expected_content=$(build_community_content "$repo_path" "$project_name")
        ;;
      *)
        continue
        ;;
    esac
    
    local current_checksum=$(canvas_get_checksum "$expected_content")
    
    if [ "$current_checksum" != "$expected_checksum" ]; then
      echo "⚠️  $key: Git state has changed since last sync"
      echo "   Expected: $expected_checksum"
      echo "   Current:  $current_checksum"
      issues=$((issues + 1))
    else
      echo "✅ $key: In sync"
    fi
  done
  
  echo ""
  if [ $issues -gt 0 ]; then
    echo "Found $issues canvases needing sync"
    echo "Run 'wgsd sync-canvas all' to update"
  else
    echo "All canvases in sync ✨"
  fi
  
  return $issues
}
```

---

## AI Edit Authorization

Verify that canvas edits come from AI, not humans.

```bash
# Log AI edit for audit trail
log_ai_edit() {
  local repo_path="$1"
  local canvas_key="$2"
  local edit_description="$3"
  
  local log_file="$repo_path/.planning/canvas-edit-log.json"
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  # Ensure log file exists
  if [ ! -f "$log_file" ]; then
    echo '{"edits":[]}' > "$log_file"
  fi
  
  # Append edit record
  local edit_record=$(jq -n \
    --arg key "$canvas_key" \
    --arg desc "$edit_description" \
    --arg time "$timestamp" \
    --arg agent "wgsd-ai" \
    '{
      canvas_key: $key,
      description: $desc,
      timestamp: $time,
      agent: $agent,
      source: "git-sync"
    }')
  
  jq --argjson edit "$edit_record" '.edits += [$edit]' "$log_file" > "$log_file.tmp"
  mv "$log_file.tmp" "$log_file"
  
  echo "📝 Edit logged: $canvas_key"
}
```

---

## User Guidance

Help users understand the proper workflow.

```bash
# Show proper workflow for canvas changes
show_canvas_workflow() {
  cat << 'GUIDE'
# 📋 WGSD Canvas Workflow

Canvases in WGSD are **live dashboards** that reflect your planning files.
They update automatically - you don't edit them directly!

## How Changes Flow

```
You discuss     →    AI processes    →    Git updates    →    Canvas syncs
in channel           conversation         .planning/          automatically
```

## Examples

### Adding a new concept idea:

1. **In the focus group channel**, say:
   "I think we need a feature for single sign-on integration"

2. **AI responds** with structured concept, asks for confirmation

3. **Once confirmed**, AI creates:
   - `.planning/focus-groups/security/concepts/sso-integration.md`
   - Updates the focus group canvas

### Promoting a concept to implementation:

1. **In the focus group channel**, say:
   "The SSO concept is ready - let's promote it"

2. **AI confirms** requirements are complete, assigns owner

3. **On approval**, AI:
   - Moves concept to implementation queue
   - Updates implementation dashboard
   - Creates implementation channel

## Quick Commands

| Command | What it does |
|---------|--------------|
| `/wgsd sync-canvas` | Force sync all canvases |
| `/wgsd create-concept <name>` | Create a new concept |
| `/wgsd promote-concept <name>` | Promote to implementation |
| `/wgsd status` | Show current project state |

## Why This Approach?

- **Single source of truth** - Git files are authoritative
- **Audit trail** - All changes tracked in git history
- **No conflicts** - No merge conflicts between canvas and git
- **Structured process** - Changes go through proper workflow

---

*Need help? Just ask in your project's dev channel!*
GUIDE
}
```

---

## Entry Points

```bash
main() {
  local command="$1"
  shift
  
  local repo_path="${WGSD_REPO_PATH:-$(pwd)}"
  
  case "$command" in
    check|integrity)
      check_canvas_integrity "$repo_path"
      ;;
    restore)
      local key="$1"
      [ -z "$key" ] && { echo "ERROR: Canvas key required"; return 1; }
      restore_canvas "$repo_path" "$key"
      ;;
    restore-all)
      local keys=$(jq -r '.canvases | keys[]' "$repo_path/.planning/CANVAS-REGISTRY.json" 2>/dev/null)
      for key in $keys; do
        restore_canvas "$repo_path" "$key" "false"
      done
      echo "✅ All canvases restored"
      ;;
    explain|help|workflow)
      show_canvas_workflow
      ;;
    policy)
      local channel_id="$1"
      local canvas_name="${2:-WGSD Canvas}"
      post_canvas_policy_message "$channel_id" "$canvas_name"
      ;;
    *)
      echo "Canvas Enforcement Commands:"
      echo ""
      echo "  check        - Check all canvases for drift"
      echo "  restore KEY  - Restore specific canvas from git"
      echo "  restore-all  - Restore all canvases from git"
      echo "  explain      - Show proper canvas workflow"
      echo "  policy       - Post policy message to channel"
      ;;
  esac
}
```

---

## Integration Points

### With Sync Workflow

After sync, log the edit:
```bash
# In canvas-sync.md after successful update
log_ai_edit "$repo_path" "$canvas_key" "Synced from git: $change_description"
```

### With Create Workflow

On canvas creation, include policy footer:
```bash
# All templates include:
*This canvas is AI-managed. Discuss changes in the channel, not by editing here.*
```

### With Heartbeat

Periodic integrity check:
```bash
# In HEARTBEAT.md or cron job
check_canvas_integrity "$repo_path"
```

---

*Workflow created for WGSD Phase 4*
