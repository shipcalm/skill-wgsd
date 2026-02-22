# Priority Queue Library

**Purpose:** Implementation queue management and prioritization for WGSD

---

## Overview

The priority queue manages implementation ordering:
- Concepts ready for implementation enter the queue
- Priority tiers determine execution order
- Queue state is visible in master dashboard canvas
- Natural FIFO ordering within priority tiers

---

## Queue File Structure

Queue state is stored in `.planning/IMPLEMENTATION-QUEUE.md`:

```markdown
# Implementation Queue

**Last Updated:** 2026-02-22T10:00:00Z
**Queue Length:** 4
**Active Implementations:** 2/4

## Active Implementations

| Implementation | Focus Group | Owner | Status | Days |
|----------------|-------------|-------|--------|------|
| auth-v2-impl | security | @alice | implementing | 2 |
| onboarding-wizard-impl | onboarding | @bob | planning | 1 |

## Queue

### 🔴 Critical
*Empty*

### 🟠 High
1. **sso-integration** (security) - Awaiting owner
   - Approved: 2026-02-21
   - Priority set: 2026-02-21

### 🟡 Medium  
2. **billing-api** (billing) - Owner: @charlie (pending acceptance)
   - Approved: 2026-02-20
   - Priority set: 2026-02-20

### 🟢 Low
3. **docs-refactor** (docs) - No owner
   - Approved: 2026-02-19
   - Priority set: 2026-02-19

## Recently Completed

| Implementation | Focus Group | Owner | Completed | Duration |
|----------------|-------------|-------|-----------|----------|
| api-cleanup | api | @dave | 2026-02-18 | 2 days |
```

---

## Priority Tiers

| Tier | Emoji | Description | Use When |
|------|-------|-------------|----------|
| critical | 🔴 | Blocking other work | Outages, blockers |
| high | 🟠 | Business priority | Customer-facing, deadlines |
| medium | 🟡 | Standard priority | Normal development |
| low | 🟢 | Nice to have | Cleanup, optimization |

---

## Add to Queue

Add an approved concept to the implementation queue.

### Usage
```bash
add_to_queue "concept-name" "focus-group" "priority"
```

### Implementation
```bash
add_to_queue() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local PRIORITY="${3:-medium}"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    
    # Validate priority
    case "$PRIORITY" in
        critical|high|medium|low) ;;
        *)
            echo "❌ Invalid priority: $PRIORITY"
            echo "   Valid: critical, high, medium, low"
            return 1
            ;;
    esac
    
    # Map priority to section
    case "$PRIORITY" in
        critical) SECTION="### 🔴 Critical" ;;
        high) SECTION="### 🟠 High" ;;
        medium) SECTION="### 🟡 Medium" ;;
        low) SECTION="### 🟢 Low" ;;
    esac
    
    # Get next queue position for this priority
    POSITION=$(grep -A100 "$SECTION" "$QUEUE_FILE" | grep -E "^[0-9]+\." | tail -1 | sed 's/\..*//')
    POSITION=$((POSITION + 1))
    if [ "$POSITION" -eq 1 ]; then
        # Find last number across all priorities
        POSITION=$(grep -E "^[0-9]+\." "$QUEUE_FILE" | tail -1 | sed 's/\..*//')
        POSITION=$((POSITION + 1))
    fi
    
    # Build queue entry
    QUEUE_ENTRY="${POSITION}. **${CONCEPT_NAME}** (${FOCUS_GROUP}) - Awaiting owner
   - Approved: ${TIMESTAMP}
   - Priority set: ${TIMESTAMP}"
    
    # Insert after section header
    # Using a temp file approach for portability
    awk -v section="$SECTION" -v entry="$QUEUE_ENTRY" '
        $0 == section { print; print entry; next }
        { print }
    ' "$QUEUE_FILE" > "${QUEUE_FILE}.tmp" && mv "${QUEUE_FILE}.tmp" "$QUEUE_FILE"
    
    # Update queue length
    update_queue_stats
    
    echo "✅ Added ${CONCEPT_NAME} to queue (priority: ${PRIORITY}, position: ${POSITION})"
}
```

