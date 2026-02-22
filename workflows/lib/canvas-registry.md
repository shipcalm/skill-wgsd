---
name: wgsd:lib:canvas-registry
description: Canvas registry for tracking WGSD canvases
---

# Canvas Registry Library

Track and manage WGSD canvases across channels.

---

## Registry Location

Canvas registry is stored in the project's planning directory:
```
{repo}/.planning/CANVAS-REGISTRY.json
```

---

## Registry Schema

```json
{
  "version": "1.0",
  "project_stub": "mvn",
  "last_updated": "2026-02-22T12:00:00Z",
  "canvases": {
    "master-dashboard": {
      "canvas_id": "F123ABC",
      "channel_id": "C123DEV",
      "channel_name": "mvn-dev",
      "type": "master-dashboard",
      "title": "Marvin Development Dashboard",
      "created_at": "2026-02-22T10:00:00Z",
      "last_sync": "2026-02-22T12:00:00Z",
      "content_checksum": "abc123def456"
    },
    "fg-security": {
      "canvas_id": "F456DEF",
      "channel_id": "C456FG",
      "channel_name": "mvn-fg-security",
      "type": "focus-group",
      "focus_group": "security",
      "title": "Security Focus Group",
      "created_at": "2026-02-22T11:00:00Z",
      "last_sync": "2026-02-22T12:00:00Z",
      "content_checksum": "def456ghi789"
    }
  }
}
```

---

## registry_init

Initialize a new canvas registry.

```bash
# Usage: registry_init <repo_path> <project_stub>
# Arguments:
#   repo_path    - Path to repository
#   project_stub - Project stub (e.g., "mvn")
# Returns: Success or error
registry_init() {
  local repo_path="$1"
  local stub="$2"
  
  if [ -z "$repo_path" ] || [ -z "$stub" ]; then
    echo "ERROR: Repository path and project stub are required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  # Check if already exists
  if [ -f "$registry_file" ]; then
    echo "⚠️  Registry already exists at $registry_file"
    return 0
  fi
  
  echo "📋 Initializing canvas registry for $stub"
  
  # Create initial registry
  local now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  local registry=$(jq -n \
    --arg version "1.0" \
    --arg stub "$stub" \
    --arg now "$now" \
    '{
      version: $version,
      project_stub: $stub,
      last_updated: $now,
      canvases: {}
    }')
  
  # Ensure .planning directory exists
  mkdir -p "$repo_path/.planning"
  
  echo "$registry" > "$registry_file"
  
  echo "✅ Registry initialized: $registry_file"
  return 0
}
```

---

## registry_add

Add a canvas to the registry.

```bash
# Usage: registry_add <repo_path> <key> <canvas_id> <channel_id> <channel_name> <type> <title> [extra_json]
# Arguments:
#   repo_path    - Path to repository
#   key          - Unique key for canvas (e.g., "master-dashboard", "fg-security")
#   canvas_id    - Slack canvas ID
#   channel_id   - Slack channel ID
#   channel_name - Channel name
#   type         - Canvas type
#   title        - Canvas title
#   extra_json   - Optional extra fields as JSON
# Returns: Success or error
registry_add() {
  local repo_path="$1"
  local key="$2"
  local canvas_id="$3"
  local channel_id="$4"
  local channel_name="$5"
  local type="$6"
  local title="$7"
  local extra_json="${8:-{}}"
  
  if [ -z "$repo_path" ] || [ -z "$key" ] || [ -z "$canvas_id" ]; then
    echo "ERROR: Required arguments missing"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found. Run registry_init first."
    return 1
  fi
  
  echo "➕ Adding canvas to registry: $key"
  
  local now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  # Build canvas entry
  local entry=$(jq -n \
    --arg canvas_id "$canvas_id" \
    --arg channel_id "$channel_id" \
    --arg channel_name "$channel_name" \
    --arg type "$type" \
    --arg title "$title" \
    --arg now "$now" \
    --argjson extra "$extra_json" \
    '{
      canvas_id: $canvas_id,
      channel_id: $channel_id,
      channel_name: $channel_name,
      type: $type,
      title: $title,
      created_at: $now,
      last_sync: $now,
      content_checksum: ""
    } + $extra')
  
  # Update registry
  local updated=$(jq \
    --arg key "$key" \
    --argjson entry "$entry" \
    --arg now "$now" \
    '.canvases[$key] = $entry | .last_updated = $now' \
    "$registry_file")
  
  echo "$updated" > "$registry_file"
  
  echo "✅ Canvas added: $key → $canvas_id"
  return 0
}
```

---

## registry_get

Get a canvas entry from registry.

```bash
# Usage: registry_get <repo_path> <key>
# Returns: Canvas JSON or error
registry_get() {
  local repo_path="$1"
  local key="$2"
  
  if [ -z "$repo_path" ] || [ -z "$key" ]; then
    echo "ERROR: Repository path and key are required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found"
    return 1
  fi
  
  local entry=$(jq -r --arg key "$key" '.canvases[$key] // empty' "$registry_file")
  
  if [ -z "$entry" ]; then
    echo "ERROR: Canvas not found: $key"
    return 1
  fi
  
  echo "$entry"
  return 0
}
```

---

## registry_get_by_type

Get all canvases of a specific type.

