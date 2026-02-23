# Approval System Library

**Purpose:** Voting and approval tracking for concept maturation in WGSD

---

## Overview

The approval system manages concept state transitions through voting:
- Track approvals from focus group members
- Enforce minimum approval thresholds
- Support both emoji reactions and explicit commands
- Manage voting windows and expiration

---

## Concept States

```
📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready for Implementation → 🔧 In Implementation → ✔️ Complete
```

### State Transitions

| From | To | Approval Required |
|------|----|-------------------|
| Draft | Exploring | None (author decision) |
| Exploring | Mature | Informal consensus |
| Mature | Ready | **2+ explicit approvals** |
| Ready | In Implementation | Owner assignment |
| In Implementation | Complete | Code merged |

---

## Approval Configuration

```yaml
# Default approval settings
approval:
  min_votes: 2                    # Minimum approvals required
  voting_window_hours: 24         # Hours to collect votes
  allow_author_approval: false    # Author cannot self-approve
  approval_emojis: ["👍", "✅", "+1"]
  rejection_emojis: ["👎", "❌", "-1"]
  expedite_threshold: 3           # Votes to skip waiting period
```

---

## Approval File Structure

Each concept tracks approval status in its markdown file:

```markdown
## Approval Status

**Current State:** ✅ Mature
**Votes to Advance:** 1/2
**Voting Started:** 2026-02-22T10:00:00Z
**Voting Ends:** 2026-02-23T10:00:00Z

### Votes

| User | Vote | Comment | Timestamp |
|------|------|---------|-----------|
| @alice | 👍 | LGTM | 2026-02-22T11:30:00Z |

### Vote History
- 2026-02-20: Advanced to "Exploring"
- 2026-02-22: Advanced to "Mature", voting opened
```

---

## Record Vote

Record a vote for a concept.

### Usage
```bash
record_vote "concept-name" "focus-group" "user" "approve|reject" "optional comment"
```

### Implementation
```bash
record_vote() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local USER="$3"
    local VOTE_TYPE="$4"
    local COMMENT="${5:-}"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    
    if [ ! -f "$CONCEPT_FILE" ]; then
        echo "❌ Concept not found: ${CONCEPT_NAME}"
        return 1
    fi
    
    # Check if user already voted
    if grep -q "| @${USER} |" "$CONCEPT_FILE"; then
        echo "⚠️ User @${USER} has already voted on this concept"
        return 1
    fi
    
    # Check if author is trying to self-approve
    AUTHOR=$(grep "Creator:" "$CONCEPT_FILE" | sed 's/.*: @*//')
    if [ "$USER" = "$AUTHOR" ] && [ "$VOTE_TYPE" = "approve" ]; then
        echo "⚠️ Authors cannot approve their own concepts"
        return 1
    fi
    
    # Determine vote emoji
    local VOTE_EMOJI="👍"
    if [ "$VOTE_TYPE" = "reject" ]; then
        VOTE_EMOJI="👎"
    fi
    
    # Add vote to table
    local VOTE_LINE="| @${USER} | ${VOTE_EMOJI} | ${COMMENT} | ${TIMESTAMP} |"
    
    # Insert vote after the table header
    sed -i "/| User | Vote | Comment | Timestamp |/a\\
${VOTE_LINE}" "$CONCEPT_FILE"
    
    echo "✅ Vote recorded: @${USER} ${VOTE_TYPE}d ${CONCEPT_NAME}"
    
    # Check if approval threshold reached
    check_approval_status "$CONCEPT_NAME" "$FOCUS_GROUP"
}
```

---

## Check Approval Status

Calculate current approval status for a concept.

### Usage
```bash
check_approval_status "concept-name" "focus-group"
```

### Implementation
```bash
check_approval_status() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    
    # Count approvals (👍 or ✅)
    APPROVE_COUNT=$(grep -E "\| @.+ \| (👍|✅|\+1) \|" "$CONCEPT_FILE" | wc -l)
    
    # Count rejections (👎 or ❌)
    REJECT_COUNT=$(grep -E "\| @.+ \| (👎|❌|\-1) \|" "$CONCEPT_FILE" | wc -l)
    
    # Get configuration
    MIN_VOTES=2  # Default
    
    echo "📊 Approval Status: ${APPROVE_COUNT}/${MIN_VOTES} approvals, ${REJECT_COUNT} rejections"
    
    if [ "$APPROVE_COUNT" -ge "$MIN_VOTES" ]; then
        echo "✅ Approval threshold reached!"
        return 0
    elif [ "$REJECT_COUNT" -gt 0 ]; then
        echo "⚠️ Has rejections - needs discussion"
        return 2
    else
        REMAINING=$((MIN_VOTES - APPROVE_COUNT))
        echo "⏳ Needs ${REMAINING} more approval(s)"
        return 1
    fi
}
```

