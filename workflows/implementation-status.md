---
name: wgsd:implementation-status
description: Query and display implementation status with progress tracking
argument-hint: "[implementation-name|all]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Provide comprehensive status reporting for WGSD implementations:
- Query individual implementation status
- Show all active implementations dashboard
- Generate daily status summaries
- Track blockers across implementations
- Display progress visualizations
</objective>

<libraries>
- workflows/lib/priority-queue.md - Queue data
- workflows/lib/canvas-api.md - Dashboard updates
</libraries>

<process>

## Action: Query Single Implementation

```bash
IMPLEMENTATION_NAME="{implementation-name}"

echo "📊 Implementation Status: ${IMPLEMENTATION_NAME}"
echo ""

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
if [ ! -d "$IMPL_DIR" ]; then
    # Check completed
    COMPLETED_DIR=".planning/completed-implementations/${IMPLEMENTATION_NAME}"
    if [ -d "$COMPLETED_DIR" ]; then
        echo "✅ This implementation is COMPLETE"
        echo ""
        cat "${COMPLETED_DIR}/STATE.md" 2>/dev/null
        exit 0
    fi
    
    echo "❌ Implementation not found: ${IMPLEMENTATION_NAME}"
    exit 1
fi

# Parse state file
STATE_FILE="${IMPL_DIR}/STATE.md"
ROADMAP_FILE="${IMPL_DIR}/ROADMAP.md"

OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: @*//' | xargs)
STATUS=$(grep "^Status:" "$STATE_FILE" 2>/dev/null | sed 's/Status: //' | xargs)
FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //' | xargs)
PROGRESS=$(grep "^Progress:" "$STATE_FILE" 2>/dev/null | sed 's/Progress: //' | xargs)
CURRENT_PHASE=$(grep "^Current Phase:" "$STATE_FILE" 2>/dev/null | sed 's/Current Phase: //' | xargs)
STARTED=$(grep "^Started:" "$STATE_FILE" 2>/dev/null | sed 's/Started: //' | xargs)
LAST_UPDATE=$(grep "^Last Update:" "$STATE_FILE" 2>/dev/null | sed 's/Last Update: //' | xargs)

# Calculate days active
if [ -n "$STARTED" ]; then
    START_EPOCH=$(date -d "$STARTED" +%s 2>/dev/null || echo "0")
    NOW_EPOCH=$(date +%s)
    DAYS_ACTIVE=$(( (NOW_EPOCH - START_EPOCH) / 86400 ))
else
    DAYS_ACTIVE="?"
fi

# Count phases
TOTAL_PHASES=$(grep -c "^## Phase" "$ROADMAP_FILE" 2>/dev/null || echo "?")
COMPLETED_PHASES=$(grep -c "✅ Complete" "$ROADMAP_FILE" 2>/dev/null || echo "0")

# Progress bar
PROGRESS_NUM="${PROGRESS:-0}"
PROGRESS_NUM="${PROGRESS_NUM%\%}"
BAR_WIDTH=20
FILLED=$((PROGRESS_NUM * BAR_WIDTH / 100))
EMPTY=$((BAR_WIDTH - FILLED))
PROGRESS_BAR=$(printf '%*s' "$FILLED" '' | tr ' ' '█')$(printf '%*s' "$EMPTY" '' | tr ' ' '░')

echo "┌────────────────────────────────────────────────────┐"
echo "│ ${IMPLEMENTATION_NAME}"
echo "├────────────────────────────────────────────────────┤"
echo "│ Owner:       @${OWNER:-unassigned}"
echo "│ Focus Group: ${FOCUS_GROUP:-unknown}"
echo "│ Status:      ${STATUS:-unknown}"
echo "│ Phase:       ${CURRENT_PHASE:-1}/${TOTAL_PHASES} (${COMPLETED_PHASES} complete)"
echo "│ Progress:    [${PROGRESS_BAR}] ${PROGRESS_NUM}%"
echo "│ Days Active: ${DAYS_ACTIVE}"
echo "│ Last Update: ${LAST_UPDATE:-never}"
echo "└────────────────────────────────────────────────────┘"

# Check for blockers
if grep -q "🚫 Blocked" "$STATE_FILE"; then
    echo ""
    echo "⚠️ BLOCKED:"
    echo ""
    grep -A 5 "## Blocker" "$STATE_FILE" 2>/dev/null | tail -n +3 | head -5
fi

# Show recent progress
PROGRESS_LOG="${IMPL_DIR}/PROGRESS.md"
if [ -f "$PROGRESS_LOG" ]; then
    echo ""
    echo "📝 Recent Updates:"
    tail -20 "$PROGRESS_LOG" | head -15
fi
```

