---
name: wgsd:lib:channel-registry
description: Channel registry management for tracking WGSD Slack channels
---

# Channel Registry Library

Track and manage WGSD channel state across projects.

---

## Registry Location

Channel registry is stored in each project's WGSD config:
```
{workspace}/.planning/WGSD-CONFIG.md
```

### Registry Format

```yaml
# WGSD Configuration
---
stub: mvn
project: marvin
created: 2026-02-22

channels:
  mvn-dev:
    id: C0123456789
    type: dev
    status: active
    created: 2026-02-22
    is_private: true
    
  mvn-community:
    id: C0234567890
    type: community
    status: active
    created: 2026-02-22
    is_private: false
    
  mvn-fg-security:
    id: C0345678901
    type: fg
    name: security
    status: active
    created: 2026-02-22
    is_private: true
    
  mvn-impl-auth-v2:
    id: C0456789012
    type: impl
    name: auth-v2
    status: archived
    created: 2026-02-22
    archived_at: 2026-02-25
    archived_reason: "Implementation complete, merged to develop"
    is_private: true
```

---

## registry_get_config_path

Get the path to the WGSD config file for a workspace.

```bash
# Usage: registry_get_config_path <workspace_path>
# Returns: Path to WGSD-CONFIG.md
registry_get_config_path() {
  local workspace="${1:-.}"
  echo "$workspace/.planning/WGSD-CONFIG.md"
}
```

---

## registry_init

Initialize a new channel registry for a project.

```bash
# Usage: registry_init <workspace_path> <stub> <project_name>
# Returns: Success message or error
registry_init() {
  local workspace="${1:-.}"
  local stub="$2"
  local project="$3"
  
  if [ -z "$stub" ]; then
    echo "ERROR: Stub is required"
    return 1
  fi
  
  local config_path=$(registry_get_config_path "$workspace")
  local planning_dir="$workspace/.planning"
  
  # Create .planning directory if needed
  if [ ! -d "$planning_dir" ]; then
    mkdir -p "$planning_dir"
  fi
  
  # Check if config already exists
  if [ -f "$config_path" ]; then
    echo "⚠️  WGSD config already exists at $config_path"
    return 0
  fi
  
  local today=$(date +%Y-%m-%d)
  
  cat > "$config_path" << EOF
# WGSD Configuration

**Project:** ${project:-$stub}
**Stub:** $stub
**Created:** $today

---

## Channel Registry

| Channel | ID | Type | Status | Created |
|---------|----|----- |--------|---------|

---

## Configuration

\`\`\`yaml
stub: $stub
project: ${project:-$stub}
created: $today
channels: {}
\`\`\`

---

*Managed by WGSD Channel Infrastructure*
EOF

  echo "✅ Registry initialized at $config_path"
  return 0
}
```

---

## registry_add_channel

Add a channel to the registry.

```bash
# Usage: registry_add_channel <workspace> <channel_name> <channel_id> <type> [name] [is_private]
# Returns: Success message or error
registry_add_channel() {
  local workspace="${1:-.}"
  local channel_name="$2"
  local channel_id="$3"
  local type="$4"
  local name="$5"
  local is_private="${6:-true}"
  
  if [ -z "$channel_name" ] || [ -z "$channel_id" ] || [ -z "$type" ]; then
    echo "ERROR: channel_name, channel_id, and type are required"
    return 1
  fi
  
  local config_path=$(registry_get_config_path "$workspace")
  local today=$(date +%Y-%m-%d)
  
  # Check if config exists
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found. Run registry_init first."
    return 1
  fi
  
  echo "📝 Adding channel to registry: $channel_name"
  
  # Add to the markdown table
  local table_line="| $channel_name | $channel_id | $type | active | $today |"
  
  # Insert before the last table separator or at end of table
  if grep -q "^|.*|.*|.*|.*|.*|$" "$config_path"; then
    # Find the empty row marker and insert before it
    sed -i "/^---$/i $table_line" "$config_path" 2>/dev/null || \
    echo "$table_line" >> "$config_path"
  fi
  
  echo "✅ Channel registered: $channel_name ($channel_id)"
  
  # Output for parsing
  echo "REGISTERED: $channel_name $channel_id $type"
  return 0
}
```

