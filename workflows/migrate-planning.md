---
name: wgsd:migrate-planning
description: Migrate GSD planning structure to WGSD format
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUser
---

# GSD to WGSD Planning Migration

Transform existing GSD `.planning/` structure to WGSD format while preserving all content.

---

## Objective

Migrate a GSD-structured project to WGSD by:
- Detecting existing GSD planning files
- Creating backup before modifications
- Transforming content to WGSD structure
- Generating WGSD configuration files
- **Automatically creating Slack channels**
- Preserving all original content

---

## GSD to WGSD Structure Mapping

| GSD Source | WGSD Target | Transformation |
|------------|-------------|----------------|
| PROJECT.md | PROJECT.md | Add WGSD workflow section |
| ROADMAP.md | MASTER-ROADMAP.md | Rename, add focus group refs |
| REQUIREMENTS.md | focus-groups/*/concepts/ | Split by domain |
| STATE.md | STATE.md | Add focus group status section |
| phases/* | focus-groups/* | Rename, restructure |
| codebase/* | codebase/* | Preserve unchanged |

---

## Process

### Step 1: Detect GSD Structure

```bash
echo "🔍 Detecting GSD planning structure..."

if [ ! -d ".planning" ]; then
  echo "⚠️  No .planning/ directory found"
  echo "   Creating fresh WGSD structure..."
  GSD_MODE="fresh"
else
  GSD_MODE="migrate"
  
  # Check for GSD files
  GSD_FILES=""
  [ -f ".planning/PROJECT.md" ] && GSD_FILES="$GSD_FILES PROJECT.md"
  [ -f ".planning/ROADMAP.md" ] && GSD_FILES="$GSD_FILES ROADMAP.md"
  [ -f ".planning/REQUIREMENTS.md" ] && GSD_FILES="$GSD_FILES REQUIREMENTS.md"
  [ -f ".planning/STATE.md" ] && GSD_FILES="$GSD_FILES STATE.md"
  [ -d ".planning/phases" ] && GSD_FILES="$GSD_FILES phases/"
  [ -d ".planning/codebase" ] && GSD_FILES="$GSD_FILES codebase/"
  
  if [ -z "$GSD_FILES" ]; then
    echo "⚠️  .planning/ exists but contains no GSD files"
    GSD_MODE="partial"
  else
    echo "✅ Found GSD files:$GSD_FILES"
  fi
fi
```

### Step 2: Create Backup

```bash
if [ "$GSD_MODE" != "fresh" ] && [ -d ".planning" ]; then
  BACKUP_DIR=".planning-backup-$(date +%Y%m%d-%H%M%S)"
  echo "📦 Creating backup: $BACKUP_DIR"
  cp -r .planning "$BACKUP_DIR"
  echo "✅ Backup created"
fi
```

### Step 3: Validate Slack Connectivity

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "🔑 Step 3: Validating Slack Connectivity"
echo "═══════════════════════════════════════════════════"

# Initialize channel tracking variables
SLACK_ENABLED="false"
CREATED_CHANNELS=""
EXISTING_CHANNELS=""
FAILED_CHANNELS=""
DEV_ID=""
COMMUNITY_ID=""

# Get Slack token from OpenClaw config
SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json 2>/dev/null | jq -r '.channels.slack.botToken // empty')

if [ -z "$SLACK_TOKEN" ] || [ "$SLACK_TOKEN" == "null" ]; then
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
    echo "   Error: $(echo "$SLACK_TEST" | jq -r '.error // "unknown"')"
    SLACK_ENABLED="false"
  fi
fi

echo ""
```

### Step 4: Create WGSD Directory Structure

```bash
echo "📁 Creating WGSD directory structure..."

mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations

echo "✅ Directory structure created"
```

### Step 5: Transform PROJECT.md

```bash
if [ -f ".planning/PROJECT.md" ]; then
  echo "📝 Enhancing PROJECT.md with WGSD section..."
  
  # Check if WGSD section already exists
  if ! grep -q "## WGSD Workflow" .planning/PROJECT.md; then
    cat >> .planning/PROJECT.md << 'WGSD_SECTION'

---

## WGSD Workflow

This project uses **WGSD (We Get Shit Done)** collaborative development:

1. **Focus Groups** - Long-lived topic discussions (security, onboarding, billing)
2. **Concepts** - Feature ideas developed socially within focus groups
3. **Implementations** - Short-lived code execution (1-3 days)

### Active Focus Groups

*List of focus groups will be populated as they are created.*

### Current Implementations

*List of active implementations (max 2-4 concurrent).*

### Slack Integration

Channels follow the pattern: `#{stub}-{type}[-{name}]`
- Main: `#{stub}-dev`
- Focus Groups: `#{stub}-fg-{name}`
- Implementations: `#{stub}-impl-{name}`

---

*WGSD section added during migration*
WGSD_SECTION
    
    echo "✅ PROJECT.md enhanced"
  else
    echo "ℹ️  PROJECT.md already has WGSD section"
  fi
fi
```

### Step 6: Transform ROADMAP.md to MASTER-ROADMAP.md

```bash
if [ -f ".planning/ROADMAP.md" ] && [ ! -f ".planning/MASTER-ROADMAP.md" ]; then
  echo "📝 Converting ROADMAP.md to MASTER-ROADMAP.md..."
  
  # Add header and rename
  {
    echo "# Master Roadmap"
    echo ""
    echo "**WGSD Project Roadmap** - Aggregated from all focus groups"
    echo ""
    echo "---"
    echo ""
    echo "*Migrated from GSD ROADMAP.md on $(date -I)*"
    echo ""
    cat .planning/ROADMAP.md
  } > .planning/MASTER-ROADMAP.md
  
  # Keep original as reference (don't delete)
  echo "✅ MASTER-ROADMAP.md created (original ROADMAP.md preserved)"
fi
```

### Step 7: Transform Phases to Focus Groups

```bash
if [ -d ".planning/phases" ]; then
  echo "📝 Transforming phases to focus groups..."
  
  for phase_dir in .planning/phases/*/; do
    if [ -d "$phase_dir" ]; then
      phase_name=$(basename "$phase_dir")
      
      # Create focus group directory
      fg_dir=".planning/focus-groups/$phase_name"
      mkdir -p "$fg_dir"
      mkdir -p "$fg_dir/concepts"
      
      # Copy phase contents
      if [ -f "${phase_dir}PLAN.md" ]; then
        cp "${phase_dir}PLAN.md" "$fg_dir/ROADMAP.md"
      fi
      
      # Copy any other files
      for file in "${phase_dir}"*; do
        if [ -f "$file" ]; then
          filename=$(basename "$file")
          if [ "$filename" != "PLAN.md" ]; then
            cp "$file" "$fg_dir/"
          fi
        fi
      done
      
      # Create focus group STATE.md
      cat > "$fg_dir/STATE.md" << EOF