---

## Remove from Queue

Remove a concept from the queue (when implementation starts or cancelled).

### Usage
```bash
remove_from_queue "concept-name" "reason"
```

### Implementation
```bash
remove_from_queue() {
    local CONCEPT_NAME="$1"
    local REASON="${2:-started}"
    
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Check if in queue
    if ! grep -q "\*\*${CONCEPT_NAME}\*\*" "$QUEUE_FILE"; then
        echo "⚠️ ${CONCEPT_NAME} not found in queue"
        return 1
    fi
    
    # Remove entry (concept and its metadata lines)
    sed -i "/\*\*${CONCEPT_NAME}\*\*/,/^[0-9]\|^$/d" "$QUEUE_FILE"
    
    # Renumber queue
    renumber_queue
    
    # Update stats
    update_queue_stats
    
    echo "✅ Removed ${CONCEPT_NAME} from queue (reason: ${REASON})"
}
```

---

## Update Queue Stats

Update the queue statistics at the top of the file.

### Implementation
```bash
update_queue_stats() {
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Count queued items
    QUEUE_LENGTH=$(grep -E "^\d+\. \*\*" "$QUEUE_FILE" | grep -c "Awaiting\|pending")
    
    # Count active implementations
    ACTIVE_COUNT=$(ls -d .planning/active-implementations/*-impl 2>/dev/null | wc -l)
    
    # Update stats
    sed -i "s/\*\*Last Updated:\*\* .*/\*\*Last Updated:\*\* ${TIMESTAMP}/" "$QUEUE_FILE"
    sed -i "s/\*\*Queue Length:\*\* .*/\*\*Queue Length:\*\* ${QUEUE_LENGTH}/" "$QUEUE_FILE"
    sed -i "s/\*\*Active Implementations:\*\* .*/\*\*Active Implementations:\*\* ${ACTIVE_COUNT}\/4/" "$QUEUE_FILE"
}
```

---

## Renumber Queue

Renumber queue positions after removal.

### Implementation
```bash
renumber_queue() {
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    
    # This is a simplified version - actual implementation would
    # properly renumber across priority sections
    
    local NUM=1
    while IFS= read -r line; do
        if echo "$line" | grep -qE "^[0-9]+\. \*\*"; then
            # Replace number prefix
            NEW_LINE=$(echo "$line" | sed "s/^[0-9]\+\./${NUM}./")
            sed -i "s|${line}|${NEW_LINE}|" "$QUEUE_FILE"
            NUM=$((NUM + 1))
        fi
    done < "$QUEUE_FILE"
}
```

---

## Set Priority

Change priority of a queued concept.

### Usage
```bash
set_priority "concept-name" "new-priority"
```

### Implementation
```bash
set_priority() {
    local CONCEPT_NAME="$1"
    local NEW_PRIORITY="$2"
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Validate priority
    case "$NEW_PRIORITY" in
        critical|high|medium|low) ;;
        *)
            echo "❌ Invalid priority: $NEW_PRIORITY"
            return 1
            ;;
    esac
    
    # Get current entry details
    ENTRY=$(grep -A2 "\*\*${CONCEPT_NAME}\*\*" "$QUEUE_FILE")
    if [ -z "$ENTRY" ]; then
        echo "❌ ${CONCEPT_NAME} not found in queue"
        return 1
    fi
    
    # Extract focus group and owner info
    FOCUS_GROUP=$(echo "$ENTRY" | head -1 | sed 's/.*(\([^)]*\)).*/\1/')
    OWNER_INFO=$(echo "$ENTRY" | head -1 | sed 's/.*- //')
    
    # Remove from current position
    remove_from_queue "$CONCEPT_NAME" "reprioritize"
    
    # Add back at new priority (preserving owner info)
    add_to_queue "$CONCEPT_NAME" "$FOCUS_GROUP" "$NEW_PRIORITY"
    
    # Update priority set timestamp
    sed -i "/${CONCEPT_NAME}/,/Priority set:/{s/Priority set:.*/Priority set: ${TIMESTAMP}/}" "$QUEUE_FILE"
    
    echo "✅ ${CONCEPT_NAME} priority changed to: ${NEW_PRIORITY}"
}
```

