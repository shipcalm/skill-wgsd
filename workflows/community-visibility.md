---
name: wgsd:community-visibility
description: Manage transparent development visibility for community members
argument-hint: "[update-canvas|attribute|status|publish] [args]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Provide transparent development visibility to the community:
- Generate sanitized roadmap for community canvas
- Display community contributor attributions
- Publish progress updates without internal details
- Track community engagement and contributions
</objective>

<libraries>
- workflows/lib/canvas-api.md - Canvas operations
- workflows/lib/canvas-templates.md - Template generation
</libraries>

<process>

## Action: Update Community Canvas

Generate and update the community-facing roadmap canvas:

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "🎨 Generating community canvas content..."

# Get project info
PROJECT_FILE=".planning/PROJECT.md"
PROJECT_NAME=$(grep "^# " "$PROJECT_FILE" 2>/dev/null | head -1 | sed 's/# //' || echo "Project")
STUB=$(grep "^Stub:" .planning/WGSD-CONFIG.md 2>/dev/null | sed 's/Stub: //' || echo "proj")

# Generate focus groups section
echo ""
echo "📊 Scanning focus groups..."

FOCUS_GROUPS_MD=""
for fg_dir in .planning/focus-groups/*/; do
    [ -d "$fg_dir" ] || continue
    
    FG_NAME=$(basename "$fg_dir")
    FG_TITLE=$(echo "$FG_NAME" | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
    
    # Count concepts
    CONCEPT_COUNT=$(find "${fg_dir}concepts/" -name "*.md" 2>/dev/null | wc -l)
    
    # Get status indicator
    ACTIVE_IMPL=$(find .planning/active-implementations -name "*.md" -exec grep -l "Focus Group: ${FG_NAME}" {} \; 2>/dev/null | wc -l)
    if [ "$ACTIVE_IMPL" -gt 0 ]; then
        STATUS="🟢 Active Development"
    elif [ "$CONCEPT_COUNT" -gt 0 ]; then
        STATUS="🟡 Planning"
    else
        STATUS="⚪ Idle"
    fi
    
    FOCUS_GROUPS_MD="${FOCUS_GROUPS_MD}| ${FG_TITLE} | ${CONCEPT_COUNT} | ${STATUS} |\n"
done

# Generate in-development section
echo "📊 Scanning implementations..."

IN_DEV_MD=""
for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    
    IMPL_NAME=$(basename "$impl_dir")
    STATE_FILE="${impl_dir}STATE.md"
    
    FOCUS_GROUP=$(grep "^Focus Group:" "$STATE_FILE" 2>/dev/null | sed 's/Focus Group: //' || echo "unknown")
    PROGRESS=$(grep "^Progress:" "$STATE_FILE" 2>/dev/null | sed 's/Progress: //' || echo "0%")
    
    # Check for community origin
    SOURCE=$(grep "^Source Concept:" "$STATE_FILE" 2>/dev/null | sed 's/Source Concept: //')
    COMMUNITY_ORIGIN=""
    if [ -n "$SOURCE" ]; then
        CONCEPT_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${SOURCE}.md"
        if [ -f "$CONCEPT_FILE" ] && grep -q "Community Feedback" "$CONCEPT_FILE"; then
            AUTHOR=$(grep "^**Original Contributor:**" "$CONCEPT_FILE" | sed 's/.*: //')
            COMMUNITY_ORIGIN="✅ ${AUTHOR:-Community}"
        fi
    fi
    
    # Sanitize name for public display
    PUBLIC_NAME=$(echo "$IMPL_NAME" | sed 's/-impl$//' | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
    
    IN_DEV_MD="${IN_DEV_MD}| ${PUBLIC_NAME} | ${FOCUS_GROUP} | ${PROGRESS} | ${COMMUNITY_ORIGIN:-—} |\n"
done

# Generate recently shipped section
echo "📊 Scanning completed implementations..."

SHIPPED_MD=""
for impl_dir in .planning/completed-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    
    IMPL_NAME=$(basename "$impl_dir")
    STATE_FILE="${impl_dir}STATE.md"
    
    COMPLETED_DATE=$(grep "^Completed:" "$STATE_FILE" 2>/dev/null | sed 's/Completed: //' | head -c 10)
    
    # Only show last 5
    PUBLIC_NAME=$(echo "$IMPL_NAME" | sed 's/-impl$//' | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
    SHIPPED_MD="${SHIPPED_MD}- **${PUBLIC_NAME}** - Shipped ${COMPLETED_DATE:-recently}\n"
done | head -5

# Generate community contributors section
echo "📊 Scanning community contributors..."

CONTRIBUTORS_MD=""
for fg_dir in .planning/focus-groups/*/; do
    [ -d "$fg_dir" ] || continue
    
    CONTRIB_FILE="${fg_dir}CONTRIBUTORS.md"
    if [ -f "$CONTRIB_FILE" ]; then
        grep "^| @" "$CONTRIB_FILE" 2>/dev/null | while read -r line; do
            USER=$(echo "$line" | awk -F'|' '{print $2}' | xargs)
            CONTRIB=$(echo "$line" | awk -F'|' '{print $4}' | xargs)
            CONTRIBUTORS_MD="${CONTRIBUTORS_MD}- ${USER}: ${CONTRIB}\n"
        done
    fi
