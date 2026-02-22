---
name: wgsd:lib:canvas-api
description: Canvas API library for WGSD canvas management
---

# Canvas API Library

Extended Slack Canvas API functions for WGSD canvas management.

---

## Configuration

Canvas API uses the same Slack bot token as channel operations.
See `lib/slack-api.md` for token retrieval.

---

## canvas_create_attached

Create a canvas attached to a channel. The canvas appears as the channel's canvas icon.

```bash
# Usage: canvas_create_attached <channel_id> <title> [markdown_content]
# Arguments:
#   channel_id - Channel to attach canvas to
#   title      - Canvas title
#   content    - Markdown content (optional)
# Returns: Canvas ID or error
canvas_create_attached() {
  local channel_id="$1"
  local title="$2"
  local content="${3:-# $title\n\n*Canvas content will be populated by WGSD*}"
  
  if [ -z "$channel_id" ] || [ -z "$title" ]; then
    echo "ERROR: Channel ID and title are required"
    return 1
  fi
  
  echo "📋 Creating attached canvas: $title"
  
  # Escape content for JSON
  local escaped_content=$(echo "$content" | jq -Rs .)
  
  local data=$(jq -n \
    --arg channel "$channel_id" \
    --arg title "$title" \
    --argjson content "$escaped_content" \
    '{
      channel_id: $channel,
      title: $title,
      document_content: {
        type: "markdown",
        markdown: $content
      }
    }')
  
  local response=$(slack_api_call POST "conversations.canvases.create" "$data")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to create canvas"
    echo "$response"
    return 1
  fi
  
  local canvas_id=$(echo "$response" | jq -r '.canvas_id')
  
  if [ -z "$canvas_id" ] || [ "$canvas_id" = "null" ]; then
    echo "ERROR: No canvas_id in response"
    echo "$response"
    return 1
  fi
  
  echo "✅ Canvas created: $canvas_id"
  echo "CANVAS_ID:$canvas_id"
  return 0
}
```

---

## canvas_create_standalone

Create a standalone canvas (not attached to channel).

```bash
# Usage: canvas_create_standalone <title> [markdown_content]
# Arguments:
#   title   - Canvas title
#   content - Markdown content (optional)
# Returns: Canvas ID or error
canvas_create_standalone() {
  local title="$1"
  local content="${2:-# $title\n\n*Canvas content will be populated by WGSD*}"
  
  if [ -z "$title" ]; then
    echo "ERROR: Title is required"
    return 1
  fi
  
  echo "📋 Creating standalone canvas: $title"
  
  local escaped_content=$(echo "$content" | jq -Rs .)
  
  local data=$(jq -n \
    --arg title "$title" \
    --argjson content "$escaped_content" \
    '{
      title: $title,
      document_content: {
        type: "markdown",
        markdown: $content
      }
    }')
  
  local response=$(slack_api_call POST "canvases.create" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  local canvas_id=$(echo "$response" | jq -r '.canvas_id')
  echo "✅ Canvas created: $canvas_id"
  echo "CANVAS_ID:$canvas_id"
  return 0
}
```

---

## canvas_edit

Replace entire canvas content.

```bash
# Usage: canvas_edit <canvas_id> <markdown_content>
# Arguments:
#   canvas_id - Canvas to edit
#   content   - New markdown content
# Returns: Success or error
canvas_edit() {
  local canvas_id="$1"
  local content="$2"
  
  if [ -z "$canvas_id" ] || [ -z "$content" ]; then
    echo "ERROR: Canvas ID and content are required"
    return 1
  fi
  
  echo "✏️  Editing canvas: $canvas_id"
  
  local escaped_content=$(echo "$content" | jq -Rs .)
  
  local data=$(jq -n \
    --arg canvas_id "$canvas_id" \
    --argjson content "$escaped_content" \
    '{
      canvas_id: $canvas_id,
      changes: [
        {
          operation: "replace",
          document_content: {
            type: "markdown",
            markdown: $content
          }
        }
      ]
    }')
  
  local response=$(slack_api_call POST "canvases.edit" "$data")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to edit canvas"
    echo "$response"
    return 1
  fi
  
  echo "✅ Canvas updated"
  return 0
}
```

---

## canvas_edit_section

Edit a specific section of canvas by section ID.

