---
name: wgsd:migrate-planning
description: Migrate GSD planning structure to WGSD format with interactive approval
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUser
---

# GSD to WGSD Planning Migration (v2.1)

Transform existing GSD `.planning/` structure to WGSD format with interactive preview and approval.

---

## Objective

Migrate a GSD-structured project to WGSD by:
- Detecting existing GSD planning files
- Creating backup before modifications
- **Analyzing and consolidating migration data**
- **Generating interactive preview**
- **Requiring explicit approval before execution**
- **Supporting edit mode for modifications**
- Transforming content to WGSD structure
- Generating WGSD configuration files
- Automatically creating approved Slack channels
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

### Step 4: Comprehensive Analysis (Consolidate All Data)

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "📊 Step 4: Analyzing Project for Migration"
echo "═══════════════════════════════════════════════════"
echo ""

# Initialize data arrays
FOCUS_GROUPS=()
CONCEPTS=()
FILES_TO_TRANSFORM=()
CHANNELS=()

# Determine project stub
PROJECT_NAME=$(basename "$(pwd)")
SUGGESTED_STUB=$(echo "$PROJECT_NAME" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)

# ─── Extract Phases from ROADMAP.md → Concepts ───
if [ -f ".planning/ROADMAP.md" ]; then
  echo "📄 Analyzing ROADMAP.md..."
  
  PHASE_NUM=0
  while IFS= read -r phase_line; do
    [ -z "$phase_line" ] && continue
    PHASE_NUM=$((PHASE_NUM + 1))
    
    # Generate concept slug from phase title
    concept_slug=$(echo "$phase_line" | \
      sed 's/^##*[[:space:]]*//' | \
      sed 's/^Phase[ -]*[0-9]*[: -]*//' | \
      tr '[:upper:]' '[:lower:]' | \
      tr ' ' '-' | \
      tr -cd 'a-z0-9-' | \
      sed 's/--*/-/g' | \
      sed 's/^-//' | sed 's/-$//' | \
      cut -c1-25)
    
    [ -z "$concept_slug" ] && concept_slug="phase-$PHASE_NUM"
    
    # Store: slug:focus_group:source:status
    # Focus group assigned later after domain analysis
    CONCEPTS+=("$concept_slug:unassigned:Phase $PHASE_NUM:include")
    
    echo "   💡 Phase $PHASE_NUM → Concept: $concept_slug"
  done <<< "$(grep -E "^##? Phase [0-9]+:" .planning/ROADMAP.md | head -15)"
  
  CONCEPT_COUNT=${#CONCEPTS[@]}
  echo "   📝 Found $CONCEPT_COUNT concepts from ROADMAP.md"
  echo ""
fi

# ─── Analyze REQUIREMENTS.md for Focus Groups ───
echo "📄 Analyzing REQUIREMENTS.md for focus group suggestions..."

if [ -f ".planning/REQUIREMENTS.md" ]; then
  REQ_CONTENT=$(cat .planning/REQUIREMENTS.md | tr '[:upper:]' '[:lower:]')
  
  # Check for domain keywords
  if echo "$REQ_CONTENT" | grep -qE "security|auth|permission|access|login|password"; then
    FOCUS_GROUPS+=("security:requirements")
    echo "   🎯 security ← (auth, permission, access keywords)"
  fi
  if echo "$REQ_CONTENT" | grep -qE "api|integrat|webhook|endpoint|rest|graphql"; then
    FOCUS_GROUPS+=("api:requirements")
    echo "   🎯 api ← (api, webhook, integration keywords)"
  fi
  if echo "$REQ_CONTENT" | grep -qE "infra|devops|deploy|ci.cd|pipeline|docker"; then
    FOCUS_GROUPS+=("infrastructure:requirements")
    echo "   🎯 infrastructure ← (deploy, ci/cd, devops keywords)"
  fi
  if echo "$REQ_CONTENT" | grep -qE "ui|frontend|component|design|ux|interface"; then
    FOCUS_GROUPS+=("frontend:requirements")
    echo "   🎯 frontend ← (ui, component, design keywords)"
  fi
  if echo "$REQ_CONTENT" | grep -qE "onboard|setup|wizard|getting.started"; then
    FOCUS_GROUPS+=("onboarding:requirements")
    echo "   🎯 onboarding ← (setup, wizard, onboarding keywords)"
  fi
  if echo "$REQ_CONTENT" | grep -qE "billing|payment|subscription|pricing"; then
    FOCUS_GROUPS+=("billing:requirements")
    echo "   🎯 billing ← (billing, payment, pricing keywords)"
  fi
fi

# Add "core" if no focus groups detected
if [ ${#FOCUS_GROUPS[@]} -eq 0 ]; then
  FOCUS_GROUPS+=("core:default")
  echo "   🎯 core ← (default focus group)"
fi

FG_COUNT=${#FOCUS_GROUPS[@]}
echo "   📂 Suggested $FG_COUNT focus groups"
echo ""

# ─── Assign Concepts to Focus Groups ───
echo "🔗 Mapping concepts to focus groups..."

# Build list of focus group names
FG_NAMES=()
for fg_entry in "${FOCUS_GROUPS[@]}"; do
  fg_name="${fg_entry%%:*}"
  FG_NAMES+=("$fg_name")
done

# Default focus group (first one or 'core')
DEFAULT_FG="${FG_NAMES[0]:-core}"

# Update concepts with focus group assignment
UPDATED_CONCEPTS=()
for concept_entry in "${CONCEPTS[@]}"; do
  slug="${concept_entry%%:*}"
  rest="${concept_entry#*:}"
  source="${rest#*:}"
  source="${source%%:*}"
  status="${concept_entry##*:}"
  
  # Simple keyword matching for assignment
  concept_lower=$(echo "$slug" | tr '[:upper:]' '[:lower:]')
  assigned_fg="$DEFAULT_FG"
  
  if echo "$concept_lower" | grep -qE "auth|security|permission|access|login"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "security")" ] && assigned_fg="security"
  elif echo "$concept_lower" | grep -qE "api|webhook|endpoint|integrat"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "api")" ] && assigned_fg="api"
  elif echo "$concept_lower" | grep -qE "infra|deploy|ci|cd|pipeline|docker"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "infrastructure")" ] && assigned_fg="infrastructure"
  elif echo "$concept_lower" | grep -qE "ui|frontend|component|design"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "frontend")" ] && assigned_fg="frontend"
  elif echo "$concept_lower" | grep -qE "onboard|setup|wizard"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "onboarding")" ] && assigned_fg="onboarding"
  elif echo "$concept_lower" | grep -qE "billing|payment|subscription"; then
    [ -n "$(printf '%s\n' "${FG_NAMES[@]}" | grep -x "billing")" ] && assigned_fg="billing"
  fi
  
  UPDATED_CONCEPTS+=("$slug:$assigned_fg:$source:$status")
  echo "   💡 $slug → $assigned_fg"