---

## Action: Show All Implementations

```bash
echo "📊 Active Implementations Dashboard"
echo ""
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Count implementations
ACTIVE_COUNT=0
BLOCKED_COUNT=0
COMPLETED_COUNT=0

if [ -d .planning/active-implementations ]; then
    for impl_dir in .planning/active-implementations/*/; do
        [ -d "$impl_dir" ] || continue
        ACTIVE_COUNT=$((ACTIVE_COUNT + 1))
        
        STATE_FILE="${impl_dir}STATE.md"
        if grep -q "🚫 Blocked" "$STATE_FILE" 2>/dev/null; then
            BLOCKED_COUNT=$((BLOCKED_COUNT + 1))
        fi
    done
fi

if [ -d .planning/completed-implementations ]; then
    COMPLETED_COUNT=$(find .planning/completed-implementations -maxdepth 1 -type d | wc -l)
    COMPLETED_COUNT=$((COMPLETED_COUNT - 1))
fi

# Summary header
echo "┌─────────────────────────────────────────────────────────────────┐"
echo "│                 WGSD Implementation Status                     │"
echo "│                 Generated: ${TIMESTAMP}                        │"
echo "├─────────────────────────────────────────────────────────────────┤"
echo "│ Active: ${ACTIVE_COUNT}/4    Blocked: ${BLOCKED_COUNT}    Completed: ${COMPLETED_COUNT}            │"
echo "└─────────────────────────────────────────────────────────────────┘"
echo ""

if [ "$ACTIVE_COUNT" -eq 0 ]; then
    echo "No active implementations."
    echo ""
    echo "💡 Start an implementation with:"
    echo "   /wgsd create implementation [concept-name]"
    exit 0
fi

# Table header
echo "┌──────────────────────┬─────────────┬──────────┬────────┬──────┬───────────┐"
echo "│ Implementation       │ Focus Group │ Owner    │ Status │ Prog │ Days      │"
echo "├──────────────────────┼─────────────┼──────────┼────────┼──────┼───────────┤"

for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    
    IMPL_NAME=$(basename "$impl_dir")
    STATE_FILE="${impl_dir}STATE.md"
    
    OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: @*//' | head -c8)
    STATUS=$(grep "^Status:" "$STATE_FILE" 2>/dev/null | sed 's/Status: //' | head -c8)
    FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //' | head -c11)
    PROGRESS=$(grep "^Progress:" "$STATE_FILE" 2>/dev/null | sed 's/Progress: //' | head -c4)
    STARTED=$(grep "^Started:" "$STATE_FILE" 2>/dev/null | sed 's/Started: //')
    
    # Calculate days
    if [ -n "$STARTED" ]; then
        START_EPOCH=$(date -d "$STARTED" +%s 2>/dev/null || echo "0")
        NOW_EPOCH=$(date +%s)
        DAYS=$((( NOW_EPOCH - START_EPOCH) / 86400))
    else
        DAYS="?"
    fi
    
    # Pad fields
    printf "│ %-20s │ %-11s │ %-8s │ %-6s │ %4s │ %3s days  │\n" \
        "${IMPL_NAME:0:20}" "${FOCUS_GROUP:0:11}" "${OWNER:0:8}" \
        "${STATUS:0:6}" "${PROGRESS:-0%}" "$DAYS"
done

echo "└──────────────────────┴─────────────┴──────────┴────────┴──────┴───────────┘"

# Show blockers if any
if [ "$BLOCKED_COUNT" -gt 0 ]; then
    echo ""
    echo "⚠️ Blocked Implementations:"
    echo ""
    for impl_dir in .planning/active-implementations/*/; do
        [ -d "$impl_dir" ] || continue
        STATE_FILE="${impl_dir}STATE.md"
        if grep -q "🚫 Blocked" "$STATE_FILE" 2>/dev/null; then
            IMPL_NAME=$(basename "$impl_dir")
            BLOCKER=$(grep -A 3 "## Blocker" "$STATE_FILE" 2>/dev/null | tail -1)
            echo "  • ${IMPL_NAME}: ${BLOCKER:0:50}..."
        fi
    done
fi

# Show queue
echo ""
echo "📋 Implementation Queue:"
if [ -f .planning/IMPLEMENTATION-QUEUE.md ]; then
    grep -A 20 "^## Queue" .planning/IMPLEMENTATION-QUEUE.md 2>/dev/null | \
        grep -E "^\d+\.|^### " | head -10
else
    echo "  (empty)"
fi
```

