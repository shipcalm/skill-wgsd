---
name: wgsd:lib:canvas-templates
description: Canvas template engine for WGSD
---

# Canvas Templates Library

Templates for WGSD canvases with variable substitution.

---

## Template Variables

Templates use `{VARIABLE_NAME}` syntax for substitution.

### Common Variables
| Variable | Description |
|----------|-------------|
| `{PROJECT}` | Project name (e.g., "Marvin") |
| `{STUB}` | Project stub (e.g., "mvn") |
| `{TIMESTAMP}` | Last sync timestamp |
| `{DATE}` | Current date |

### State Variables
| Variable | Description |
|----------|-------------|
| `{FOCUS_GROUPS_LIST}` | Formatted list of focus groups |
| `{IMPLEMENTATIONS_LIST}` | Active implementations |
| `{QUEUE_LIST}` | Implementation queue |
| `{CONCEPTS_LIST}` | Concepts in focus group |
| `{COMPLETED_LIST}` | Recently completed items |

---

## template_render

Render a template with variable substitution.

```bash
# Usage: template_render <template> <variables_json>
# Arguments:
#   template       - Template string with {VARIABLES}
#   variables_json - JSON object of variable values
# Returns: Rendered template
template_render() {
  local template="$1"
  local variables="$2"
  
  if [ -z "$template" ]; then
    echo "ERROR: Template is required"
    return 1
  fi
  
  # Default empty variables
  variables="${variables:-{}}"
  
  local result="$template"
  
  # Extract and substitute each variable
  local keys=$(echo "$variables" | jq -r 'keys[]')
  
  for key in $keys; do
    local value=$(echo "$variables" | jq -r --arg k "$key" '.[$k]')
    # Escape special characters in value for sed
    local escaped_value=$(echo "$value" | sed -e 's/[\/&]/\\&/g' -e 's/$/\\n/' | tr -d '\n')
    result=$(echo "$result" | sed "s/{$key}/$escaped_value/g")
  done
  
  # Clean up any unsubstituted variables (set to empty)
  result=$(echo "$result" | sed 's/{[A-Z_]*}//g')
  
  echo "$result"
  return 0
}
```

---

## Template: Master Dashboard

```bash
# Usage: template_master_dashboard
# Returns: Master dashboard template
template_master_dashboard() {
  cat << 'TEMPLATE'
# {PROJECT} Development Dashboard

## 🎯 WGSD Methodology

**We Get Shit Done** - Social collaborative development workflow:

### How It Works
1. **Focus Groups** 🎯 - Long-lived topic channels for ideation and planning
2. **Concepts** 💡 - Feature ideas that mature through team discussion
3. **Implementations** 🚀 - Short-lived execution sprints (1-3 days max)

### Key Principles
- **Parallel Development** - Multiple concepts can evolve simultaneously
- **Natural Scaling** - ~1 active implementation per focus group
- **AI-Managed Canvas** - This dashboard updates automatically
- **Git + Slack Sync** - Planning files stay synchronized

---

## 📊 Current State

### Active Focus Groups
{FOCUS_GROUPS_LIST}

### 🚀 Active Implementations
{IMPLEMENTATIONS_LIST}

### 📋 Implementation Queue
{QUEUE_LIST}

---

## 🔗 Quick Links

- **Repository**: `{REPO_URL}`
- **Planning Files**: `.planning/`
- **Command**: `/wgsd status`

---

## 🔄 Last Sync
{TIMESTAMP}

---

*This canvas is AI-managed. Discuss changes in the channel, not by editing here.*
TEMPLATE
}
```

---

## Template: Focus Group Canvas