---

## registry_get_channel

Get channel information from registry.

```bash
# Usage: registry_get_channel <workspace> <channel_name>
# Returns: Channel info or error
registry_get_channel() {
  local workspace="${1:-.}"
  local channel_name="$2"
  
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  # Parse the table to find the channel
  local line=$(grep "| $channel_name |" "$config_path" | head -1)
  
  if [ -z "$line" ]; then
    echo "ERROR: Channel $channel_name not found in registry"
    return 1
  fi
  
  # Extract fields
  local id=$(echo "$line" | awk -F'|' '{print $3}' | tr -d ' ')
  local type=$(echo "$line" | awk -F'|' '{print $4}' | tr -d ' ')
  local status=$(echo "$line" | awk -F'|' '{print $5}' | tr -d ' ')
  local created=$(echo "$line" | awk -F'|' '{print $6}' | tr -d ' ')
  
  echo "CHANNEL_NAME: $channel_name"
  echo "CHANNEL_ID: $id"
  echo "TYPE: $type"
  echo "STATUS: $status"
  echo "CREATED: $created"
  
  return 0
}
```

---

## registry_update_status

Update channel status in registry.

```bash
# Usage: registry_update_status <workspace> <channel_name> <new_status> [reason]
# Status values: active, archived, deleted
# Returns: Success message or error
registry_update_status() {
  local workspace="${1:-.}"
  local channel_name="$2"
  local new_status="$3"
  local reason="${4:-}"
  
  if [ -z "$channel_name" ] || [ -z "$new_status" ]; then
    echo "ERROR: channel_name and new_status are required"
    return 1
  fi
  
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  echo "📝 Updating channel status: $channel_name → $new_status"
  
  # Update the status in the table
  # This is a simplified update - in production, use proper YAML parsing
  sed -i "s/| $channel_name |\\(.*\\)| active |/| $channel_name |\\1| $new_status |/g" "$config_path"
  sed -i "s/| $channel_name |\\(.*\\)| archived |/| $channel_name |\\1| $new_status |/g" "$config_path"
  
  echo "✅ Status updated: $channel_name is now $new_status"
  
  if [ -n "$reason" ]; then
    echo "📋 Reason: $reason"
  fi
  
  return 0
}
```

---

## registry_list_channels

List all channels in registry.

```bash
# Usage: registry_list_channels <workspace> [type] [status]
# Returns: Channel list
registry_list_channels() {
  local workspace="${1:-.}"
  local type_filter="$2"
  local status_filter="$3"
  
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  echo "📋 Channels in registry:"
  echo ""
  
  # Extract and filter table rows
  grep "^|" "$config_path" | grep -v "^| Channel" | grep -v "^|--" | while read -r line; do
    local name=$(echo "$line" | awk -F'|' '{print $2}' | tr -d ' ')
    local id=$(echo "$line" | awk -F'|' '{print $3}' | tr -d ' ')
    local type=$(echo "$line" | awk -F'|' '{print $4}' | tr -d ' ')
    local status=$(echo "$line" | awk -F'|' '{print $5}' | tr -d ' ')
    
    # Skip empty rows
    [ -z "$name" ] && continue
    
    # Apply filters
    if [ -n "$type_filter" ] && [ "$type" != "$type_filter" ]; then
      continue
    fi
    
    if [ -n "$status_filter" ] && [ "$status" != "$status_filter" ]; then
      continue
    fi
    
    # Format status indicator
    local status_icon="✅"
    case "$status" in
      archived) status_icon="📦" ;;
      deleted) status_icon="❌" ;;
    esac
    
    echo "$status_icon #$name ($type) - $id"
  done
  
  return 0
}
```

---

## registry_channel_exists

Check if a channel exists in registry.

```bash
# Usage: registry_channel_exists <workspace> <channel_name>
# Returns: 0 if exists, 1 if not
registry_channel_exists() {
  local workspace="${1:-.}"
  local channel_name="$2"
  
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    return 1
  fi
  
  if grep -q "| $channel_name |" "$config_path"; then
    return 0
  fi
  
  return 1
}
```