```bash
# Usage: canvas_edit_section <canvas_id> <section_id> <operation> <content>
# Arguments:
#   canvas_id  - Canvas to edit
#   section_id - Section to target
#   operation  - "replace", "insert_after", "insert_before", "delete"
#   content    - New markdown content (not required for delete)
# Returns: Success or error
canvas_edit_section() {
  local canvas_id="$1"
  local section_id="$2"
  local operation="$3"
  local content="$4"
  
  if [ -z "$canvas_id" ] || [ -z "$section_id" ] || [ -z "$operation" ]; then
    echo "ERROR: Canvas ID, section ID, and operation are required"
    return 1
  fi
  
  echo "✏️  Editing section $section_id ($operation)"
  
  local change
  if [ "$operation" = "delete" ]; then
    change=$(jq -n \
      --arg op "$operation" \
      --arg section "$section_id" \
      '{operation: $op, section_id: $section}')
  else
    local escaped_content=$(echo "$content" | jq -Rs .)
    change=$(jq -n \
      --arg op "$operation" \
      --arg section "$section_id" \
      --argjson content "$escaped_content" \
      '{
        operation: $op,
        section_id: $section,
        document_content: {
          type: "markdown",
          markdown: $content
        }
      }')
  fi
  
  local data=$(jq -n \
    --arg canvas_id "$canvas_id" \
    --argjson change "$change" \
    '{canvas_id: $canvas_id, changes: [$change]}')
  
  local response=$(slack_api_call POST "canvases.edit" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "✅ Section updated"
  return 0
}
```

---

## canvas_lookup_sections

Find sections in canvas by criteria.

```bash
# Usage: canvas_lookup_sections <canvas_id> [criteria]
# Arguments:
#   canvas_id - Canvas to search
#   criteria  - Optional JSON criteria
# Returns: Sections JSON or error
canvas_lookup_sections() {
  local canvas_id="$1"
  local criteria="${2:-{}}"
  
  if [ -z "$canvas_id" ]; then
    echo "ERROR: Canvas ID is required"
    return 1
  fi
  
  local data=$(jq -n \
    --arg canvas_id "$canvas_id" \
    --argjson criteria "$criteria" \
    '{canvas_id: $canvas_id, criteria: $criteria}')
  
  local response=$(slack_api_call POST "canvases.sections.lookup" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "$response" | jq '.sections'
  return 0
}
```

---

## canvas_share_to_channel

Share a standalone canvas to a channel.

```bash
# Usage: canvas_share_to_channel <canvas_id> <channel_id> [access_level]
# Arguments:
#   canvas_id    - Canvas to share
#   channel_id   - Channel to share to
#   access_level - "read" or "write" (default: read)
# Returns: Success or error
canvas_share_to_channel() {
  local canvas_id="$1"
  local channel_id="$2"
  local access_level="${3:-read}"
  
  if [ -z "$canvas_id" ] || [ -z "$channel_id" ]; then
    echo "ERROR: Canvas ID and channel ID are required"
    return 1
  fi
  
  echo "🔗 Sharing canvas to channel..."
  
  local data=$(jq -n \
    --arg canvas_id "$canvas_id" \
    --arg access "$access_level" \
    --arg channel "$channel_id" \
    '{
      canvas_id: $canvas_id,
      access_level: $access,
      channel_ids: [$channel]
    }')
  
  local response=$(slack_api_call POST "canvases.access.set" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "✅ Canvas shared to channel"
  return 0
}
```

---

## canvas_delete

Delete a canvas.

```bash
# Usage: canvas_delete <canvas_id>
# Returns: Success or error
canvas_delete() {
  local canvas_id="$1"
  
  if [ -z "$canvas_id" ]; then
    echo "ERROR: Canvas ID is required"
    return 1
  fi
  
  echo "🗑️  Deleting canvas: $canvas_id"
  
  local data=$(jq -n --arg id "$canvas_id" '{canvas_id: $id}')
  local response=$(slack_api_call POST "canvases.delete" "$data")
  
  if [ $? -ne 0 ]; then
    return 1
  fi
  
  echo "✅ Canvas deleted"
  return 0
}
```

---

## canvas_get_checksum

Generate checksum of content for change detection.

```bash
# Usage: canvas_get_checksum <content>
# Returns: MD5 checksum
canvas_get_checksum() {
  local content="$1"
  echo -n "$content" | md5sum | cut -d' ' -f1
}
```

---

## Canvas Type Constants

```bash
# Canvas types for registry
CANVAS_TYPE_MASTER="master-dashboard"
CANVAS_TYPE_IMPL="implementation-dashboard"
CANVAS_TYPE_FG="focus-group"
CANVAS_TYPE_COMMUNITY="community-roadmap"
CANVAS_TYPE_CONCEPT="concept"
```

---

## Usage Examples

```bash
# Create attached canvas for dev channel
canvas_create_attached "C123DEV" "Development Dashboard" "# Dashboard\n\nContent here"

# Edit canvas with new content
canvas_edit "F123ABC" "# Updated Dashboard\n\nNew content"

# Share standalone canvas to channel
canvas_share_to_channel "F456DEF" "C789COMMUNITY" "read"

# Look up sections
canvas_lookup_sections "F123ABC"

# Delete old canvas
canvas_delete "F123ABC"
```

---

## Error Handling

All functions return:
- `0` on success with output
- `1` on error with ERROR: message

Common errors:
- `canvas_not_found` - Canvas ID invalid
- `channel_not_found` - Channel ID invalid
- `not_allowed` - Insufficient permissions
- `invalid_content` - Malformed markdown

---

*Library created for WGSD Phase 4*