### Return Values
| Code | Meaning |
|------|---------|
| 0 | Approved (threshold reached) |
| 1 | Pending (needs more votes) |
| 2 | Has rejections |

---

## Get Vote Count

Get detailed vote counts for a concept.

### Usage
```bash
get_vote_count "concept-name" "focus-group"
```

### Implementation
```bash
get_vote_count() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    
    if [ ! -f "$CONCEPT_FILE" ]; then
        echo '{"error": "concept_not_found"}'
        return 1
    fi
    
    APPROVE_COUNT=$(grep -E "\| @.+ \| (👍|✅|\+1) \|" "$CONCEPT_FILE" | wc -l)
    REJECT_COUNT=$(grep -E "\| @.+ \| (👎|❌|\-1) \|" "$CONCEPT_FILE" | wc -l)
    TOTAL_COUNT=$((APPROVE_COUNT + REJECT_COUNT))
    
    # Get voter names
    VOTERS=$(grep -E "\| @.+ \| (👍|✅|👎|❌) \|" "$CONCEPT_FILE" | sed 's/| @\([^ ]*\) |.*/\1/' | tr '\n' ',' | sed 's/,$//')
    
    cat <<EOF
{
  "concept": "${CONCEPT_NAME}",
  "focus_group": "${FOCUS_GROUP}",
  "approvals": ${APPROVE_COUNT},
  "rejections": ${REJECT_COUNT},
  "total": ${TOTAL_COUNT},
  "threshold": 2,
  "approved": $([ "$APPROVE_COUNT" -ge 2 ] && echo "true" || echo "false"),
  "voters": "${VOTERS}"
}
EOF
}
```

---

## Initialize Voting

Start the voting process for a concept.

### Usage
```bash
initialize_voting "concept-name" "focus-group"
```

### Implementation
```bash
initialize_voting() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local VOTING_HOURS="${3:-24}"
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    local NOW=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    local END=$(date -u -d "+${VOTING_HOURS} hours" +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null || date -u -v+${VOTING_HOURS}H +"%Y-%m-%dT%H:%M:%SZ")
    
    # Check current state
    CURRENT_STATE=$(grep "Status:" "$CONCEPT_FILE" | head -1 | sed 's/.*Status: //')
    
    if [[ "$CURRENT_STATE" != *"Mature"* ]]; then
        echo "❌ Concept must be in 'Mature' state to start voting"
        echo "   Current state: $CURRENT_STATE"
        return 1
    fi
    
    # Add approval section if not exists
    if ! grep -q "## Approval Status" "$CONCEPT_FILE"; then
        cat >> "$CONCEPT_FILE" << EOF

## Approval Status

**Current State:** ✅ Mature
**Votes to Advance:** 0/2
**Voting Started:** ${NOW}
**Voting Ends:** ${END}

### Votes

| User | Vote | Comment | Timestamp |
|------|------|---------|-----------|

### Vote History
- ${NOW}: Voting opened for promotion to "Ready for Implementation"
EOF
    fi
    
    echo "🗳️ Voting initialized for ${CONCEPT_NAME}"
    echo "   Voting ends: ${END}"
    echo "   Required: 2 approvals"
}
```

---

## Advance State

Advance concept to next state after approval.

### Usage
```bash
advance_state "concept-name" "focus-group" "new-state"
```

