# Phase 8: Slack Channel Automation - Execution Plan

**Phase:** 8 - Slack Channel Automation
**Requirements:** 4 (SLACK-AUTO-01 through SLACK-AUTO-04)
**Estimated Duration:** ~1.5 hours
**Plan Date:** 2026-02-23

---

## Overview

Integrate automated Slack channel creation into the migration workflow so that all necessary channels exist when migration completes.

| Requirement | Description | Priority |
|-------------|-------------|----------|
| SLACK-AUTO-01 | Auto-create `{stub}-dev` | High |
| SLACK-AUTO-02 | Auto-create `{stub}-fg-{focus}` channels | High |
| SLACK-AUTO-03 | Auto-create `{stub}-community` | High |
| SLACK-AUTO-04 | Integrate into migrate.md flow | Critical |

---

## Execution Sequence

### Wave 1: Add Slack Validation Step
**File:** `workflows/migrate-planning.md`
**Duration:** ~15 min

Add preflight check for Slack connectivity before any channel operations.

#### Task 1.1: Add Slack Token Validation (After Step 2)

Insert new step after backup creation:

```bash
echo "═══════════════════════════════════════════════════"
echo "🔑 Step 2.5: Validating Slack Connectivity"
echo "═══════════════════════════════════════════════════"

SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json 2>/dev/null | jq -r '.channels.slack.botToken')

if [ "$SLACK_TOKEN" == "null" ] || [ -z "$SLACK_TOKEN" ]; then
  echo "⚠️  No Slack bot token found"
  echo ""
  echo "Slack channel automation will be skipped."
  echo "Create channels manually after migration:"
  echo "  /wgsd setup-core-channels <stub>"
  echo "  /wgsd create-focus-group <name>"
  echo ""
  SLACK_ENABLED="false"
else
  echo "✅ Slack token found"
  
  # Verify token works
  SLACK_TEST=$(curl -s -X GET "https://slack.com/api/auth.test" \
    -H "Authorization: Bearer $SLACK_TOKEN")
  
  if [ "$(echo "$SLACK_TEST" | jq -r '.ok')" = "true" ]; then
    BOT_NAME=$(echo "$SLACK_TEST" | jq -r '.user')
    echo "✅ Connected as: $BOT_NAME"
    SLACK_ENABLED="true"
  else
    echo "⚠️  Token invalid or expired"
    SLACK_ENABLED="false"
  fi
fi

echo ""
```

---

### Wave 2: Add Core Channel Creation Step
**File:** `workflows/migrate-planning.md`
**Duration:** ~25 min

After Step 8 (Create WGSD-CONFIG.md), add core channel creation.

#### Task 2.1: Add Core Channels Step (After Step 8)