done

# Generate canvas content
CANVAS_CONTENT=$(cat << EOF
# ${PROJECT_NAME} Development Roadmap

**Last Updated:** ${TIMESTAMP}

---

## What We're Building

Our team is actively developing new features across several focus areas:

| Focus Area | Ideas | Status |
|------------|-------|--------|
$(echo -e "$FOCUS_GROUPS_MD")

---

## In Development

These features are currently being implemented:

| Feature | Area | Progress | Community Origin |
|---------|------|----------|------------------|
$(echo -e "$IN_DEV_MD")

---

## Recently Shipped 🚀

$(echo -e "$SHIPPED_MD")

---

## Community Contributors 🙏

Thank you to our community members who have contributed ideas and feedback!

$(echo -e "$CONTRIBUTORS_MD")

---

## How to Contribute

We love hearing from you! Here's how to get involved:

1. **Share Ideas** - Post your suggestions in #${STUB}-community
2. **Report Issues** - Let us know if something isn't working
3. **Join Development** - Request access to focus groups for deeper involvement

Your feedback directly shapes what we build!

---

*This roadmap is automatically generated from our development progress.*
*Some internal details are not shown for security reasons.*
EOF
)

echo ""
echo "✅ Community canvas content generated"
echo ""
echo "───────────────────────────────────────"
echo "$CANVAS_CONTENT"
echo "───────────────────────────────────────"
echo ""
echo "💡 Update the community channel canvas with this content"
```

**Post to community channel canvas using canvas-sync workflow.**

---

## Action: Add Attribution

Add or update contributor attribution:

```bash
USERNAME="{username}"
CONCEPT="{concept-name}"
CONTRIBUTION="{contribution-description}"
FOCUS_GROUP="{focus-group}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📝 Adding attribution..."

USERNAME=$(echo "$USERNAME" | sed 's/@//')

# Verify focus group
FG_DIR=".planning/focus-groups/${FOCUS_GROUP}"
if [ ! -d "$FG_DIR" ]; then
    echo "❌ Focus group not found: ${FOCUS_GROUP}"
    exit 1
fi

# Add to contributors file
CONTRIB_FILE="${FG_DIR}/CONTRIBUTORS.md"
if [ ! -f "$CONTRIB_FILE" ]; then
    cat > "$CONTRIB_FILE" << EOF
# ${FOCUS_GROUP} Focus Group Contributors

## Core Team
_(Focus group owners and maintainers)_

## Community Contributors
| User | Joined | Contribution |
|------|--------|--------------|
EOF
fi

# Check if already attributed for this concept
if grep -q "@${USERNAME}.*${CONCEPT}" "$CONTRIB_FILE" 2>/dev/null; then
    echo "ℹ️ Attribution already exists for @${USERNAME} on ${CONCEPT}"
    exit 0
fi

# Add attribution
echo "| @${USERNAME} | ${TIMESTAMP:0:10} | ${CONTRIBUTION} (${CONCEPT}) |" >> "$CONTRIB_FILE"

echo ""
echo "✅ Attribution added!"
echo ""
echo "📋 Details:"
echo "   User: @${USERNAME}"
echo "   Concept: ${CONCEPT}"
echo "   Contribution: ${CONTRIBUTION}"
echo "   Focus Group: ${FOCUS_GROUP}"
echo ""

# Update concept file if exists
CONCEPT_FILE="${FG_DIR}/concepts/${CONCEPT}.md"
if [ -f "$CONCEPT_FILE" ]; then
    if ! grep -q "Community Attribution" "$CONCEPT_FILE"; then
        cat >> "$CONCEPT_FILE" << EOF

---

## Community Attribution

| Contributor | Contribution |
|-------------|--------------|
| @${USERNAME} | ${CONTRIBUTION} |
EOF
    else
        # Add to existing attribution section
        sed -i "/## Community Attribution/a | @${USERNAME} | ${CONTRIBUTION} |" "$CONCEPT_FILE"
    fi
    echo "📝 Updated concept file with attribution"
fi

echo ""
echo "💡 Run canvas sync to update community visibility:"
echo "   /wgsd update-community-canvas"
```

---

## Action: Community Status

Show community engagement statistics:

```bash
echo "📊 Community Engagement Status"
echo ""
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Feedback stats
FEEDBACK_DIR=".planning/community/feedback"
FEEDBACK_TOTAL=0
FEEDBACK_TRIAGED=0
FEEDBACK_PENDING=0