```bash
# Usage: template_focus_group <focus_group_name>
# Returns: Focus group template
template_focus_group() {
  local name="${1:-Focus Group}"
  
  cat << TEMPLATE
# {FOCUS_GROUP_NAME} Focus Group

## 📋 Roadmap

{ROADMAP_CONTENT}

---

## 💡 Concepts

### ✅ Ready for Implementation
{READY_CONCEPTS}

### 🔄 In Development
{ACTIVE_CONCEPTS}

### 📝 Proposed
{PROPOSED_CONCEPTS}

---

## 📈 Recent Activity

{RECENT_ACTIVITY}

---

## 🎯 Focus Group Info

- **Channel**: #{CHANNEL_NAME}
- **Created**: {CREATED_DATE}
- **Concepts**: {CONCEPT_COUNT}

---

## 🔄 Last Sync
{TIMESTAMP}

---

*AI-managed canvas. Discuss ideas in the channel to add or update concepts.*
TEMPLATE
}
```

---

## Template: Implementation Dashboard

```bash
# Usage: template_implementation_dashboard
# Returns: Implementation dashboard template
template_implementation_dashboard() {
  cat << 'TEMPLATE'
# 🚀 Implementation Dashboard

## Active Implementations ({ACTIVE_COUNT}/{MAX_COUNT})

{ACTIVE_IMPLEMENTATIONS}

---

## 📋 Queue

{QUEUED_IMPLEMENTATIONS}

---

## ✅ Recently Completed

{COMPLETED_IMPLEMENTATIONS}

---

## 📊 Velocity Metrics

| Metric | Value |
|--------|-------|
| Avg Implementation Time | {AVG_TIME} |
| Completed This Week | {WEEKLY_COUNT} |
| In Queue | {QUEUE_COUNT} |

---

## ⚡ Quick Actions

- `/wgsd create-implementation [concept]` - Start new implementation
- `/wgsd complete-implementation [name]` - Mark as complete
- `/wgsd list-implementations` - View detailed status

---

## 🔄 Last Sync
{TIMESTAMP}

*AI-managed dashboard. Use WGSD commands to manage implementations.*
TEMPLATE
}
```

---

## Template: Community Roadmap

```bash
# Usage: template_community_roadmap
# Returns: Community roadmap template
template_community_roadmap() {
  cat << 'TEMPLATE'
# {PROJECT} Public Roadmap

Welcome to the {PROJECT} development roadmap! 👋

---

## 🎯 What We're Building

{PROJECT_SUMMARY}

---

## 📅 Current Focus Areas

{PUBLIC_FOCUS_AREAS}

---

## 🚀 In Progress

{IN_PROGRESS}

---

## ✅ Recently Shipped

{RECENTLY_SHIPPED}

---

## 💬 Want to Contribute?

We love community feedback! Here's how you can help:

1. **Share Ideas** - Post feature suggestions in this channel
2. **Report Issues** - Let us know about bugs or problems
3. **Discuss** - Join conversations about upcoming features

Great suggestions may be promoted to our internal development process, and contributors may be invited to participate in focus group discussions!

---

## 📊 Development Stats

| Metric | Value |
|--------|-------|
| Active Focus Areas | {FOCUS_COUNT} |
| In Development | {ACTIVE_COUNT} |
| Shipped This Month | {MONTHLY_COUNT} |

---

*Updated automatically by our development system.*
*Last update: {TIMESTAMP}*
TEMPLATE
}
```

---

## Template: Concept Canvas

```bash
# Usage: template_concept <concept_name>
# Returns: Concept canvas template
template_concept() {
  cat << 'TEMPLATE'
# 💡 {CONCEPT_NAME}

## Overview

{CONCEPT_OVERVIEW}

---

## Status: {CONCEPT_STATUS}

{STATUS_DETAILS}

---

## Requirements

{REQUIREMENTS}

---

## Design Notes

{DESIGN_NOTES}

---

## Discussion Highlights

{DISCUSSION_HIGHLIGHTS}

---

## Next Steps

{NEXT_STEPS}

---

## Metadata

| Field | Value |
|-------|-------|
| Focus Group | {FOCUS_GROUP} |
| Created | {CREATED_DATE} |
| Author | {AUTHOR} |
| Status | {CONCEPT_STATUS} |

---

*AI-managed canvas. Discuss in #{CHANNEL_NAME} to update.*
TEMPLATE
}
```