```bash
if [ "$SLACK_ENABLED" = "true" ]; then
  echo "═══════════════════════════════════════════════════"
  echo "📦 Step 9: Creating Core Slack Channels"
  echo "═══════════════════════════════════════════════════"
  echo ""
  
  STUB="$SUGGESTED_STUB"
  DEV_CHANNEL="${STUB}-dev"
  COMMUNITY_CHANNEL="${STUB}-community"
  
  CREATED_CHANNELS=""
  EXISTING_CHANNELS=""
  FAILED_CHANNELS=""
  
  # --- Create Dev Channel (Private) ---
  echo "🔧 Creating: #$DEV_CHANNEL (private)"
  
  DEV_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"$DEV_CHANNEL\",\"is_private\":true}")
  
  DEV_OK=$(echo "$DEV_RESPONSE" | jq -r '.ok')
  DEV_ERROR=$(echo "$DEV_RESPONSE" | jq -r '.error // empty')
  
  if [ "$DEV_OK" = "true" ]; then
    DEV_ID=$(echo "$DEV_RESPONSE" | jq -r '.channel.id')
    echo "   ✅ Created: #$DEV_CHANNEL ($DEV_ID)"
    CREATED_CHANNELS="$CREATED_CHANNELS $DEV_CHANNEL"
    
    # Set topic and purpose
    curl -s -X POST https://slack.com/api/conversations.setTopic \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"$DEV_ID\",\"topic\":\"🛠️ Core development | WGSD managed\"}" >/dev/null
      
    curl -s -X POST https://slack.com/api/conversations.setPurpose \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"$DEV_ID\",\"purpose\":\"Main development coordination for $STUB. Architecture, planning, cross-cutting concerns.\"}" >/dev/null
      
  elif [ "$DEV_ERROR" = "name_taken" ]; then
    echo "   ℹ️  Already exists: #$DEV_CHANNEL"
    EXISTING_CHANNELS="$EXISTING_CHANNELS $DEV_CHANNEL"
    # Find existing ID
    DEV_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=private_channel&limit=500" \
      -H "Authorization: Bearer $SLACK_TOKEN" | \
      jq -r --arg name "$DEV_CHANNEL" '.channels[] | select(.name == $name) | .id')
  else
    echo "   ❌ Failed: $DEV_ERROR"
    FAILED_CHANNELS="$FAILED_CHANNELS $DEV_CHANNEL"
  fi
  
  sleep 1  # Rate limit courtesy
  
  # --- Create Community Channel (Public) ---
  echo "🌍 Creating: #$COMMUNITY_CHANNEL (public)"
  
  COMMUNITY_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
    -H "Authorization: Bearer $SLACK_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"$COMMUNITY_CHANNEL\",\"is_private\":false}")
  
  COMMUNITY_OK=$(echo "$COMMUNITY_RESPONSE" | jq -r '.ok')
  COMMUNITY_ERROR=$(echo "$COMMUNITY_RESPONSE" | jq -r '.error // empty')
  
  if [ "$COMMUNITY_OK" = "true" ]; then
    COMMUNITY_ID=$(echo "$COMMUNITY_RESPONSE" | jq -r '.channel.id')
    echo "   ✅ Created: #$COMMUNITY_CHANNEL ($COMMUNITY_ID)"
    CREATED_CHANNELS="$CREATED_CHANNELS $COMMUNITY_CHANNEL"
    
    # Set topic and purpose
    curl -s -X POST https://slack.com/api/conversations.setTopic \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"$COMMUNITY_ID\",\"topic\":\"💬 Community feedback and discussion | Public\"}" >/dev/null
      
    curl -s -X POST https://slack.com/api/conversations.setPurpose \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"$COMMUNITY_ID\",\"purpose\":\"Public channel for community feedback, feature requests, and discussion.\"}" >/dev/null
      
  elif [ "$COMMUNITY_ERROR" = "name_taken" ]; then
    echo "   ℹ️  Already exists: #$COMMUNITY_CHANNEL"
    EXISTING_CHANNELS="$EXISTING_CHANNELS $COMMUNITY_CHANNEL"
    COMMUNITY_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel&limit=500" \
      -H "Authorization: Bearer $SLACK_TOKEN" | \
      jq -r --arg name "$COMMUNITY_CHANNEL" '.channels[] | select(.name == $name) | .id')
  else
    echo "   ❌ Failed: $COMMUNITY_ERROR"
    FAILED_CHANNELS="$FAILED_CHANNELS $COMMUNITY_CHANNEL"
  fi
  
  echo ""
fi
```

---

### Wave 3: Add Focus Group Channel Creation
**File:** `workflows/migrate-planning.md`
**Duration:** ~30 min

Create focus group channels based on migration analysis suggestions.

#### Task 3.1: Capture Focus Groups from Analysis

Earlier in migration (Step 7), after domain analysis, capture suggested focus groups:

```bash
# In Step 7 (Analyze REQUIREMENTS.md)
# After domain detection, save suggestions
SUGGESTED_FOCUS_GROUPS=""

if grep -qi "security\|auth\|permission" .planning/REQUIREMENTS.md 2>/dev/null; then
  SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS security"
fi
if grep -qi "onboard\|setup\|wizard" .planning/REQUIREMENTS.md 2>/dev/null; then
  SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS onboarding"
fi
# ... (existing domain detection)

# Trim and save
SUGGESTED_FOCUS_GROUPS=$(echo "$SUGGESTED_FOCUS_GROUPS" | xargs)
```

#### Task 3.2: Add Focus Group Channels Step (After Core Channels)