### Implementation
```bash
advance_state() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local NEW_STATE="$3"
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Valid state transitions
    case "$NEW_STATE" in
        "🔍 Exploring"|"exploring")
            NEW_STATE="🔍 Exploring"
            ;;
        "✅ Mature"|"mature")
            NEW_STATE="✅ Mature"
            ;;
        "🚀 Ready for Implementation"|"ready")
            # Check approvals
            check_approval_status "$CONCEPT_NAME" "$FOCUS_GROUP"
            if [ $? -ne 0 ]; then
                echo "❌ Cannot advance: insufficient approvals"
                return 1
            fi
            NEW_STATE="🚀 Ready for Implementation"
            ;;
        "🔧 In Implementation"|"implementing")
            NEW_STATE="🔧 In Implementation"
            ;;
        "✔️ Complete"|"complete")
            NEW_STATE="✔️ Complete"
            ;;
        *)
            echo "❌ Invalid state: $NEW_STATE"
            return 1
            ;;
    esac
    
    # Update status in file
    sed -i "s/\*\*Status:\*\* .*/\*\*Status:\*\* ${NEW_STATE}/" "$CONCEPT_FILE"
    
    # Add to vote history
    echo "- ${TIMESTAMP}: Advanced to \"${NEW_STATE}\"" >> "$CONCEPT_FILE"
    
    echo "✅ ${CONCEPT_NAME} advanced to: ${NEW_STATE}"
}
```

---

## Notify Voters

Send notification about voting.

### Usage
```bash
notify_voters "concept-name" "focus-group" "message-type"
```

### Implementation
```bash
notify_voters() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    local MESSAGE_TYPE="$3"
    
    local REPO_NAME=$(basename $(pwd))
    local CHANNEL="${REPO_NAME}-${FOCUS_GROUP}"
    
    case "$MESSAGE_TYPE" in
        voting_opened)
            MESSAGE="🗳️ **Voting Opened: ${CONCEPT_NAME}**

The concept \"${CONCEPT_NAME}\" is ready for promotion voting.

**Required:** 2 approvals to advance to 'Ready for Implementation'
**Voting Window:** 24 hours

**To approve:**
- React with 👍 to this message, OR
- Run \`/wgsd approve ${CONCEPT_NAME}\`

**To review:**
- View concept: \`.planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md\`

*Focus group members: please review and vote!*"
            ;;
        approval_reached)
            MESSAGE="✅ **Concept Approved: ${CONCEPT_NAME}**

The concept has received sufficient approvals!

**Next step:** Assign an owner to begin implementation
- \`/wgsd assign ${CONCEPT_NAME} @user\`
- Or claim it yourself: \`/wgsd claim ${CONCEPT_NAME}\`

*Congrats team on maturing this concept!*"
            ;;
        voting_expired)
            MESSAGE="⏰ **Voting Expired: ${CONCEPT_NAME}**

The voting window has closed without reaching approval threshold.

**Options:**
- Continue discussion and re-open voting later
- Archive the concept for future consideration

*Use \`/wgsd reopen-voting ${CONCEPT_NAME}\` to try again.*"
            ;;
    esac
    
    echo "$MESSAGE"
    # Would send via message tool in actual workflow
}
```

---

## Handle Emoji Reaction

Process emoji reaction as vote.

### Usage
```bash
handle_reaction "user" "emoji" "message-context"
```

### Implementation
```bash
handle_reaction() {
    local USER="$1"
    local EMOJI="$2"
    local CONTEXT="$3"  # Should contain concept and focus group info
    
    # Parse context to get concept name and focus group
    # Context format: "concept:auth-v2,fg:security"
    CONCEPT_NAME=$(echo "$CONTEXT" | grep -o 'concept:[^,]*' | sed 's/concept://')
    FOCUS_GROUP=$(echo "$CONTEXT" | grep -o 'fg:[^,]*' | sed 's/fg://')
    
    if [ -z "$CONCEPT_NAME" ] || [ -z "$FOCUS_GROUP" ]; then
        echo "❌ Cannot determine concept from context"
        return 1
    fi
    
    # Map emoji to vote type
    case "$EMOJI" in
        👍|✅|+1|:thumbsup:|:white_check_mark:)
            record_vote "$CONCEPT_NAME" "$FOCUS_GROUP" "$USER" "approve" "via reaction"
            ;;
        👎|❌|-1|:thumbsdown:|:x:)
            record_vote "$CONCEPT_NAME" "$FOCUS_GROUP" "$USER" "reject" "via reaction"
            ;;
        *)
            # Not an approval emoji, ignore
            return 0
            ;;
    esac
}
```

---

## Check Voting Expiration

Check if voting window has expired.

### Usage
```bash
check_voting_expiration "concept-name" "focus-group"
```