# Focus Group: $phase_name

**Status:** Active
**Migrated From:** phases/$phase_name
**Migration Date:** $(date -I)

## Concepts

*Concepts will be listed here as they are created.*

## Notes

This focus group was created from GSD phase: $phase_name
EOF
      
      echo "   ✅ $phase_name → focus-groups/$phase_name"
    fi
  done
  
  echo "✅ Phases transformed to focus groups"
fi
```

### Step 8: Analyze REQUIREMENTS.md for Focus Group Suggestions

```bash
# Initialize suggested focus groups variable
SUGGESTED_FOCUS_GROUPS=""

if [ -f ".planning/REQUIREMENTS.md" ]; then
  echo "🔍 Analyzing REQUIREMENTS.md for focus group suggestions..."
  
  # Look for common domain keywords and build suggestions list
  if grep -qi "security\|auth\|permission\|access" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS security"
  fi
  if grep -qi "onboard\|setup\|wizard\|getting.started" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS onboarding"
  fi
  if grep -qi "billing\|payment\|subscription\|pricing" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS billing"
  fi
  if grep -qi "api\|integrat\|webhook\|endpoint" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS api"
  fi
  if grep -qi "ui\|frontend\|component\|design" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS frontend"
  fi
  if grep -qi "infra\|devops\|deploy\|ci.cd\|pipeline" .planning/REQUIREMENTS.md; then
    SUGGESTED_FOCUS_GROUPS="$SUGGESTED_FOCUS_GROUPS infrastructure"
  fi
  
  # Trim whitespace
  SUGGESTED_FOCUS_GROUPS=$(echo "$SUGGESTED_FOCUS_GROUPS" | xargs)
  
  if [ -n "$SUGGESTED_FOCUS_GROUPS" ]; then
    echo "💡 Suggested focus groups based on REQUIREMENTS.md:"
    for domain in $SUGGESTED_FOCUS_GROUPS; do
      echo "   - $domain"
    done
    echo ""
  else
    echo "ℹ️  No obvious domain groupings detected"
  fi