---

## Action: Daily Summary

Generate daily summary for team standup:

```bash
echo "📅 Daily Implementation Summary"
echo ""
TODAY=$(date +%Y-%m-%d)
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d 2>/dev/null || date -v-1d +%Y-%m-%d)

echo "## WGSD Daily Summary - ${TODAY}"
echo ""

# Yesterday's completions
echo "### ✅ Completed Yesterday"
echo ""
FOUND_COMPLETIONS=0
if [ -d .planning/completed-implementations ]; then
    for impl_dir in .planning/completed-implementations/*/; do
        [ -d "$impl_dir" ] || continue
        STATE_FILE="${impl_dir}STATE.md"
        if grep -q "Completed: ${YESTERDAY}" "$STATE_FILE" 2>/dev/null; then
            IMPL_NAME=$(basename "$impl_dir")
            echo "- ${IMPL_NAME}"
            FOUND_COMPLETIONS=1
        fi
    done
fi
[ "$FOUND_COMPLETIONS" -eq 0 ] && echo "_None_"
echo ""

# Today's focus
echo "### 🎯 Today's Focus"
echo ""
for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    IMPL_NAME=$(basename "$impl_dir")
    STATE_FILE="${impl_dir}STATE.md"
    OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: //')
    CURRENT_PHASE=$(grep "^Current Phase:" "$STATE_FILE" 2>/dev/null | sed 's/Current Phase: //')
    STATUS=$(grep "^Status:" "$STATE_FILE" 2>/dev/null | sed 's/Status: //')
    
    echo "- **${IMPL_NAME}** (${OWNER})"
    echo "  - Phase ${CURRENT_PHASE:-1}, Status: ${STATUS:-implementing}"
done
echo ""

# Blockers
echo "### ⚠️ Blockers"
echo ""
FOUND_BLOCKERS=0
for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    STATE_FILE="${impl_dir}STATE.md"
    if grep -q "🚫 Blocked" "$STATE_FILE" 2>/dev/null; then
        IMPL_NAME=$(basename "$impl_dir")
        BLOCKER=$(grep -A 3 "## Blocker" "$STATE_FILE" 2>/dev/null | tail -1)
        echo "- **${IMPL_NAME}**: ${BLOCKER}"
        FOUND_BLOCKERS=1
    fi
done
[ "$FOUND_BLOCKERS" -eq 0 ] && echo "_No blockers!_ 🎉"
echo ""

# Queue preview
echo "### 📋 Up Next"
echo ""
if [ -f .planning/IMPLEMENTATION-QUEUE.md ]; then
    grep -E "^\d+\. " .planning/IMPLEMENTATION-QUEUE.md 2>/dev/null | head -3 || echo "_Queue empty_"
else
    echo "_Queue empty_"
fi
```

---

## Action: Progress Report

Generate detailed progress report for stakeholders:

```bash
IMPLEMENTATION_NAME="{implementation-name}"

echo "📈 Progress Report: ${IMPLEMENTATION_NAME}"
echo ""

IMPL_DIR=".planning/active-implementations/${IMPLEMENTATION_NAME}"
if [ ! -d "$IMPL_DIR" ]; then
    echo "❌ Implementation not found"
    exit 1
fi

STATE_FILE="${IMPL_DIR}/STATE.md"
ROADMAP_FILE="${IMPL_DIR}/ROADMAP.md"
REQUIREMENTS_FILE="${IMPL_DIR}/REQUIREMENTS.md"

OWNER=$(grep "^Owner:" "$STATE_FILE" 2>/dev/null | sed 's/Owner: //')
FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //')
STARTED=$(grep "^Started:" "$STATE_FILE" 2>/dev/null | sed 's/Started: //')
PROGRESS=$(grep "^Progress:" "$STATE_FILE" 2>/dev/null | sed 's/Progress: //')

echo "## ${IMPLEMENTATION_NAME} Progress Report"
echo ""
echo "**Generated:** $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
echo "**Owner:** ${OWNER}"
echo "**Focus Group:** ${FOCUS_GROUP}"
echo "**Started:** ${STARTED}"
echo "**Progress:** ${PROGRESS:-0%}"
echo ""

# Phases overview
echo "### Phase Status"
echo ""
echo "| Phase | Name | Status |"
echo "|-------|------|--------|"
grep "^## Phase" "$ROADMAP_FILE" 2>/dev/null | while read -r line; do
    PHASE_NUM=$(echo "$line" | sed 's/## Phase \([0-9]*\):.*/\1/')
    PHASE_NAME=$(echo "$line" | sed 's/## Phase [0-9]*: //')
    
    if grep -q "| ${PHASE_NUM} |.*Complete" "$ROADMAP_FILE" 2>/dev/null; then
        echo "| ${PHASE_NUM} | ${PHASE_NAME} | ✅ Complete |"
    elif grep -q "| ${PHASE_NUM} |.*In Progress" "$ROADMAP_FILE" 2>/dev/null; then
        echo "| ${PHASE_NUM} | ${PHASE_NAME} | 🔄 In Progress |"
    else
        echo "| ${PHASE_NUM} | ${PHASE_NAME} | ⏳ Not Started |"
    fi
done
echo ""

# Requirements checklist
echo "### Requirements Coverage"
echo ""
if [ -f "$REQUIREMENTS_FILE" ]; then
    grep -E "^\- \[.\]" "$REQUIREMENTS_FILE" 2>/dev/null | head -15
fi
echo ""

# Recent activity
echo "### Recent Activity"
echo ""
PROGRESS_LOG="${IMPL_DIR}/PROGRESS.md"
if [ -f "$PROGRESS_LOG" ]; then
    tail -30 "$PROGRESS_LOG" | head -20
else
    echo "_No progress updates recorded_"
fi
```

---

## Action: Compare Implementations

Compare status across implementations:

```bash
echo "📊 Implementation Comparison"
echo ""

echo "| Implementation | Progress | Phase | Days | Velocity |"
echo "|----------------|----------|-------|------|----------|"

for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    
    IMPL_NAME=$(basename "$impl_dir")
    STATE_FILE="${impl_dir}STATE.md"
    ROADMAP_FILE="${impl_dir}ROADMAP.md"
    
    PROGRESS=$(grep "^Progress:" "$STATE_FILE" 2>/dev/null | sed 's/Progress: //' || echo "0%")
    CURRENT_PHASE=$(grep "^Current Phase:" "$STATE_FILE" 2>/dev/null | sed 's/Current Phase: //' || echo "1")
    TOTAL_PHASES=$(grep -c "^## Phase" "$ROADMAP_FILE" 2>/dev/null || echo "3")
    STARTED=$(grep "^Started:" "$STATE_FILE" 2>/dev/null | sed 's/Started: //')
    
    # Calculate days and velocity
    if [ -n "$STARTED" ]; then
        START_EPOCH=$(date -d "$STARTED" +%s 2>/dev/null || echo "0")
        NOW_EPOCH=$(date +%s)
        DAYS=$(( (NOW_EPOCH - START_EPOCH) / 86400 ))
        DAYS=$((DAYS + 1))  # Avoid div by zero
        
        PROG_NUM="${PROGRESS%\%}"
        VELOCITY=$((PROG_NUM / DAYS))
    else
        DAYS="?"
        VELOCITY="?"
    fi
    
    echo "| ${IMPL_NAME:0:14} | ${PROGRESS:0:8} | ${CURRENT_PHASE}/${TOTAL_PHASES} | ${DAYS} | ${VELOCITY}%/day |"
done
```

</process>

<routing>
| Query | Action |
|-------|--------|
| [name] | Show single implementation status |
| all, status, list | Show all implementations |
| daily, standup | Generate daily summary |
| report [name] | Detailed progress report |
| compare | Compare all implementations |
</routing>

<success_criteria>
- Status accurately reflects implementation state
- Progress percentages displayed with visualizations
- Blockers prominently highlighted
- Daily summaries ready for standup use
- Queue visibility maintained
</success_criteria>