done
CONCEPTS=("${UPDATED_CONCEPTS[@]}")

echo ""

# ─── Build Channels List ───
echo "📱 Planning Slack channels..."

# Core channels
CHANNELS+=("${SUGGESTED_STUB}-dev:core:private")
CHANNELS+=("${SUGGESTED_STUB}-community:community:public")

# Focus group channels
for fg_entry in "${FOCUS_GROUPS[@]}"; do
  fg_name="${fg_entry%%:*}"
  CHANNELS+=("${SUGGESTED_STUB}-fg-${fg_name}:focus-group:private")
done

CHANNEL_COUNT=${#CHANNELS[@]}
echo "   📱 Planning $CHANNEL_COUNT channels"
echo ""

# ─── Track Files to Transform ───
FILES_TO_TRANSFORM=()
[ -f ".planning/PROJECT.md" ] && FILES_TO_TRANSFORM+=("PROJECT.md:enhance")
[ -f ".planning/ROADMAP.md" ] && FILES_TO_TRANSFORM+=("ROADMAP.md:create-master")
[ -f ".planning/STATE.md" ] && FILES_TO_TRANSFORM+=("STATE.md:add-wgsd-section")
FILES_TO_TRANSFORM+=("WGSD-CONFIG.md:create-new")

echo "✅ Analysis complete"
echo ""
```

### Step 5: Generate Preview Function

```bash
# ─── Preview Generation Function ───
generate_preview() {
  echo ""
  echo "═══════════════════════════════════════════════════════════════"
  echo "📋 MIGRATION PREVIEW"
  echo "═══════════════════════════════════════════════════════════════"
  echo ""
  echo "📁 Project: $(basename "$(pwd)")"
  echo "📍 Source: GSD (.planning/)"
  echo "📱 Stub: $SUGGESTED_STUB"
  echo ""
  
  # ─── Focus Groups Section ───
  echo "───────────────────────────────────────────────────────────────"
  echo "🎯 FOCUS GROUPS TO CREATE (${#FOCUS_GROUPS[@]})"
  echo "───────────────────────────────────────────────────────────────"
  
  idx=1
  for fg_entry in "${FOCUS_GROUPS[@]}"; do
    fg_name="${fg_entry%%:*}"
    fg_source="${fg_entry#*:}"
    printf "  [%d] %-20s ← %s\n" "$idx" "$fg_name" "$fg_source"
    idx=$((idx + 1))
  done
  echo ""
  
  # ─── Concepts Section ───
  echo "───────────────────────────────────────────────────────────────"
  echo "💡 CONCEPTS TO CREATE (${#CONCEPTS[@]})"
  echo "───────────────────────────────────────────────────────────────"
  
  idx=1
  for concept_entry in "${CONCEPTS[@]}"; do
    IFS=':' read -r slug fg source status <<< "$concept_entry"
    if [ "$status" = "include" ]; then
      printf "  [%d] %-20s → %-15s ← %s\n" "$idx" "$slug" "$fg" "$source"
    else
      printf "  [%d] %-20s (%s)\n" "$idx" "$slug" "EXCLUDED"
    fi
    idx=$((idx + 1))
  done
  echo ""
  
  # ─── Slack Channels Section ───
  echo "───────────────────────────────────────────────────────────────"
  echo "📱 SLACK CHANNELS TO CREATE"
  echo "───────────────────────────────────────────────────────────────"
  
  if [ "$SLACK_ENABLED" = "true" ]; then
    echo "  ✅ Slack connected (bot: $BOT_NAME)"
    echo ""
    echo "  Core:"
    for ch_entry in "${CHANNELS[@]}"; do
      IFS=':' read -r ch_name ch_type ch_vis <<< "$ch_entry"
      if [ "$ch_type" = "core" ] || [ "$ch_type" = "community" ]; then
        printf "    • #%-25s (%s)\n" "$ch_name" "$ch_vis"
      fi
    done
    echo ""
    echo "  Focus Groups:"
    for ch_entry in "${CHANNELS[@]}"; do
      IFS=':' read -r ch_name ch_type ch_vis <<< "$ch_entry"
      [ "$ch_type" = "focus-group" ] && \
        printf "    • #%-25s (%s)\n" "$ch_name" "$ch_vis"
    done
  else
    echo "  ⚠️  Slack not configured (channels will be created manually)"
  fi
  echo ""
  
  # ─── Files Section ───
  echo "───────────────────────────────────────────────────────────────"
  echo "📄 FILES TO TRANSFORM"
  echo "───────────────────────────────────────────────────────────────"
  
  for file_entry in "${FILES_TO_TRANSFORM[@]}"; do
    IFS=':' read -r filename action <<< "$file_entry"
    case "$action" in
      enhance) printf "  • %-20s → Add WGSD workflow section\n" "$filename" ;;
      create-master) printf "  • %-20s → Create MASTER-ROADMAP.md\n" "$filename" ;;
      add-wgsd-section) printf "  • %-20s → Add WGSD status section\n" "$filename" ;;
      create-new) printf "  • %-20s → Create new file\n" "$filename" ;;
    esac
  done
  echo ""
  
  echo "═══════════════════════════════════════════════════════════════"
}
```

### Step 6: Display Preview

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "📋 Step 5: Migration Preview"
echo "═══════════════════════════════════════════════════"

generate_preview
```