fi
```

### Step 9: Generate WGSD-CONFIG.md

```bash
if [ ! -f ".planning/WGSD-CONFIG.md" ]; then
  echo "📝 Creating WGSD-CONFIG.md..."
  
  # Try to detect stub from existing config or suggest
  PROJECT_NAME=$(basename "$(pwd)")
  SUGGESTED_STUB=$(echo "$PROJECT_NAME" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)
  
  cat > .planning/WGSD-CONFIG.md << EOF
# WGSD Configuration

**Project:** $PROJECT_NAME
**Migrated From:** GSD
**Migration Date:** $(date -I)

---

## Slack Configuration

**Stub:** $SUGGESTED_STUB
**Channel Prefix:** #${SUGGESTED_STUB}-

### Channel Patterns
| Type | Pattern | Example |
|------|---------|---------|
| Main Dev | \`${SUGGESTED_STUB}-dev\` | #${SUGGESTED_STUB}-dev |
| Community | \`${SUGGESTED_STUB}-community\` | #${SUGGESTED_STUB}-community |
| Focus Group | \`${SUGGESTED_STUB}-fg-{name}\` | #${SUGGESTED_STUB}-fg-security |
| Concept | \`${SUGGESTED_STUB}-cpt-{name}\` | #${SUGGESTED_STUB}-cpt-byof |
| Implementation | \`${SUGGESTED_STUB}-impl-{name}\` | #${SUGGESTED_STUB}-impl-auth-v2 |

> **Note:** Update the stub value if needed. Run \`wgsd init\` to change.

---

## Channel Registry

| Channel | ID | Type | Status | Created |
|---------|-----|------|--------|---------|

---

## Git Configuration

**Primary Branch:** (auto-detected)
**Focus Groups Base:** focus-groups/
**Implementations Base:** implementations/

---

## Migration Notes

- Original GSD structure backed up to: ${BACKUP_DIR:-"(no backup needed)"}
- Requirements analyzed for focus group suggestions
- Phase structure converted to focus groups (if present)

---

*Configuration generated during GSD → WGSD migration*
EOF
  
  echo "✅ WGSD-CONFIG.md created"
fi
```

### Step 10: Create Core Slack Channels