### Implementation
```bash
check_voting_expiration() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    
    local CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md"
    
    # Get voting end time
    VOTING_ENDS=$(grep "Voting Ends:" "$CONCEPT_FILE" | sed 's/.*: //')
    
    if [ -z "$VOTING_ENDS" ]; then
        echo "no_voting"
        return 0
    fi
    
    NOW=$(date -u +%s)
    END=$(date -d "$VOTING_ENDS" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%SZ" "$VOTING_ENDS" +%s 2>/dev/null)
    
    if [ "$NOW" -gt "$END" ]; then
        echo "expired"
        return 1
    else
        REMAINING=$(( (END - NOW) / 3600 ))
        echo "active:${REMAINING}h"
        return 0
    fi
}
```

---

## Get Approval Summary

Get human-readable approval summary for a concept.

### Usage
```bash
get_approval_summary "concept-name" "focus-group"
```

### Implementation
```bash
get_approval_summary() {
    local CONCEPT_NAME="$1"
    local FOCUS_GROUP="$2"
    
    VOTE_JSON=$(get_vote_count "$CONCEPT_NAME" "$FOCUS_GROUP")
    
    APPROVALS=$(echo "$VOTE_JSON" | jq -r '.approvals')
    REJECTIONS=$(echo "$VOTE_JSON" | jq -r '.rejections')
    APPROVED=$(echo "$VOTE_JSON" | jq -r '.approved')
    VOTERS=$(echo "$VOTE_JSON" | jq -r '.voters')
    
    if [ "$APPROVED" = "true" ]; then
        STATUS="✅ Approved"
    elif [ "$REJECTIONS" -gt 0 ]; then
        STATUS="⚠️ Has Objections"
    elif [ "$APPROVALS" -gt 0 ]; then
        STATUS="⏳ Pending (${APPROVALS}/2)"
    else
        STATUS="🗳️ No Votes Yet"
    fi
    
    cat <<EOF
**Concept:** ${CONCEPT_NAME}
**Focus Group:** ${FOCUS_GROUP}
**Status:** ${STATUS}
**Votes:** ${APPROVALS} approve, ${REJECTIONS} reject
**Voters:** ${VOTERS:-none}
EOF
}
```

---

## Integration Notes

### With promote-concept.md
```bash
# Before promoting, check approval
check_approval_status "$CONCEPT" "$FG"
if [ $? -ne 0 ]; then
    echo "Cannot promote - needs more approvals"
    exit 1
fi
```

### With Canvas Sync
When concept state changes, trigger canvas update:
```bash
# After advancing state
advance_state "$CONCEPT" "$FG" "ready"
# Trigger sync
# /wgsd sync-canvas "$FG"
```

---

## Matrix-Based Approval System (Phase 12)

The following functions support the matrix-based approval system where each impacted focus group independently approves a concept.

---

### approval_matrix_summary

Get a human-readable summary of approval matrix status.

```bash
# Usage: approval_matrix_summary <concept_path>
# Returns: Formatted approval matrix summary
approval_matrix_summary() {
  local concept_path="$1"
  local impact_file="$concept_path/impact-matrix.md"
  
  if [ ! -f "$impact_file" ]; then
    echo "ERROR: Impact matrix not found"
    return 1
  fi
  
  local matrix=$(approval_matrix_get "$impact_file")
  
  python3 -c "
import json

matrix = json.loads('''$matrix''')

concept = matrix.get('concept', 'unknown')
approved = matrix['approved']
total = matrix['total_approvals']
pending = matrix['pending']
rejected = matrix['rejected']
blocked = matrix['blocked']
completion = matrix['completion_percent']
fully = matrix['fully_approved']

status_emoji = {
    'pending': '⏳',
    'approved': '✅',
    'rejected': '🚫',
    'blocked': '⏸️'
}

priority_emoji = {
    'P0': '🔴',
    'P1': '🟠',
    'P2': '🟡',
    'P3': '🟢'
}

print(f'📊 **Approval Matrix: {concept}**')
print()

if fully:
    print('✅ **FULLY APPROVED** - Ready for implementation!')
else:
    print(f'**Progress:** {approved}/{total} ({completion}%)')

print()
print('| Focus Group | Priority | Status | Approver | Date |')
print('|-------------|----------|--------|----------|------|')

for impact in matrix['details']:
    fg = impact.get('focus_group', '?')
    priority = impact.get('priority', '?')
    status = impact.get('status', 'pending')
    approver = impact.get('approver', '—')
    date = impact.get('approved_date', '—')
    blocked_by = impact.get('blocked_by')
    
    p_icon = priority_emoji.get(priority, '⚪')
    s_icon = status_emoji.get(status, '❓')
    
    status_display = f'{s_icon} {status.title()}'
    if blocked_by:
        status_display = f'⏸️ Blocked by {blocked_by}'
    
    print(f'| {fg} | {p_icon} {priority} | {status_display} | {approver or \"—\"} | {date or \"—\"} |')

if matrix['blocking_fgs']:
    print()
    print(f'⚠️ **Blocking Dependencies:** {', '.join(matrix[\"blocking_fgs\"])}')

if rejected > 0:
    print()
    print(f'🚫 **Has Rejections:** Review feedback before proceeding')
"
  return 0
}
```