### Step 7: Approval Loop with Edit Mode

```bash
echo ""
echo "───────────────────────────────────────────────────────────────"
echo "⚙️  APPROVAL OPTIONS"
echo "───────────────────────────────────────────────────────────────"
echo ""
echo "  [yes]    Approve and proceed with migration"
echo "  [no]     Cancel migration"
echo "  [edit]   Modify suggestions before approval"
echo ""

APPROVED="false"

while [ "$APPROVED" = "false" ]; do
  read -p "Enter choice (yes/no/edit): " approval_choice
  
  case "$approval_choice" in
    yes|YES|Yes|y|Y)
      echo ""
      echo "✅ Migration approved"
      APPROVED="true"
      ;;
      
    no|NO|No|n|N)
      echo ""
      echo "❌ Migration cancelled by user"
      echo ""
      echo "Your project is unchanged. Run /wgsd migrate again when ready."
      
      # Clean up backup if we created one
      if [ -n "${BACKUP_DIR:-}" ] && [ -d "$BACKUP_DIR" ]; then
        rm -rf "$BACKUP_DIR"
        echo "🧹 Backup removed: $BACKUP_DIR"
      fi
      
      exit 0
      ;;
      
    edit|EDIT|Edit|e|E)
      echo ""
      echo "═══════════════════════════════════════════════════"
      echo "✏️  EDIT MODE"
      echo "═══════════════════════════════════════════════════"
      echo ""
      echo "Commands:"
      echo "  rename-fg <num> <new-name>    Rename focus group"
      echo "  remove-fg <num>               Remove focus group"
      echo "  add-fg <name>                 Add new focus group"
      echo "  move-concept <num> <fg-name>  Move concept to different focus group"
      echo "  exclude-concept <num>         Exclude concept from migration"
      echo "  include-concept <num>         Re-include excluded concept"
      echo "  done                          Finish editing and show updated preview"
      echo ""
      
      while true; do
        read -p "edit> " edit_cmd edit_arg1 edit_arg2
        
        case "$edit_cmd" in
          rename-fg)
            if [ -n "$edit_arg1" ] && [ -n "$edit_arg2" ]; then
              idx=$((edit_arg1 - 1))
              if [ $idx -ge 0 ] && [ $idx -lt ${#FOCUS_GROUPS[@]} ]; then
                old_entry="${FOCUS_GROUPS[$idx]}"
                old_name="${old_entry%%:*}"
                old_source="${old_entry#*:}"
                FOCUS_GROUPS[$idx]="$edit_arg2:$old_source"
                
                # Update channel name
                for i in "${!CHANNELS[@]}"; do
                  ch_entry="${CHANNELS[$i]}"
                  if [[ "$ch_entry" == *"-fg-$old_name:"* ]]; then
                    new_ch="${ch_entry/$old_name/$edit_arg2}"
                    CHANNELS[$i]="$new_ch"
                  fi
                done
                
                # Update concepts assigned to this focus group
                for i in "${!CONCEPTS[@]}"; do
                  IFS=':' read -r slug fg source status <<< "${CONCEPTS[$i]}"
                  if [ "$fg" = "$old_name" ]; then
                    CONCEPTS[$i]="$slug:$edit_arg2:$source:$status"
                  fi
                done
                
                # Update FG_NAMES array
                FG_NAMES=()
                for fg_entry in "${FOCUS_GROUPS[@]}"; do
                  fg_name="${fg_entry%%:*}"
                  FG_NAMES+=("$fg_name")
                done
                
                echo "   ✅ Renamed '$old_name' → '$edit_arg2'"
              else
                echo "   ❌ Invalid focus group number"
              fi
            else
              echo "   Usage: rename-fg <num> <new-name>"
            fi
            ;;
            
          remove-fg)
            if [ -n "$edit_arg1" ]; then
              idx=$((edit_arg1 - 1))
              if [ $idx -ge 0 ] && [ $idx -lt ${#FOCUS_GROUPS[@]} ]; then
                removed="${FOCUS_GROUPS[$idx]}"
                removed_name="${removed%%:*}"
                
                # Remove focus group
                unset 'FOCUS_GROUPS[$idx]'
                FOCUS_GROUPS=("${FOCUS_GROUPS[@]}")  # Re-index
                
                # Remove associated channel
                NEW_CHANNELS=()
                for ch in "${CHANNELS[@]}"; do
                  [[ "$ch" != *"-fg-$removed_name:"* ]] && NEW_CHANNELS+=("$ch")
                done
                CHANNELS=("${NEW_CHANNELS[@]}")
                
                # Move orphaned concepts to first remaining FG
                if [ ${#FOCUS_GROUPS[@]} -gt 0 ]; then
                  new_fg="${FOCUS_GROUPS[0]%%:*}"
                  for i in "${!CONCEPTS[@]}"; do
                    IFS=':' read -r slug fg source status <<< "${CONCEPTS[$i]}"
                    if [ "$fg" = "$removed_name" ]; then
                      CONCEPTS[$i]="$slug:$new_fg:$source:$status"
                    fi
                  done
                fi
                
                # Update FG_NAMES array
                FG_NAMES=()
                for fg_entry in "${FOCUS_GROUPS[@]}"; do
                  fg_name="${fg_entry%%:*}"
                  FG_NAMES+=("$fg_name")
                done
                
                echo "   ✅ Removed focus group '$removed_name'"
              else
                echo "   ❌ Invalid focus group number"
              fi
            else
              echo "   Usage: remove-fg <num>"
            fi
            ;;
            
          add-fg)
            if [ -n "$edit_arg1" ]; then
              FOCUS_GROUPS+=("$edit_arg1:user-added")
              CHANNELS+=("${SUGGESTED_STUB}-fg-${edit_arg1}:focus-group:private")
              FG_NAMES+=("$edit_arg1")
              echo "   ✅ Added focus group '$edit_arg1'"
            else
              echo "   Usage: add-fg <name>"
            fi
            ;;
            
          move-concept)
            if [ -n "$edit_arg1" ] && [ -n "$edit_arg2" ]; then
              idx=$((edit_arg1 - 1))
              if [ $idx -ge 0 ] && [ $idx -lt ${#CONCEPTS[@]} ]; then
                IFS=':' read -r slug fg source status <<< "${CONCEPTS[$idx]}"
                CONCEPTS[$idx]="$slug:$edit_arg2:$source:$status"
                echo "   ✅ Moved '$slug' → $edit_arg2"
              else
                echo "   ❌ Invalid concept number"
              fi
            else
              echo "   Usage: move-concept <num> <focus-group-name>"
            fi
            ;;
            
          exclude-concept)
            if [ -n "$edit_arg1" ]; then
              idx=$((edit_arg1 - 1))
              if [ $idx -ge 0 ] && [ $idx -lt ${#CONCEPTS[@]} ]; then
                IFS=':' read -r slug fg source status <<< "${CONCEPTS[$idx]}"
                CONCEPTS[$idx]="$slug:$fg:$source:exclude"
                echo "   ✅ Excluded '$slug' from migration"
              else
                echo "   ❌ Invalid concept number"
              fi
            else
              echo "   Usage: exclude-concept <num>"
            fi
            ;;
            
          include-concept)
            if [ -n "$edit_arg1" ]; then
              idx=$((edit_arg1 - 1))
              if [ $idx -ge 0 ] && [ $idx -lt ${#CONCEPTS[@]} ]; then
                IFS=':' read -r slug fg source status <<< "${CONCEPTS[$idx]}"
                CONCEPTS[$idx]="$slug:$fg:$source:include"
                echo "   ✅ Re-included '$slug' in migration"
              else
                echo "   ❌ Invalid concept number"
              fi
            else
              echo "   Usage: include-concept <num>"
            fi
            ;;
            
          done|DONE|Done|d|D)
            echo ""
            echo "───────────────────────────────────────────────────"
            echo "📋 UPDATED PREVIEW"
            echo "───────────────────────────────────────────────────"
            generate_preview
            
            echo ""
            echo "───────────────────────────────────────────────────"
            echo "⚙️  APPROVAL OPTIONS"
            echo "───────────────────────────────────────────────────"
            echo ""
            echo "  [yes]    Approve and proceed with migration"
            echo "  [no]     Cancel migration"
            echo "  [edit]   Continue editing"
            echo ""
            break
            ;;
            
          help|h|?)
            echo ""
            echo "Commands:"
            echo "  rename-fg <num> <new-name>    Rename focus group"
            echo "  remove-fg <num>               Remove focus group"
            echo "  add-fg <name>                 Add new focus group"
            echo "  move-concept <num> <fg-name>  Move concept to different focus group"
            echo "  exclude-concept <num>         Exclude concept from migration"
            echo "  include-concept <num>         Re-include excluded concept"
            echo "  done                          Finish editing and show updated preview"
            echo ""
            ;;
            
          "")
            # Ignore empty input
            ;;
            
          *)
            echo "   ❌ Unknown command. Type 'help' for available commands or 'done' to finish editing."
            ;;
        esac
      done
      ;;
      
    *)
      echo "   Please enter 'yes', 'no', or 'edit'"
      ;;
  esac
done

echo ""
echo "═══════════════════════════════════════════════════"
echo "🚀 EXECUTING APPROVED MIGRATION"
echo "═══════════════════════════════════════════════════"
echo ""
```