```bash
if [ "$SLACK_ENABLED" = "true" ]; then
  echo ""
  echo "═══════════════════════════════════════════════════"
  echo "📦 Step 10: Creating Core Slack Channels"
  echo "═══════════════════════════════════════════════════"
  echo ""
  
  STUB="$SUGGESTED_STUB"
  DEV_CHANNEL="${STUB}-dev"
  COMMUNITY_CHANNEL="${STUB}-community"
  
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
    
    # Add to channel registry
    sed -i "/^| Channel | ID | Type | Status | Created |$/a | $DEV_CHANNEL | $DEV_ID | dev | active | $(date +%Y-%m-%d) |" .planning/WGSD-CONFIG.md
      
  elif [ "$DEV_ERROR" = "name_taken" ]; then
    echo "   ℹ️  Already exists: #$DEV_CHANNEL"
    EXISTING_CHANNELS="$EXISTING_CHANNELS $DEV_CHANNEL"
    # Find existing ID
    DEV_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=private_channel&limit=500" \
      -H "Authorization: Bearer $SLACK_TOKEN" | \
      jq -r --arg name "$DEV_CHANNEL" '.channels[] | select(.name == $name) | .id')
    
    # Add to registry if ID found
    if [ -n "$DEV_ID" ] && [ "$DEV_ID" != "null" ]; then
      sed -i "/^| Channel | ID | Type | Status | Created |$/a | $DEV_CHANNEL | $DEV_ID | dev | active | existing |" .planning/WGSD-CONFIG.md
    fi
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
    
    # Add to channel registry
    sed -i "/^| Channel | ID | Type | Status | Created |$/a | $COMMUNITY_CHANNEL | $COMMUNITY_ID | community | active | $(date +%Y-%m-%d) |" .planning/WGSD-CONFIG.md
      
  elif [ "$COMMUNITY_ERROR" = "name_taken" ]; then
    echo "   ℹ️  Already exists: #$COMMUNITY_CHANNEL"
    EXISTING_CHANNELS="$EXISTING_CHANNELS $COMMUNITY_CHANNEL"
    COMMUNITY_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel&limit=500" \
      -H "Authorization: Bearer $SLACK_TOKEN" | \
      jq -r --arg name "$COMMUNITY_CHANNEL" '.channels[] | select(.name == $name) | .id')
    
    # Add to registry if ID found
    if [ -n "$COMMUNITY_ID" ] && [ "$COMMUNITY_ID" != "null" ]; then
      sed -i "/^| Channel | ID | Type | Status | Created |$/a | $COMMUNITY_CHANNEL | $COMMUNITY_ID | community | active | existing |" .planning/WGSD-CONFIG.md
    fi
  else
    echo "   ❌ Failed: $COMMUNITY_ERROR"
    FAILED_CHANNELS="$FAILED_CHANNELS $COMMUNITY_CHANNEL"
  fi
  
  echo ""
fi
```

### Step 11: Create Focus Group Channels

```bash
if [ "$SLACK_ENABLED" = "true" ] && [ -n "$SUGGESTED_FOCUS_GROUPS" ]; then
  echo "═══════════════════════════════════════════════════"
  echo "🎯 Step 11: Creating Focus Group Channels"
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
      
      # Add to channel registry
      sed -i "/^| Channel | ID | Type | Status | Created |$/a | $FG_CHANNEL | $FG_ID | focus-group | active | $(date +%Y-%m-%d) |" .planning/WGSD-CONFIG.md
        
    elif [ "$FG_ERROR" = "name_taken" ]; then
      echo "   ℹ️  Already exists: #$FG_CHANNEL"
      EXISTING_CHANNELS="$EXISTING_CHANNELS $FG_CHANNEL"
      
      # Find existing ID and add to registry
      FG_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=private_channel&limit=500" \
        -H "Authorization: Bearer $SLACK_TOKEN" | \
        jq -r --arg name "$FG_CHANNEL" '.channels[] | select(.name == $name) | .id')
      
      if [ -n "$FG_ID" ] && [ "$FG_ID" != "null" ]; then
        sed -i "/^| Channel | ID | Type | Status | Created |$/a | $FG_CHANNEL | $FG_ID | focus-group | active | existing |" .planning/WGSD-CONFIG.md
      fi
    else
      echo "   ❌ Failed: $FG_ERROR"
      FAILED_CHANNELS="$FAILED_CHANNELS $FG_CHANNEL"
    fi
    
    sleep 1  # Rate limit courtesy
  done
  
  echo ""
fi
```

### Step 12: Update STATE.md

```bash
if [ -f ".planning/STATE.md" ]; then
  echo "📝 Adding WGSD section to STATE.md..."
  
  if ! grep -q "## WGSD Status" .planning/STATE.md; then
    cat >> .planning/STATE.md << 'WGSD_STATE'

---

## WGSD Status

### Migration
- **Status:** ✅ Complete
- **Date:** Migrated on $(date -I)
- **Mode:** GSD → WGSD

### Focus Groups
*Focus groups will be listed as they become active.*

### Implementations  
*Active implementations listed here (max 2-4).*

---

*WGSD status section added during migration*
WGSD_STATE
    
    echo "✅ STATE.md updated"
  fi
fi
```

### Step 13: Validate Migration