if [ -d "$FEEDBACK_DIR" ]; then
    for f in "$FEEDBACK_DIR"/*.md; do
        [ -f "$f" ] || continue
        FEEDBACK_TOTAL=$((FEEDBACK_TOTAL + 1))
        STATUS=$(grep "^**Status:**" "$f" | sed 's/.*: //')
        case "$STATUS" in
            *Triaged*|*Acknowledged*) FEEDBACK_TRIAGED=$((FEEDBACK_TRIAGED + 1)) ;;
            *Pending*) FEEDBACK_PENDING=$((FEEDBACK_PENDING + 1)) ;;
        esac
    done
fi

# Invitation stats
INVITE_DIR=".planning/community/invitations"
INVITES_SENT=0
INVITES_ACCEPTED=0
INVITES_PENDING=0

if [ -d "$INVITE_DIR" ]; then
    for f in "$INVITE_DIR"/*.md; do
        [ -f "$f" ] || continue
        INVITES_SENT=$((INVITES_SENT + 1))
        STATUS=$(grep "^**Status:**" "$f" | sed 's/.*: //')
        case "$STATUS" in
            *Accepted*) INVITES_ACCEPTED=$((INVITES_ACCEPTED + 1)) ;;
            *Pending*) INVITES_PENDING=$((INVITES_PENDING + 1)) ;;
        esac
    done
fi

# Access request stats
REQUEST_DIR=".planning/community/access-requests"
REQUESTS_TOTAL=0
REQUESTS_APPROVED=0

if [ -d "$REQUEST_DIR" ]; then
    for f in "$REQUEST_DIR"/*.md; do
        [ -f "$f" ] || continue
        REQUESTS_TOTAL=$((REQUESTS_TOTAL + 1))
        STATUS=$(grep "^**Status:**" "$f" | sed 's/.*: //')
        [[ "$STATUS" == *Approved* ]] && REQUESTS_APPROVED=$((REQUESTS_APPROVED + 1))
    done
fi

# Community contributors
CONTRIBUTOR_COUNT=0
for fg_dir in .planning/focus-groups/*/; do
    [ -d "$fg_dir" ] || continue
    CONTRIB_FILE="${fg_dir}CONTRIBUTORS.md"
    if [ -f "$CONTRIB_FILE" ]; then
        COUNT=$(grep -c "^| @" "$CONTRIB_FILE" 2>/dev/null || echo 0)
        CONTRIBUTOR_COUNT=$((CONTRIBUTOR_COUNT + COUNT))
    fi
done