---

## State Formatting Functions

### format_focus_groups_list

Format focus groups for dashboard display.

```bash
# Usage: format_focus_groups_list <repo_path>
# Returns: Formatted markdown list
format_focus_groups_list() {
  local repo_path="$1"
  local fg_dir="$repo_path/.planning/focus-groups"
  
  if [ ! -d "$fg_dir" ]; then
    echo "*No focus groups yet. Create one with `/wgsd create-focus-group [name]`*"
    return 0
  fi
  
  local result=""
  
  for fg in "$fg_dir"/*/; do
    [ -d "$fg" ] || continue
    
    local name=$(basename "$fg")
    local state_file="$fg/STATE.md"
    
    # Get concept count
    local concept_count=0
    if [ -d "$fg/concepts" ]; then
      concept_count=$(ls -1 "$fg/concepts"/*.md 2>/dev/null | wc -l)
    fi
    
    # Get status from STATE.md if exists
    local status="🟢 Active"
    if [ -f "$state_file" ]; then
      local state_line=$(grep -m1 "^Status:" "$state_file" 2>/dev/null || true)
      if [ -n "$state_line" ]; then
        status=$(echo "$state_line" | cut -d':' -f2 | xargs)
      fi
    fi
    
    result="${result}- **$name** ($concept_count concepts) - $status\n"
  done
  
  if [ -z "$result" ]; then
    echo "*No focus groups yet*"
  else
    echo -e "$result"
  fi
  
  return 0
}
```

### format_implementations_list

Format active implementations for dashboard.

```bash
# Usage: format_implementations_list <repo_path>
# Returns: Formatted markdown list
format_implementations_list() {
  local repo_path="$1"
  local impl_dir="$repo_path/.planning/active-implementations"
  
  if [ ! -d "$impl_dir" ]; then
    echo "*No active implementations*"
    return 0
  fi
  
  local result=""
  local count=0
  
  for impl in "$impl_dir"/*/; do
    [ -d "$impl" ] || continue
    
    local name=$(basename "$impl")
    local state_file="$impl/STATE.md"
    
    # Get status and owner from STATE.md
    local status="🔄 In Progress"
    local owner="Unassigned"
    
    if [ -f "$state_file" ]; then
      local owner_line=$(grep -m1 "^Owner:" "$state_file" 2>/dev/null || true)
      if [ -n "$owner_line" ]; then
        owner=$(echo "$owner_line" | cut -d':' -f2 | xargs)
      fi
      
      local status_line=$(grep -m1 "^Status:" "$state_file" 2>/dev/null || true)
      if [ -n "$status_line" ]; then
        status=$(echo "$status_line" | cut -d':' -f2 | xargs)
      fi
    fi
    
    result="${result}- **$name** - $status (Owner: $owner)\n"
    count=$((count + 1))
  done
  
  if [ -z "$result" ]; then
    echo "*No active implementations*"
  else
    echo -e "$result"
  fi
  
  return 0
}
```

### format_queue_list

Format implementation queue.

```bash
# Usage: format_queue_list <repo_path>
# Returns: Formatted markdown list
format_queue_list() {
  local repo_path="$1"
  local queue_file="$repo_path/.planning/IMPLEMENTATION-QUEUE.md"
  
  if [ ! -f "$queue_file" ]; then
    echo "*Queue empty*"
    return 0
  fi
  
  # Extract queued items from file
  local items=$(grep -E "^-\s*\[" "$queue_file" 2>/dev/null | head -5)
  
  if [ -z "$items" ]; then
    echo "*Queue empty*"
  else
    echo "$items"
  fi
  
  return 0
}
```

### format_concepts_list

Format concepts for focus group canvas. Supports v2.2 directories and legacy files.