---

### approval_matrix_check_completion

Comprehensive check for concept completion with detailed status.

```bash
# Usage: approval_matrix_check_completion <concept_path>
# Returns: JSON with completion status and next actions
approval_matrix_check_completion() {
  local concept_path="$1"
  local impact_file="$concept_path/impact-matrix.md"
  
  if [ ! -f "$impact_file" ]; then
    echo '{"error": "Impact matrix not found"}'
    return 1
  fi
  
  local matrix=$(approval_matrix_get "$impact_file")
  
  python3 -c "
import json

matrix = json.loads('''$matrix''')

approved = matrix['approved']
total = matrix['total_approvals']
pending = matrix['pending']
rejected = matrix['rejected']
blocked = matrix['blocked']
fully = matrix['fully_approved']

result = {
    'is_complete': fully,
    'progress': f'{approved}/{total}',
    'completion_percent': matrix['completion_percent'],
    'can_implement': fully,
    'blockers': [],
    'next_actions': []
}

# Identify blockers
for impact in matrix['details']:
    status = impact.get('status')
    fg = impact.get('focus_group')
    
    if status == 'pending':
        result['blockers'].append({
            'type': 'pending_approval',
            'focus_group': fg,
            'action': f'Request review from {fg} focus group'
        })
    elif status == 'rejected':
        result['blockers'].append({
            'type': 'rejection',
            'focus_group': fg,
            'reason': impact.get('approval_comment', 'No reason given'),
            'action': f'Address {fg} feedback and re-request approval'
        })
    elif status == 'blocked':
        result['blockers'].append({
            'type': 'blocked',
            'focus_group': fg,
            'blocked_by': impact.get('blocked_by'),
            'action': f'Wait for {impact.get(\"blocked_by\")} to approve first'
        })

# Generate next actions
if not fully:
    if pending > 0:
        result['next_actions'].append('Follow up with pending focus groups')
    if rejected > 0:
        result['next_actions'].append('Address rejection feedback')
    if blocked > 0:
        result['next_actions'].append('Resolve blocking dependencies')
else:
    result['next_actions'].append('Create implementation: /wgsd create-implementation')
    result['next_actions'].append('Assign owner: /wgsd assign')

print(json.dumps(result, indent=2))
"
  return 0
}
```

---

### approval_matrix_get_pending_approvers

Get list of users/FGs that still need to approve.

```bash
# Usage: approval_matrix_get_pending_approvers <concept_path>
# Returns: JSON array of pending approvers with FG and user info
approval_matrix_get_pending_approvers() {
  local concept_path="$1"
  local impact_file="$concept_path/impact-matrix.md"
  local repo_path="${2:-.}"
  
  local impacts=$(impact_parse_file "$impact_file")
  
  python3 -c "
import json

impacts = json.loads('''$impacts''')

pending = []
for impact in impacts:
    status = impact.get('status', 'pending')
    if status in ['pending', 'blocked']:
        pending.append({
            'focus_group': impact.get('focus_group'),
            'priority': impact.get('priority'),
            'status': status,
            'blocked_by': impact.get('blocked_by'),
            'sla_deadline': impact.get('sla_deadline')
        })

# Sort by priority (P0 first)
priority_order = {'P0': 0, 'P1': 1, 'P2': 2, 'P3': 3}
pending.sort(key=lambda x: priority_order.get(x['priority'], 99))

print(json.dumps(pending, indent=2))
"
  return 0
}
```

---

### approval_matrix_reminder

Generate reminder message for pending approvals.