```bash
if [ "$SLACK_ENABLED" = "true" ] && [ -n "$SUGGESTED_FOCUS_GROUPS" ]; then
  echo "═══════════════════════════════════════════════════"
  echo "🎯 Step 10: Creating Focus Group Channels"
  echo "═══════════════════════════════════════════════════"
  echo ""
  
  for FG_NAME in $SUGGESTED_FOCUS_GROUPS; do
    FG_CHANNEL="${STUB}-fg-${FG_NAME}"
    
    echo "🎯 Creating: #$FG_CHANNEL (private)"
    
    FG_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"name\":\"$FG_CHANNEL\",\"is_private\":true}")
    
    FG_OK=$(echo "$FG_RESPONSE" | jq -r '.ok')
    FG_ERROR=$(echo "$FG_RESPONSE" | jq -r '.error // empty')
    
    if [ "$FG_OK" = "true" ]; then
      FG_ID=$(echo "$FG_RESPONSE" | jq -r '.channel.id')
      echo "   ✅ Created: #$FG_CHANNEL ($FG_ID)"
      CREATED_CHANNELS="$CREATED_CHANNELS $FG_CHANNEL"
      
      # Set topic and purpose
      FG_TITLE=$(echo "$FG_NAME" | sed 's/-/ /g' | sed 's/\b\(.\)/\u\1/g')
      curl -s -X POST https://slack.com/api/conversations.setTopic \
        -H "Authorization: Bearer $SLACK_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"channel\":\"$FG_ID\",\"topic\":\"🎯 Focus Group: $FG_TITLE | Planning & ideation\"}" >/dev/null
        
      curl -s -X POST https://slack.com/api/conversations.setPurpose \
        -H "Authorization: Bearer $SLACK_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"channel\":\"$FG_ID\",\"purpose\":\"Long-lived focus group for $FG_NAME development. Concepts developed here before implementation.\"}" >/dev/null
        
      # Record in WGSD-CONFIG.md
      echo "| $FG_CHANNEL | $FG_ID | focus-group | active | $(date +%Y-%m-%d) |" >> .planning/WGSD-CONFIG.md
      
    elif [ "$FG_ERROR" = "name_taken" ]; then
      echo "   ℹ️  Already exists: #$FG_CHANNEL"
      EXISTING_CHANNELS="$EXISTING_CHANNELS $FG_CHANNEL"
    else
      echo "   ❌ Failed: $FG_ERROR"
      FAILED_CHANNELS="$FAILED_CHANNELS $FG_CHANNEL"
    fi
    
    sleep 1  # Rate limit courtesy
  done
  
  echo ""
fi
```

---

### Wave 4: Update Config and Summary
**File:** `workflows/migrate-planning.md`
**Duration:** ~20 min

#### Task 4.1: Add Channel Registry to WGSD-CONFIG.md (Step 8)

Update the WGSD-CONFIG.md template generation to include a channel registry section:

```bash
# Add to Step 8 (Create WGSD-CONFIG.md) - inside the heredoc
cat >> .planning/WGSD-CONFIG.md << 'CHANNELS_SECTION'

---

## Channel Registry

| Channel | ID | Type | Status | Created |
|---------|----|----|--------|---------|
|---------|----|----|--------|---------|
CHANNELS_SECTION
```

#### Task 4.2: Add Channel IDs After Creation

After creating each channel, add to registry:
```bash
echo "| $CHANNEL_NAME | $CHANNEL_ID | $TYPE | active | $(date +%Y-%m-%d) |" >> .planning/WGSD-CONFIG.md
```

#### Task 4.3: Generate OpenClaw Patch

```bash
if [ "$SLACK_ENABLED" = "true" ]; then
  echo "═══════════════════════════════════════════════════"
  echo "⚙️  Step 11: OpenClaw Registration"
  echo "═══════════════════════════════════════════════════"
  
  # Build patch JSON
  CHANNEL_PATCHES=""
  
  [ -n "$DEV_ID" ] && CHANNEL_PATCHES="${CHANNEL_PATCHES}\"$DEV_ID\":{\"allow\":true,\"requireMention\":false},"
  [ -n "$COMMUNITY_ID" ] && CHANNEL_PATCHES="${CHANNEL_PATCHES}\"$COMMUNITY_ID\":{\"allow\":true,\"requireMention\":false},"
  
  for FG_NAME in $SUGGESTED_FOCUS_GROUPS; do
    FG_ID_VAR="FG_${FG_NAME}_ID"
    FG_ID="${!FG_ID_VAR}"
    [ -n "$FG_ID" ] && CHANNEL_PATCHES="${CHANNEL_PATCHES}\"$FG_ID\":{\"allow\":true,\"requireMention\":false},"
  done
  
  # Remove trailing comma and apply
  CHANNEL_PATCHES=$(echo "$CHANNEL_PATCHES" | sed 's/,$//')
  
  if [ -n "$CHANNEL_PATCHES" ]; then
    PATCH_JSON="{\"channels\":{\"slack\":{\"channels\":{$CHANNEL_PATCHES}}}}"
    
    if command -v openclaw &> /dev/null; then
      openclaw gateway config.patch "$PATCH_JSON" 2>/dev/null && \
        echo "✅ Channels registered with OpenClaw" || \
        echo "⚠️  Auto-registration failed (apply manually)"
    else
      echo "Run this to enable monitoring:"
      echo "openclaw gateway config.patch '$PATCH_JSON'"
    fi
  fi
  
  echo ""
fi
```