```bash
# Usage: format_concepts_list <repo_path> <focus_group> <status>
# Arguments:
#   status - "ready", "active", "proposed", "all"
# Returns: Formatted markdown list
format_concepts_list() {
  local repo_path="$1"
  local fg_name="$2"
  local status_filter="$3"
  
  local concepts_dir="$repo_path/.planning/focus-groups/$fg_name/concepts"
  
  if [ ! -d "$concepts_dir" ]; then
    echo "*No concepts yet*"
    return 0
  fi
  
  local result=""
  
  # Process v2.2 concept directories first
  for concept_dir in "$concepts_dir"/*/; do
    [ -d "$concept_dir" ] || continue
    
    local name=$(basename "$concept_dir")
    local concept_file="$concept_dir/CONCEPT.md"
    
    [ -f "$concept_file" ] || continue
    
    # Get status from file
    local status=$(grep -m1 "^\*\*Status:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Status:\*\*//' | xargs || echo "draft")
    status=$(echo "$status" | tr '[:upper:]' '[:lower:]')
    
    # Filter by status
    case "$status_filter" in
      ready)
        [[ "$status" == *"ready"* ]] || continue
        ;;
      active)
        [[ "$status" == *"active"* || "$status" == *"exploring"* || "$status" == *"development"* ]] || continue
        ;;
      proposed)
        [[ "$status" == *"proposed"* || "$status" == *"draft"* ]] || continue
        ;;
      all)
        # Include all
        ;;
    esac
    
    # Count artifacts in directory
    local artifact_count=$(ls -1 "$concept_dir" 2>/dev/null | wc -l)
    
    # Get brief description
    local desc=""
    if [ -f "$concept_dir/impact-matrix.md" ]; then
      local impact_count=$(grep -c "^### Focus Group:" "$concept_dir/impact-matrix.md" 2>/dev/null || echo "0")
      desc="📁 ${artifact_count} artifacts, 🎯 ${impact_count} impacts"
    else
      desc="📁 ${artifact_count} artifacts"
    fi
    
    # Get priority
    local priority=$(grep -m1 "^\*\*Priority:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Priority:\*\*//' | xargs || echo "")
    
    # Status emoji
    local emoji="📝"
    case "$status" in
      *"ready"*) emoji="✅" ;;
      *"exploring"*|*"active"*) emoji="🔍" ;;
      *"mature"*) emoji="🌟" ;;
      *"approved"*) emoji="✨" ;;
    esac
    
    if [ -n "$priority" ]; then
      result="${result}- $emoji **$name** ($priority) - $desc\n"
    else
      result="${result}- $emoji **$name** - $desc\n"
    fi
  done
  
  # Process legacy single-file concepts
  for concept_file in "$concepts_dir"/*.md; do
    [ -f "$concept_file" ] || continue
    
    local name=$(basename "$concept_file" .md)
    
    # Get status from file
    local status=$(grep -m1 "^\*\*Status:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Status:\*\*//' | xargs || echo "draft")
    status=$(echo "$status" | tr '[:upper:]' '[:lower:]')
    
    # Filter by status
    case "$status_filter" in
      ready)
        [[ "$status" == *"ready"* ]] || continue
        ;;
      active)
        [[ "$status" == *"active"* || "$status" == *"exploring"* ]] || continue
        ;;
      proposed)
        [[ "$status" == *"proposed"* || "$status" == *"draft"* ]] || continue
        ;;
      all)
        # Include all
        ;;
    esac
    
    # Get description
    local desc=$(grep -m1 "^>" "$concept_file" 2>/dev/null | sed 's/^>//' | xargs || echo "📄 legacy")
    
    # Status emoji
    local emoji="📄"
    case "$status" in
      *"ready"*) emoji="✅" ;;
      *"exploring"*|*"active"*) emoji="🔍" ;;
    esac
    
    result="${result}- $emoji **$name**: $desc\n"
  done
  
  if [ -z "$result" ]; then
    echo "*None*"
  else
    echo -e "$result"
  fi
  
  return 0
}
```

---

### format_concept_detail