```bash
# Usage: approval_matrix_reminder <concept_path>
# Returns: Formatted reminder message
approval_matrix_reminder() {
  local concept_path="$1"
  local concept_name=$(basename "$concept_path")
  
  local pending=$(approval_matrix_get_pending_approvers "$concept_path")
  local sla_status=$(approval_matrix_check_sla "$concept_path/impact-matrix.md")
  
  python3 -c "
import json

pending = json.loads('''$pending''')
sla_status = json.loads('''$sla_status''')
concept = '$concept_name'

if not pending:
    print('✅ No pending approvals - concept is fully approved!')
    exit(0)

print(f'⏰ **Approval Reminder: {concept}**')
print()

# Overdue
if sla_status.get('overdue'):
    print('🔴 **OVERDUE:**')
    for item in sla_status['overdue']:
        print(f'  - {item[\"focus_group\"]} (deadline: {item[\"deadline\"]})')
    print()

# Approaching deadline
if sla_status.get('approaching'):
    print('⚠️ **Approaching Deadline:**')
    for item in sla_status['approaching']:
        print(f'  - {item[\"focus_group\"]} ({item[\"remaining_hours\"]}h remaining)')
    print()

# Pending
pending_only = [p for p in pending if p['status'] == 'pending']
if pending_only:
    print('⏳ **Pending Review:**')
    for item in pending_only:
        print(f'  - {item[\"focus_group\"]} ({item[\"priority\"]})')
    print()

# Blocked
blocked = [p for p in pending if p['status'] == 'blocked']
if blocked:
    print('⏸️ **Blocked:**')
    for item in blocked:
        print(f'  - {item[\"focus_group\"]} (waiting on {item[\"blocked_by\"]})')
    print()

print('Use `/wgsd approve " + concept + " --fg <focus-group>` to approve.')
"
  return 0
}
```

---

### approval_matrix_trigger_on_complete

Called when concept becomes fully approved. Triggers Phase 13 roadmap merge.

```bash
# Usage: approval_matrix_trigger_on_complete <concept_path>
approval_matrix_trigger_on_complete() {
  local concept_path="$1"
  local concept_name=$(basename "$concept_path")
  local fg_name=$(basename $(dirname $(dirname "$concept_path")))
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  
  echo "=== APPROVAL COMPLETE: $concept_name ==="
  
  # 1. Update concept file status
  local concept_file="$concept_path/CONCEPT.md"
  if [ -f "$concept_file" ]; then
    sed -i 's/\*\*Status:\*\* .*/\*\*Status:\*\* 🚀 Ready for Implementation/' "$concept_file"
    echo "✓ Updated concept status"
  fi
  
  # 2. Create approved marker with timestamp
  cat > "$concept_path/.approved" <<EOF
approved_at: $timestamp
concept: $concept_name
focus_group: $fg_name
EOF
  echo "✓ Created approval marker"
  
  # 3. Add to implementation queue (if exists)
  local queue_file=".planning/IMPLEMENTATION-QUEUE.md"
  if [ -f "$queue_file" ]; then
    echo "- [ ] $concept_name (Approved: $(date +%Y-%m-%d))" >> "$queue_file"
    echo "✓ Added to implementation queue"
  fi
  
  # 4. Commit approval status
  git add "$concept_path" 2>/dev/null
  git commit -m "feat(${fg_name}): ${concept_name} fully approved" 2>/dev/null || true
  
  # 5. Phase 13: Trigger automatic merge to roadmap branch
  echo ""
  echo "📋 Triggering roadmap merge..."
  echo ""
  
  # Execute merge-to-roadmap workflow
  if [ -f "workflows/merge-to-roadmap.md" ]; then
    # The workflow will handle the actual merge
    echo "TRIGGER:roadmap_merge:$concept_name"
    echo ""
    echo "Run: /wgsd merge-to-roadmap $concept_name"
  else
    # Fallback: manual trigger output
    echo "⚠️ merge-to-roadmap workflow not found"
    echo "   Manual merge required:"
    echo "   git checkout roadmap"
    echo "   git merge concepts/${concept_name}"
    echo "   git push origin roadmap"
  fi
  
  # 6. Notify for canvas update
  echo ""
  echo "TRIGGER:notify_channel:$concept_name:approved"
  echo "TRIGGER:canvas_update:$fg_name"
  echo "TRIGGER:canvas_update:roadmap"
  
  return 0
}
```

---

*Library enhanced for Phase 12: Matrix-Based Approval System*
*Supports WORKFLOW-02: Focus group approval process*