---

## Get Queue Position

Get the queue position of a concept.

### Usage
```bash
get_queue_position "concept-name"
```

### Implementation
```bash
get_queue_position() {
    local CONCEPT_NAME="$1"
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    
    POSITION=$(grep -E "^[0-9]+\. \*\*${CONCEPT_NAME}\*\*" "$QUEUE_FILE" | sed 's/\..*//')
    
    if [ -z "$POSITION" ]; then
        # Check if in active
        if grep -q "${CONCEPT_NAME}-impl" "$QUEUE_FILE"; then
            echo "active"
        else
            echo "not_queued"
        fi
    else
        echo "$POSITION"
    fi
}
```

---

## Get Queue Summary

Get formatted queue summary for display.

### Usage
```bash
get_queue_summary "format"
# format: "full" | "compact" | "json"
```

### Implementation
```bash
get_queue_summary() {
    local FORMAT="${1:-compact}"
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    
    case "$FORMAT" in
        full)
            cat "$QUEUE_FILE"
            ;;
        compact)
            echo "📋 **Implementation Queue**"
            echo ""
            
            # Active
            ACTIVE=$(grep -A10 "## Active Implementations" "$QUEUE_FILE" | grep -E "^\| [a-z]" | head -5)
            if [ -n "$ACTIVE" ]; then
                echo "**Active:**"
                echo "$ACTIVE" | while read line; do
                    NAME=$(echo "$line" | awk -F'|' '{print $2}' | xargs)
                    OWNER=$(echo "$line" | awk -F'|' '{print $4}' | xargs)
                    STATUS=$(echo "$line" | awk -F'|' '{print $5}' | xargs)
                    echo "- ${NAME} (${OWNER}) - ${STATUS}"
                done
            fi
            
            echo ""
            echo "**Queued:**"
            
            # Queue by priority
            for PRIORITY in "🔴 Critical" "🟠 High" "🟡 Medium" "🟢 Low"; do
                ITEMS=$(grep -A50 "### ${PRIORITY}" "$QUEUE_FILE" | grep -E "^[0-9]+\." | head -3)
                if [ -n "$ITEMS" ]; then
                    echo "  ${PRIORITY}:"
                    echo "$ITEMS" | while read line; do
                        echo "    $line" | sed 's/- .*//'
                    done
                fi
            done
            ;;
        json)
            # Output JSON format for programmatic use
            cat <<EOF
{
  "active": $(grep -c "^\| [a-z]" "$QUEUE_FILE" || echo 0),
  "queued": {
    "critical": $(grep -A20 "### 🔴 Critical" "$QUEUE_FILE" | grep -c "^\*\*" || echo 0),
    "high": $(grep -A20 "### 🟠 High" "$QUEUE_FILE" | grep -c "^\*\*" || echo 0),
    "medium": $(grep -A20 "### 🟡 Medium" "$QUEUE_FILE" | grep -c "^\*\*" || echo 0),
    "low": $(grep -A20 "### 🟢 Low" "$QUEUE_FILE" | grep -c "^\*\*" || echo 0)
  }
}
EOF
            ;;
    esac
}
```

---

## Mark Active

Move concept from queue to active implementations.

### Usage
```bash
mark_active "concept-name" "owner"
```