Format a single concept directory for detailed display on canvas.

```bash
# Usage: format_concept_detail <repo_path> <focus_group> <concept_name>
# Returns: Detailed markdown content for canvas
format_concept_detail() {
  local repo_path="$1"
  local fg_name="$2"
  local concept_name="$3"
  
  local concept_dir="$repo_path/.planning/focus-groups/$fg_name/concepts/$concept_name"
  local concept_file="$concept_dir/CONCEPT.md"
  
  # Handle legacy format
  if [ ! -d "$concept_dir" ]; then
    concept_file="$repo_path/.planning/focus-groups/$fg_name/concepts/${concept_name}.md"
    if [ -f "$concept_file" ]; then
      cat "$concept_file"
      return 0
    fi
    echo "Concept not found: $concept_name"
    return 1
  fi
  
  local result=""
  
  # Header with concept name and key metadata
  local status=$(grep -m1 "^\*\*Status:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Status:\*\*//' | xargs || echo "Draft")
  local priority=$(grep -m1 "^\*\*Priority:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Priority:\*\*//' | xargs || echo "")
  local branch=$(grep -m1 "^\*\*Branch:\*\*" "$concept_file" 2>/dev/null | sed 's/\*\*Branch:\*\*//' | xargs || echo "")
  
  result+="## 💡 $concept_name\n\n"
  result+="**Status:** $status"
  [ -n "$priority" ] && result+=" | **Priority:** $priority"
  [ -n "$branch" ] && result+=" | **Branch:** $branch"
  result+="\n\n"
  
  # Include CONCEPT.md content (trimmed)
  result+="### Overview\n\n"
  # Extract just the overview/problem sections
  local overview=$(sed -n '/## Problem Statement/,/## Dependencies/p' "$concept_file" 2>/dev/null | head -30)
  if [ -n "$overview" ]; then
    result+="$overview\n\n"
  fi
  
  # Include impact matrix summary if exists
  local impact_file="$concept_dir/impact-matrix.md"
  if [ -f "$impact_file" ]; then
    result+="### 📊 Impact Matrix\n\n"
    # Extract impact summary table
    local summary=$(sed -n '/## Impact Summary/,/## Approval/p' "$impact_file" 2>/dev/null | head -15)
    if [ -n "$summary" ]; then
      result+="$summary\n\n"
    fi
  fi
  
  # List other artifacts
  result+="### 📎 Related Artifacts\n\n"
  local has_artifacts=false
  
  for artifact in "$concept_dir"/*; do
    [ -f "$artifact" ] || continue
    local artifact_name=$(basename "$artifact")
    
    case "$artifact_name" in
      CONCEPT.md|impact-matrix.md)
        continue  # Already shown inline
        ;;
      API-SPEC.md)
        result+="- 📡 [API Specification]($artifact_name)\n"
        has_artifacts=true
        ;;
      acceptance-criteria.md)
        result+="- ✅ [Acceptance Criteria]($artifact_name)\n"
        has_artifacts=true
        ;;
      *.md)
        result+="- 📄 [$artifact_name]($artifact_name)\n"
        has_artifacts=true
        ;;
    esac
  done
  
  # Check for wireframes directory
  if [ -d "$concept_dir/wireframes" ]; then
    local wireframe_count=$(ls -1 "$concept_dir/wireframes" 2>/dev/null | wc -l)
    result+="- 🎨 [Wireframes ($wireframe_count files)](wireframes/)\n"
    has_artifacts=true
  fi
  
  if [ "$has_artifacts" = false ]; then
    result+="*No additional artifacts*\n"
  fi
  
  result+="\n---\n"
  
  echo -e "$result"
  return 0
}
```

---

## Build Canvas Content Functions

### build_master_dashboard_content

Build complete master dashboard content from state.