### Step 8: Create WGSD Directory Structure

```bash
echo "📁 Creating WGSD directory structure..."

mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations

echo "✅ Directory structure created"
```

### Step 9: Transform PROJECT.md

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

### Step 10: Transform ROADMAP.md to MASTER-ROADMAP.md

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

### Step 11: Create Approved Focus Groups

```bash
echo ""
echo "📁 Creating approved focus groups..."

for fg_entry in "${FOCUS_GROUPS[@]}"; do
  fg_name="${fg_entry%%:*}"
  fg_dir=".planning/focus-groups/$fg_name"
  
  mkdir -p "$fg_dir/concepts"
  
  cat > "$fg_dir/STATE.md" << EOF
# Focus Group: $fg_name

**Status:** Active
**Created:** $(date -I)
**Migration:** WGSD v2.1

## Concepts

*Concepts listed below after migration*

## Channel

#${SUGGESTED_STUB}-fg-${fg_name}

---
*Focus group created during WGSD migration*
EOF

  echo "   ✅ Created: focus-groups/$fg_name/"
done
```

### Step 12: Create Approved Concept Files

```bash
echo ""
echo "💡 Creating approved concept files..."

INCLUDED_COUNT=0
EXCLUDED_COUNT=0

for concept_entry in "${CONCEPTS[@]}"; do
  IFS=':' read -r slug fg source status <<< "$concept_entry"
  
  # Skip excluded concepts
  if [ "$status" != "include" ]; then
    EXCLUDED_COUNT=$((EXCLUDED_COUNT + 1))
    echo "   ⏭️  Skipped (excluded): $slug"
    continue
  fi
  
  # Ensure focus group directory exists
  fg_dir=".planning/focus-groups/$fg"
  mkdir -p "$fg_dir/concepts"
  
  concept_file="$fg_dir/concepts/$slug.md"
  
  cat > "$concept_file" << EOF
# Concept: $slug

**Status:** Migrated
**Focus Group:** $fg
**Source:** $source
**Migrated:** $(date -I)

---

## Description

*Description from $source - to be refined*

---

## Acceptance Criteria

*Requirements from $source - to be refined*

---

## Implementation Notes

*To be populated during concept development*

---

*Concept migrated from GSD during WGSD v2.1 migration*
EOF

  echo "   💡 Created: $fg/concepts/$slug.md"
  INCLUDED_COUNT=$((INCLUDED_COUNT + 1))
done

echo "   📝 Created $INCLUDED_COUNT concept files ($EXCLUDED_COUNT excluded)"
```