```bash
# Usage: registry_get_by_type <repo_path> <type>
# Returns: Array of canvas entries
registry_get_by_type() {
  local repo_path="$1"
  local type="$2"
  
  if [ -z "$repo_path" ] || [ -z "$type" ]; then
    echo "ERROR: Repository path and type are required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found"
    return 1
  fi
  
  jq --arg type "$type" '[.canvases | to_entries[] | select(.value.type == $type)]' "$registry_file"
  return 0
}
```

---

## registry_update_sync

Update the last sync timestamp and checksum for a canvas.

```bash
# Usage: registry_update_sync <repo_path> <key> [checksum]
# Returns: Success or error
registry_update_sync() {
  local repo_path="$1"
  local key="$2"
  local checksum="${3:-}"
  
  if [ -z "$repo_path" ] || [ -z "$key" ]; then
    echo "ERROR: Repository path and key are required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found"
    return 1
  fi
  
  local now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  local updated
  if [ -n "$checksum" ]; then
    updated=$(jq \
      --arg key "$key" \
      --arg now "$now" \
      --arg checksum "$checksum" \
      '.canvases[$key].last_sync = $now | .canvases[$key].content_checksum = $checksum | .last_updated = $now' \
      "$registry_file")
  else
    updated=$(jq \
      --arg key "$key" \
      --arg now "$now" \
      '.canvases[$key].last_sync = $now | .last_updated = $now' \
      "$registry_file")
  fi
  
  echo "$updated" > "$registry_file"
  
  echo "✅ Sync updated: $key"
  return 0
}
```

---

## registry_remove

Remove a canvas from registry.

```bash
# Usage: registry_remove <repo_path> <key>
# Returns: Success or error
registry_remove() {
  local repo_path="$1"
  local key="$2"
  
  if [ -z "$repo_path" ] || [ -z "$key" ]; then
    echo "ERROR: Repository path and key are required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found"
    return 1
  fi
  
  echo "➖ Removing canvas from registry: $key"
  
  local now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  local updated=$(jq \
    --arg key "$key" \
    --arg now "$now" \
    'del(.canvases[$key]) | .last_updated = $now' \
    "$registry_file")
  
  echo "$updated" > "$registry_file"
  
  echo "✅ Canvas removed: $key"
  return 0
}
```

---

## registry_list

List all canvases in registry.

```bash
# Usage: registry_list <repo_path> [format]
# Arguments:
#   repo_path - Path to repository
#   format    - "json" or "table" (default: table)
# Returns: Canvas list
registry_list() {
  local repo_path="$1"
  local format="${2:-table}"
  
  if [ -z "$repo_path" ]; then
    echo "ERROR: Repository path is required"
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    echo "ERROR: Registry not found"
    return 1
  fi
  
  if [ "$format" = "json" ]; then
    jq '.canvases' "$registry_file"
    return 0
  fi
  
  # Table format
  echo "📋 Canvas Registry"
  echo "=================="
  
  local stub=$(jq -r '.project_stub' "$registry_file")
  local updated=$(jq -r '.last_updated' "$registry_file")
  
  echo "Project: $stub"
  echo "Updated: $updated"
  echo ""
  
  # List canvases
  jq -r '.canvases | to_entries[] | "  \(.key): \(.value.title) (\(.value.type))"' "$registry_file"
  
  local count=$(jq '.canvases | length' "$registry_file")
  echo ""
  echo "Total: $count canvases"
  
  return 0
}
```

---

## registry_exists

Check if a canvas key exists in registry.

```bash
# Usage: registry_exists <repo_path> <key>
# Returns: 0 if exists, 1 if not
registry_exists() {
  local repo_path="$1"
  local key="$2"
  
  if [ -z "$repo_path" ] || [ -z "$key" ]; then
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    return 1
  fi
  
  local exists=$(jq -r --arg key "$key" '.canvases[$key] // empty' "$registry_file")
  
  if [ -n "$exists" ]; then
    return 0
  fi
  
  return 1
}
```

---

## registry_get_canvas_id

Get just the canvas ID for a key.

```bash
# Usage: registry_get_canvas_id <repo_path> <key>
# Returns: Canvas ID or empty
registry_get_canvas_id() {
  local repo_path="$1"
  local key="$2"
  
  if [ -z "$repo_path" ] || [ -z "$key" ]; then
    return 1
  fi
  
  local registry_file="$repo_path/.planning/CANVAS-REGISTRY.json"
  
  if [ ! -f "$registry_file" ]; then
    return 1
  fi
  
  jq -r --arg key "$key" '.canvases[$key].canvas_id // empty' "$registry_file"
  return 0
}
```

---

## Usage Examples

```bash
# Initialize registry for marvin project
registry_init "/path/to/marvin" "mvn"

# Add master dashboard
registry_add "/path/to/marvin" "master-dashboard" "F123ABC" "C123DEV" "mvn-dev" "master-dashboard" "Marvin Dashboard"

# Add focus group canvas with extra field
registry_add "/path/to/marvin" "fg-security" "F456DEF" "C456FG" "mvn-fg-security" "focus-group" "Security Focus Group" '{"focus_group":"security"}'

# Get canvas entry
registry_get "/path/to/marvin" "master-dashboard"

# Update sync timestamp
registry_update_sync "/path/to/marvin" "master-dashboard" "abc123"

# List all canvases
registry_list "/path/to/marvin"

# Check if canvas exists
if registry_exists "/path/to/marvin" "fg-security"; then
  echo "Canvas exists"
fi
```

---

*Library created for WGSD Phase 4*