#### Task 4.4: Update Success Report (Step 12)

Replace the existing "Next Steps" section:

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🎉 GSD → WGSD Migration Complete"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Structure:"
echo "   .planning/"
echo "   ├── focus-groups/     ✅"
echo "   ├── active-implementations/  ✅"
echo "   ├── WGSD-CONFIG.md    ✅"
echo "   └── MASTER-ROADMAP.md ✅"
echo ""

if [ "$SLACK_ENABLED" = "true" ]; then
  echo "📱 Slack Channels:"
  [ -n "$CREATED_CHANNELS" ] && echo "   ✅ Created:$CREATED_CHANNELS"
  [ -n "$EXISTING_CHANNELS" ] && echo "   ℹ️  Existing:$EXISTING_CHANNELS"
  [ -n "$FAILED_CHANNELS" ] && echo "   ❌ Failed:$FAILED_CHANNELS"
  echo ""
  echo "   🔗 Channels are ready to use!"
  echo ""
fi

if [ -n "${BACKUP_DIR:-}" ]; then
  echo "📦 Backup: $BACKUP_DIR"
  echo ""
fi

echo "✅ Migration complete — all channels created automatically!"
echo ""
echo "Next Steps:"
echo "  1. Review WGSD-CONFIG.md"
echo "  2. Join your channels in Slack"
echo "  3. Start creating concepts: /wgsd create-concept <name>"
echo ""
```

---

## File Changes Summary

| File | Type | Changes | Status |
|------|------|---------|--------|
| `workflows/migrate-planning.md` | Modify | Add Steps 2.5, 9, 10, 11; Update Step 8, 12 | ⏳ |

**Single file modification** — all changes are integration work in the migration workflow.

---

## Testing Strategy

### Test Case 1: Fresh Migration with Slack
1. Create test GSD project
2. Run `/wgsd migrate-planning`
3. Verify: All channels created
4. Verify: WGSD-CONFIG.md has channel registry
5. Verify: OpenClaw patch generated

### Test Case 2: Migration Without Slack Token
1. Remove/invalidate Slack token
2. Run migration
3. Verify: Migration completes successfully
4. Verify: Clear message about manual channel creation

### Test Case 3: Partial Existing Channels
1. Pre-create some channels manually
2. Run migration
3. Verify: Existing channels detected and skipped
4. Verify: New channels created
5. Verify: Summary shows both created and existing

### Test Case 4: API Rate Handling
1. Run migration with many focus groups (5+)
2. Verify: 1-second delays between calls
3. Verify: No rate limit errors

---

## Risk Mitigation

| Risk | Mitigation | Severity |
|------|------------|----------|
| Slack token missing | Graceful degradation, clear instructions | Low |
| Channel creation fails | Log and continue, report at end | Medium |
| Rate limiting | 1-second delay between calls | Low |
| Existing channel conflict | Detect and skip gracefully | Low |

---

## Dependencies

- **Phase 7 Complete:** ✅ Migration logic now extracts concepts and suggests focus groups
- **Existing Workflows:** `lib/slack-api.md` provides all API functions
- **No External Dependencies:** All Slack calls use existing patterns

---

## Success Criteria

- [ ] `{stub}-dev` created automatically during migration (SLACK-AUTO-01)
- [ ] `{stub}-community` created automatically (SLACK-AUTO-03)
- [ ] `{stub}-fg-{name}` created for each suggested focus group (SLACK-AUTO-02)
- [ ] All channel IDs recorded in WGSD-CONFIG.md
- [ ] OpenClaw registration patch generated
- [ ] Migration works with OR without Slack token
- [ ] Clear summary shows created/existing/failed channels

---

## Execution Command

```
/gsd build-phase 8
```

Or manual execution:
1. Add Slack validation step (Task 1.1)
2. Add core channels step (Task 2.1)
3. Add focus group channels step (Task 3.1, 3.2)
4. Update config and summary (Tasks 4.1-4.4)
5. Run test migration

---

*Execution plan complete — Phase 8 ready for implementation*