### Step 13: Generate WGSD-CONFIG.md

```bash
if [ ! -f ".planning/WGSD-CONFIG.md" ]; then
  echo ""
  echo "📝 Creating WGSD-CONFIG.md..."
  
  # Build focus group list for config
  FG_LIST=""
  for fg_entry in "${FOCUS_GROUPS[@]}"; do
    fg_name="${fg_entry%%:*}"
    FG_LIST="$FG_LIST\n- $fg_name"
  done
  
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

## Focus Groups

$(echo -e "$FG_LIST")

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
- Concepts created: $INCLUDED_COUNT
- Concepts excluded: $EXCLUDED_COUNT
- Focus groups created: ${#FOCUS_GROUPS[@]}
- Migration approved by user

---

*Configuration generated during GSD → WGSD migration*
EOF
  
  echo "✅ WGSD-CONFIG.md created"
fi
```

### Step 14: Create Approved Slack Channels

```bash
if [ "$SLACK_ENABLED" = "true" ]; then
  echo ""
  echo "═══════════════════════════════════════════════════"
  echo "📱 Creating Approved Slack Channels"
  echo "═══════════════════════════════════════════════════"
  echo ""
  
  for ch_entry in "${CHANNELS[@]}"; do
    IFS=':' read -r ch_name ch_type ch_vis <<< "$ch_entry"
    
    is_private="true"
    [ "$ch_vis" = "public" ] && is_private="false"
    
    echo "📱 Creating: #$ch_name ($ch_vis)"
    
    RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"name\":\"$ch_name\",\"is_private\":$is_private}")
    
    CH_OK=$(echo "$RESPONSE" | jq -r '.ok')
    CH_ERROR=$(echo "$RESPONSE" | jq -r '.error // empty')
    
    if [ "$CH_OK" = "true" ]; then
      CH_ID=$(echo "$RESPONSE" | jq -r '.channel.id')
      echo "   ✅ Created: #$ch_name ($CH_ID)"
      CREATED_CHANNELS="$CREATED_CHANNELS $ch_name"
      
      # Set topic based on channel type
      case "$ch_type" in
        core)
          TOPIC="🛠️ Core development | WGSD managed"
          PURPOSE="Main development coordination for $SUGGESTED_STUB. Architecture, planning, cross-cutting concerns."
          ;;
        community)
          TOPIC="💬 Community feedback and discussion | Public"
          PURPOSE="Public channel for community feedback, feature requests, and discussion."
          ;;
        focus-group)
          fg_name="${ch_name##*-fg-}"
          TOPIC="🎯 Focus Group: $fg_name | Planning & ideation"
          PURPOSE="Long-lived focus group for $fg_name development. Concepts developed here before implementation."
          ;;
      esac
      
      curl -s -X POST https://slack.com/api/conversations.setTopic \
        -H "Authorization: Bearer $SLACK_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"channel\":\"$CH_ID\",\"topic\":\"$TOPIC\"}" >/dev/null
        
      curl -s -X POST https://slack.com/api/conversations.setPurpose \
        -H "Authorization: Bearer $SLACK_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"channel\":\"$CH_ID\",\"purpose\":\"$PURPOSE\"}" >/dev/null
      
      # Add to channel registry
      sed -i "/^| Channel | ID | Type | Status | Created |$/a | $ch_name | $CH_ID | $ch_type | active | $(date +%Y-%m-%d) |" .planning/WGSD-CONFIG.md
        
    elif [ "$CH_ERROR" = "name_taken" ]; then
      echo "   ℹ️  Already exists: #$ch_name"
      EXISTING_CHANNELS="$EXISTING_CHANNELS $ch_name"
      
      # Try to find existing ID
      if [ "$is_private" = "true" ]; then
        CH_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=private_channel&limit=500" \
          -H "Authorization: Bearer $SLACK_TOKEN" | \
          jq -r --arg name "$ch_name" '.channels[] | select(.name == $name) | .id')
      else
        CH_ID=$(curl -s -X GET "https://slack.com/api/conversations.list?types=public_channel&limit=500" \
          -H "Authorization: Bearer $SLACK_TOKEN" | \
          jq -r --arg name "$ch_name" '.channels[] | select(.name == $name) | .id')
      fi
      
      if [ -n "$CH_ID" ] && [ "$CH_ID" != "null" ]; then
        sed -i "/^| Channel | ID | Type | Status | Created |$/a | $ch_name | $CH_ID | $ch_type | active | existing |" .planning/WGSD-CONFIG.md
      fi
    else
      echo "   ❌ Failed: $CH_ERROR"
      FAILED_CHANNELS="$FAILED_CHANNELS $ch_name"
    fi
    
    sleep 1  # Rate limit courtesy
  done
  
  echo ""
fi
```