```bash
echo ""
echo "🔍 Validating migration..."

ERRORS=""

# Check required files/directories
[ ! -d ".planning/focus-groups" ] && ERRORS="$ERRORS\n❌ Missing focus-groups/"
[ ! -d ".planning/active-implementations" ] && ERRORS="$ERRORS\n❌ Missing active-implementations/"
[ ! -f ".planning/WGSD-CONFIG.md" ] && ERRORS="$ERRORS\n❌ Missing WGSD-CONFIG.md"

if [ -n "$ERRORS" ]; then
  echo "Migration validation failed:"
  echo -e "$ERRORS"
  exit 1
fi

echo "✅ Migration validated successfully"
```

### Step 14: Commit Migration

```bash
echo ""
echo "📦 Committing migration..."

# Build commit message based on what happened
COMMIT_MSG="feat: migrate planning structure from GSD to WGSD

- Added WGSD directory structure (focus-groups/, active-implementations/)
- Created WGSD-CONFIG.md with project configuration
- Enhanced PROJECT.md with WGSD workflow section
- Transformed phases to focus groups (if present)
- Created MASTER-ROADMAP.md from existing roadmap
- Updated STATE.md with WGSD status section"

if [ "$SLACK_ENABLED" = "true" ]; then
  COMMIT_MSG="$COMMIT_MSG
- Auto-created Slack channels:$CREATED_CHANNELS"
fi

COMMIT_MSG="$COMMIT_MSG

Migration backup: ${BACKUP_DIR:-"(no backup needed)"}"

git add .planning/
git commit -m "$COMMIT_MSG"

if [ $? -eq 0 ]; then
  echo "✅ Migration committed"
else
  echo "⚠️  Commit failed (may already be committed or nothing to commit)"
fi
```

### Step 15: Report Success

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🎉 GSD → WGSD Migration Complete"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Structure:"
echo "   .planning/"
echo "   ├── focus-groups/           ✅"
echo "   ├── active-implementations/ ✅"
echo "   ├── WGSD-CONFIG.md          ✅"
echo "   └── MASTER-ROADMAP.md       ✅"
echo ""

if [ "$SLACK_ENABLED" = "true" ]; then
  echo "📱 Slack Channels:"
  [ -n "$CREATED_CHANNELS" ] && echo "   ✅ Created:$CREATED_CHANNELS"
  [ -n "$EXISTING_CHANNELS" ] && echo "   ℹ️  Existing:$EXISTING_CHANNELS"
  [ -n "$FAILED_CHANNELS" ] && echo "   ❌ Failed:$FAILED_CHANNELS"
  echo ""
  echo "   🔗 Channels are ready to use!"
  echo ""
else
  echo "📱 Slack Channels:"
  echo "   ⚠️  Skipped (no token configured)"
  echo ""
  echo "   Create manually:"
  echo "   /wgsd setup-core-channels $SUGGESTED_STUB"
  echo ""
fi

if [ -n "${BACKUP_DIR:-}" ]; then
  echo "📦 Backup: $BACKUP_DIR"
  echo ""
fi

echo "Next Steps:"
echo "  1. Review WGSD-CONFIG.md"
if [ "$SLACK_ENABLED" = "true" ]; then
  echo "  2. Join your channels in Slack"
  echo "  3. Start creating concepts: /wgsd create-concept <name>"
else
  echo "  2. Create Slack channels (see above)"
  echo "  3. Start creating concepts: /wgsd create-concept <name>"
fi
echo ""
```

---

## Rollback

If migration fails and you need to restore:

```bash
# Find backup directory
BACKUP=$(ls -d .planning-backup-* | tail -1)

# Restore
rm -rf .planning
mv "$BACKUP" .planning

# Reset git
git checkout -- .planning/

echo "Rollback complete"
```

---

## Success Criteria

- [ ] GSD structure detected and analyzed
- [ ] Backup created before modifications
- [ ] WGSD directory structure created
- [ ] WGSD-CONFIG.md generated
- [ ] Content transformed and preserved
- [ ] Slack channels auto-created (if token available)
- [ ] Migration committed to git

---

*Workflow updated with Slack channel automation (Phase 8)*