### Implementation
```bash
mark_active() {
    local CONCEPT_NAME="$1"
    local OWNER="$2"
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Get focus group from queue entry
    FOCUS_GROUP=$(grep "\*\*${CONCEPT_NAME}\*\*" "$QUEUE_FILE" | sed 's/.*(\([^)]*\)).*/\1/')
    
    # Remove from queue
    remove_from_queue "$CONCEPT_NAME" "started"
    
    # Add to active section
    IMPL_NAME="${CONCEPT_NAME}-impl"
    ACTIVE_ENTRY="| ${IMPL_NAME} | ${FOCUS_GROUP} | @${OWNER} | initializing | 0 |"
    
    # Insert into active table
    sed -i "/## Active Implementations/,/^$/{
        /^\| Implementation/a\\
${ACTIVE_ENTRY}
    }" "$QUEUE_FILE"
    
    update_queue_stats
    
    echo "✅ ${CONCEPT_NAME} marked active with owner @${OWNER}"
}
```

---

## Mark Complete

Move implementation from active to completed.

### Usage
```bash
mark_complete "implementation-name"
```

### Implementation
```bash
mark_complete() {
    local IMPL_NAME="$1"
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%d")
    
    # Get implementation details from active section
    IMPL_LINE=$(grep "${IMPL_NAME}" "$QUEUE_FILE" | head -1)
    
    if [ -z "$IMPL_LINE" ]; then
        echo "❌ Implementation not found: ${IMPL_NAME}"
        return 1
    fi
    
    # Extract details
    FOCUS_GROUP=$(echo "$IMPL_LINE" | awk -F'|' '{print $3}' | xargs)
    OWNER=$(echo "$IMPL_LINE" | awk -F'|' '{print $4}' | xargs)
    DAYS=$(echo "$IMPL_LINE" | awk -F'|' '{print $6}' | xargs)
    
    # Remove from active
    sed -i "/${IMPL_NAME}/d" "$QUEUE_FILE"
    
    # Add to completed
    COMPLETED_ENTRY="| ${IMPL_NAME} | ${FOCUS_GROUP} | ${OWNER} | ${TIMESTAMP} | ${DAYS} days |"
    
    # Insert into completed table (at the top of completed list)
    sed -i "/## Recently Completed/,/^$/{
        /^\| Implementation | Focus Group | Owner | Completed/a\\
${COMPLETED_ENTRY}
    }" "$QUEUE_FILE"
    
    update_queue_stats
    
    echo "✅ ${IMPL_NAME} marked complete"
}
```

---

## Initialize Queue File

Create the queue file if it doesn't exist.

### Implementation
```bash
initialize_queue() {
    local QUEUE_FILE=".planning/IMPLEMENTATION-QUEUE.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    if [ -f "$QUEUE_FILE" ]; then
        echo "Queue file already exists"
        return 0
    fi
    
    cat > "$QUEUE_FILE" << 'EOF'
# Implementation Queue

**Last Updated:** TIMESTAMP
**Queue Length:** 0
**Active Implementations:** 0/4

## Active Implementations

| Implementation | Focus Group | Owner | Status | Days |
|----------------|-------------|-------|--------|------|

## Queue

### 🔴 Critical
*Empty*

### 🟠 High
*Empty*

### 🟡 Medium  
*Empty*

### 🟢 Low
*Empty*

## Recently Completed

| Implementation | Focus Group | Owner | Completed | Duration |
|----------------|-------------|-------|-----------|----------|

EOF
    
    sed -i "s/TIMESTAMP/${TIMESTAMP}/" "$QUEUE_FILE"
    
    echo "✅ Queue file initialized: $QUEUE_FILE"
}
```

---

## Integration Notes

### With promote-concept.md
```bash
# After approval, add to queue
add_to_queue "$CONCEPT" "$FG" "medium"
```

### With create-implementation.md
```bash
# Before creating implementation
mark_active "$CONCEPT" "$OWNER"
```

### With Canvas Dashboard
```bash
# Get queue for canvas update
get_queue_summary "compact" | sync_to_canvas "master-dashboard"
```

---

*Library created for Phase 5: Workflow Engine*
*Supports WORKFLOW-04: Implementation prioritization system*