# Community-sourced concepts
COMMUNITY_CONCEPTS=0
for concept_file in .planning/focus-groups/*/concepts/*.md; do
    [ -f "$concept_file" ] || continue
    if grep -q "Community Feedback" "$concept_file"; then
        COMMUNITY_CONCEPTS=$((COMMUNITY_CONCEPTS + 1))
    fi
done

echo "## Community Engagement Summary"
echo ""
echo "**Generated:** ${TIMESTAMP}"
echo ""
echo "### Feedback Pipeline"
echo ""
echo "| Metric | Count |"
echo "|--------|-------|"
echo "| Total Feedback | ${FEEDBACK_TOTAL} |"
echo "| Triaged | ${FEEDBACK_TRIAGED} |"
echo "| Pending | ${FEEDBACK_PENDING} |"
echo ""

echo "### Contributor Growth"
echo ""
echo "| Metric | Count |"
echo "|--------|-------|"
echo "| Invitations Sent | ${INVITES_SENT} |"
echo "| Invitations Accepted | ${INVITES_ACCEPTED} |"
echo "| Pending Invitations | ${INVITES_PENDING} |"
echo "| Access Requests | ${REQUESTS_TOTAL} |"
echo "| Approved | ${REQUESTS_APPROVED} |"
echo ""

echo "### Impact"
echo ""
echo "| Metric | Count |"
echo "|--------|-------|"
echo "| Community Contributors | ${CONTRIBUTOR_COUNT} |"
echo "| Community-Sourced Concepts | ${COMMUNITY_CONCEPTS} |"
echo ""

if [ "$FEEDBACK_TOTAL" -gt 0 ]; then
    TRIAGE_RATE=$((FEEDBACK_TRIAGED * 100 / FEEDBACK_TOTAL))
    echo "**Triage Rate:** ${TRIAGE_RATE}%"
fi

if [ "$INVITES_SENT" -gt 0 ]; then
    ACCEPT_RATE=$((INVITES_ACCEPTED * 100 / INVITES_SENT))
    echo "**Invitation Accept Rate:** ${ACCEPT_RATE}%"
fi
```

---

## Action: Publish Progress Update

Publish a progress update to the community channel:

```bash
UPDATE_TYPE="{type:-general}"
MESSAGE="{message}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📢 Preparing progress update..."

# Get project info
PROJECT_FILE=".planning/PROJECT.md"
PROJECT_NAME=$(grep "^# " "$PROJECT_FILE" 2>/dev/null | head -1 | sed 's/# //' || echo "Project")

case "$UPDATE_TYPE" in
    shipped|release)
        EMOJI="🚀"
        TITLE="New Release"
        ;;
    milestone)
        EMOJI="🎯"
        TITLE="Milestone Reached"
        ;;
    progress)
        EMOJI="📈"
        TITLE="Progress Update"
        ;;
    community)
        EMOJI="🙏"
        TITLE="Community Highlight"
        ;;
    *)
        EMOJI="📣"
        TITLE="Update"
        ;;
esac

# Get active implementations for context
ACTIVE_IMPLS=""
for impl_dir in .planning/active-implementations/*/; do
    [ -d "$impl_dir" ] || continue
    IMPL_NAME=$(basename "$impl_dir")
    PROGRESS=$(grep "^Progress:" "${impl_dir}STATE.md" 2>/dev/null | sed 's/Progress: //')
    PUBLIC_NAME=$(echo "$IMPL_NAME" | sed 's/-impl$//' | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
    ACTIVE_IMPLS="${ACTIVE_IMPLS}• ${PUBLIC_NAME}: ${PROGRESS:-0%}\n"
done

echo ""
echo "───────────────────────────────────────"
cat << EOF
${EMOJI} **${TITLE}**

${MESSAGE}

---

**Current Development Progress:**
$(echo -e "$ACTIVE_IMPLS")

---

_Have feedback? Share it in this channel!_
EOF
echo "───────────────────────────────────────"
echo ""
echo "💡 Post this in the community channel"

# Log the update
UPDATES_DIR=".planning/community/updates"
mkdir -p "$UPDATES_DIR"
UPDATE_FILE="${UPDATES_DIR}/$(date +%Y%m%d-%H%M%S).md"

cat > "$UPDATE_FILE" << EOF
# Progress Update

**Type:** ${UPDATE_TYPE}
**Posted:** ${TIMESTAMP}

---

${MESSAGE}
EOF

echo ""
echo "📝 Update logged: ${UPDATE_FILE}"
```

---

## Action: Sanitization Check

Verify content doesn't contain sensitive information:

```bash
CONTENT="{content-to-check}"

echo "🔍 Checking content for sensitive information..."
echo ""

ISSUES_FOUND=0

# Check for internal references
if echo "$CONTENT" | grep -qiE "internal|confidential|private|secret"; then
    echo "⚠️ Possible internal reference found"
    ISSUES_FOUND=$((ISSUES_FOUND + 1))
fi

# Check for specific user mentions in internal context
if echo "$CONTENT" | grep -qE "discussed with @|@\w+ said|per @"; then
    echo "⚠️ Possible internal discussion reference"
    ISSUES_FOUND=$((ISSUES_FOUND + 1))
fi

# Check for technical implementation details
if echo "$CONTENT" | grep -qiE "database|schema|api key|token|password|credential"; then
    echo "⚠️ Possible technical/security detail"
    ISSUES_FOUND=$((ISSUES_FOUND + 1))
fi

# Check for pricing/business info
if echo "$CONTENT" | grep -qiE "revenue|profit|cost|pricing strategy|competitor"; then
    echo "⚠️ Possible business-sensitive information"
    ISSUES_FOUND=$((ISSUES_FOUND + 1))
fi

if [ "$ISSUES_FOUND" -eq 0 ]; then
    echo "✅ No obvious sensitive information detected"
else
    echo ""
    echo "⚠️ Found ${ISSUES_FOUND} potential issue(s)"
    echo ""
    echo "💡 Review content before publishing to community"
fi
```

</process>

<routing>
| Action | Description |
|--------|-------------|
| update-community-canvas | Generate community roadmap |
| attribute @user [concept] [desc] | Add attribution |
| community-status, status | Show engagement stats |
| publish [type] [message] | Publish progress update |
| sanitize [content] | Check for sensitive info |
</routing>

<sanitization_rules>
**Show to community:**
- Feature names and general descriptions
- Progress percentages
- Community contributor names and attributions
- High-level status indicators
- Shipped features

**Hide from community:**
- Internal discussion details
- Technical implementation specifics
- Security-sensitive information
- Unreleased feature specifics
- Team member comments/conversations
- Business-sensitive data
</sanitization_rules>

<success_criteria>
- Community canvas shows sanitized roadmap
- Attributions prominently displayed
- Progress updates published without leaks
- Engagement statistics tracked
- Sensitive information properly filtered
</success_criteria>