---

## registry_get_stub

Get the project stub from registry.

```bash
# Usage: registry_get_stub <workspace>
# Returns: Stub string
registry_get_stub() {
  local workspace="${1:-.}"
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  # Extract stub from config
  local stub=$(grep "^stub:" "$config_path" | head -1 | awk '{print $2}')
  
  if [ -z "$stub" ]; then
    # Try alternate format
    stub=$(grep "^\*\*Stub:\*\*" "$config_path" | head -1 | awk '{print $2}')
  fi
  
  if [ -z "$stub" ]; then
    echo "ERROR: Stub not found in config"
    return 1
  fi
  
  echo "$stub"
  return 0
}
```

---

## registry_sync_from_slack

Synchronize registry with actual Slack state.

```bash
# Usage: registry_sync_from_slack <workspace>
# Returns: Sync report
registry_sync_from_slack() {
  local workspace="${1:-.}"
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "ERROR: WGSD config not found"
    return 1
  fi
  
  local stub=$(registry_get_stub "$workspace")
  if [ $? -ne 0 ]; then
    echo "$stub"
    return 1
  fi
  
  echo "🔄 Syncing registry with Slack..."
  echo ""
  
  # Get channels from Slack with this stub prefix
  # Note: This requires the slack_list_channels function from slack-api.md
  echo "📡 Fetching channels from Slack with prefix: $stub-"
  echo ""
  
  # This would integrate with slack-api.md functions:
  # local slack_channels=$(slack_list_channels "public_channel,private_channel" "$stub-")
  
  echo "⚠️  Manual sync: Review Slack channels and update registry as needed"
  echo ""
  echo "Registry location: $config_path"
  
  return 0
}
```

---

## registry_get_active_implementations

Get count of active implementations (for scaling enforcement).

```bash
# Usage: registry_get_active_implementations <workspace>
# Returns: Count of active impl channels
registry_get_active_implementations() {
  local workspace="${1:-.}"
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    echo "0"
    return 0
  fi
  
  # Count impl channels with active status
  local count=$(grep "^|" "$config_path" | grep "| impl |" | grep "| active |" | wc -l)
  
  echo "$count"
  return 0
}
```

---

## registry_get_focus_groups

List all focus groups in registry.

```bash
# Usage: registry_get_focus_groups <workspace>
# Returns: List of focus group names
registry_get_focus_groups() {
  local workspace="${1:-.}"
  local config_path=$(registry_get_config_path "$workspace")
  
  if [ ! -f "$config_path" ]; then
    return 1
  fi
  
  # Extract focus group channel names
  grep "^|" "$config_path" | grep "| fg |" | while read -r line; do
    local name=$(echo "$line" | awk -F'|' '{print $2}' | tr -d ' ')
    # Extract the focus group name from channel name (e.g., mvn-fg-security → security)
    local fg_name=$(echo "$name" | sed 's/.*-fg-//')
    echo "$fg_name"
  done
  
  return 0
}
```

---

## Usage Examples

```bash
# Initialize registry for a new project
registry_init "/path/to/marvin" "mvn" "marvin"

# Add a channel to registry
registry_add_channel "/path/to/marvin" "mvn-dev" "C0123456789" "dev"
registry_add_channel "/path/to/marvin" "mvn-fg-security" "C0234567890" "fg" "security"

# Get channel info
registry_get_channel "/path/to/marvin" "mvn-dev"

# Update channel status
registry_update_status "/path/to/marvin" "mvn-impl-auth-v2" "archived" "Implementation complete"

# List all active channels
registry_list_channels "/path/to/marvin" "" "active"

# List only focus groups
registry_list_channels "/path/to/marvin" "fg"

# Check implementation scaling
impl_count=$(registry_get_active_implementations "/path/to/marvin")
if [ "$impl_count" -ge 4 ]; then
  echo "⚠️  WARNING: $impl_count active implementations (limit: 4)"
fi
```

---

*Library created for WGSD Phase 3*