```bash
# Usage: build_master_dashboard_content <repo_path> <project_name> <stub>
# Returns: Complete markdown content
build_master_dashboard_content() {
  local repo_path="$1"
  local project_name="$2"
  local stub="$3"
  
  local template=$(template_master_dashboard)
  
  # Build variable values
  local focus_groups=$(format_focus_groups_list "$repo_path")
  local implementations=$(format_implementations_list "$repo_path")
  local queue=$(format_queue_list "$repo_path")
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  
  # Get repo URL from git config
  local repo_url=$(cd "$repo_path" && git remote get-url origin 2>/dev/null || echo "N/A")
  
  # Build variables JSON
  local vars=$(jq -n \
    --arg project "$project_name" \
    --arg stub "$stub" \
    --arg fg "$focus_groups" \
    --arg impl "$implementations" \
    --arg queue "$queue" \
    --arg timestamp "$timestamp" \
    --arg repo_url "$repo_url" \
    '{
      PROJECT: $project,
      STUB: $stub,
      FOCUS_GROUPS_LIST: $fg,
      IMPLEMENTATIONS_LIST: $impl,
      QUEUE_LIST: $queue,
      TIMESTAMP: $timestamp,
      REPO_URL: $repo_url
    }')
  
  template_render "$template" "$vars"
}
```

### build_focus_group_content

Build focus group canvas content.

```bash
# Usage: build_focus_group_content <repo_path> <focus_group_name> <channel_name>
# Returns: Complete markdown content
build_focus_group_content() {
  local repo_path="$1"
  local fg_name="$2"
  local channel_name="$3"
  
  local template=$(template_focus_group)
  local fg_dir="$repo_path/.planning/focus-groups/$fg_name"
  
  # Read roadmap content
  local roadmap="*Roadmap not yet defined*"
  if [ -f "$fg_dir/ROADMAP.md" ]; then
    roadmap=$(cat "$fg_dir/ROADMAP.md" | head -50)
  fi
  
  # Format concepts by status
  local ready=$(format_concepts_list "$repo_path" "$fg_name" "ready")
  local active=$(format_concepts_list "$repo_path" "$fg_name" "active")
  local proposed=$(format_concepts_list "$repo_path" "$fg_name" "proposed")
  
  # Count concepts
  local count=0
  if [ -d "$fg_dir/concepts" ]; then
    count=$(ls -1 "$fg_dir/concepts"/*.md 2>/dev/null | wc -l)
  fi
  
  local timestamp=$(date -u +"%Y-%m-%d %H:%M UTC")
  local created=$(stat -c %y "$fg_dir" 2>/dev/null | cut -d' ' -f1 || echo "Unknown")
  
  local vars=$(jq -n \
    --arg name "$fg_name" \
    --arg channel "$channel_name" \
    --arg roadmap "$roadmap" \
    --arg ready "$ready" \
    --arg active "$active" \
    --arg proposed "$proposed" \
    --arg count "$count" \
    --arg timestamp "$timestamp" \
    --arg created "$created" \
    '{
      FOCUS_GROUP_NAME: $name,
      CHANNEL_NAME: $channel,
      ROADMAP_CONTENT: $roadmap,
      READY_CONCEPTS: $ready,
      ACTIVE_CONCEPTS: $active,
      PROPOSED_CONCEPTS: $proposed,
      CONCEPT_COUNT: $count,
      TIMESTAMP: $timestamp,
      CREATED_DATE: $created,
      RECENT_ACTIVITY: "*Recent activity tracking coming soon*"
    }')
  
  template_render "$template" "$vars"
}
```

---

## Usage Examples

```bash
# Get master dashboard template
template=$(template_master_dashboard)

# Render with variables
vars='{"PROJECT": "Marvin", "STUB": "mvn", "FOCUS_GROUPS_LIST": "- Security\n- Billing"}'
rendered=$(template_render "$template" "$vars")

# Build complete dashboard from repo state
content=$(build_master_dashboard_content "/path/to/repo" "Marvin" "mvn")

# Format focus groups for display
list=$(format_focus_groups_list "/path/to/repo")
```

---

*Library created for WGSD Phase 4*