### Step 15: Update STATE.md

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

### Step 16: Validate Migration

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

### Step 17: Commit Migration

```bash
echo ""
echo "📦 Committing migration..."

# Build commit message based on what happened
COMMIT_MSG="feat: migrate planning structure from GSD to WGSD (v2.1)

- Added WGSD directory structure (focus-groups/, active-implementations/)
- Created WGSD-CONFIG.md with project configuration
- Enhanced PROJECT.md with WGSD workflow section
- Created ${#FOCUS_GROUPS[@]} focus groups (approved)
- Created $INCLUDED_COUNT concept files ($EXCLUDED_COUNT excluded)
- Created MASTER-ROADMAP.md from existing roadmap
- Updated STATE.md with WGSD status section"

if [ "$SLACK_ENABLED" = "true" ]; then
  COMMIT_MSG="$COMMIT_MSG
- Auto-created Slack channels:$CREATED_CHANNELS"
fi

COMMIT_MSG="$COMMIT_MSG

Migration approved interactively (WGSD v2.1)
Backup: ${BACKUP_DIR:-"(no backup needed)"}"

git add .planning/
git commit -m "$COMMIT_MSG"

if [ $? -eq 0 ]; then
  echo "✅ Migration committed"
else
  echo "⚠️  Commit failed (may already be committed or nothing to commit)"
fi
```

### Step 18: Report Success

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🎉 GSD → WGSD Migration Complete (v2.1)"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Structure:"
echo "   .planning/"
echo "   ├── focus-groups/           ✅ (${#FOCUS_GROUPS[@]} groups)"
echo "   ├── active-implementations/ ✅"
echo "   ├── WGSD-CONFIG.md          ✅"
echo "   └── MASTER-ROADMAP.md       ✅"
echo ""
echo "💡 Concepts: $INCLUDED_COUNT created, $EXCLUDED_COUNT excluded"
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

- [x] GSD structure detected and analyzed
- [x] Backup created before modifications
- [x] Analysis consolidated into data arrays
- [x] Migration preview generated with all items
- [x] Explicit approval required (yes/no/edit)
- [x] Edit mode allows modifications before approval
- [x] Only approved items executed
- [x] WGSD directory structure created
- [x] WGSD-CONFIG.md generated
- [x] Content transformed and preserved
- [x] Slack channels auto-created (if token available)
- [x] Migration committed to git

---

*Workflow updated with interactive approval workflow (WGSD v2.1 - Phase 9)*
