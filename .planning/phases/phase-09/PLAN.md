# Phase 9: Approval Workflow - Execution Plan

**Phase:** 9 - Approval Workflow (FINAL)
**Requirements:** 4 (APPROVE-01 through APPROVE-04)
**Estimated Duration:** ~2 hours
**Plan Date:** 2026-02-23

---

## Overview

Add interactive preview and approval step to migration workflow so users can review and modify suggestions before execution.

| Requirement | Description | Priority |
|-------------|-------------|----------|
| APPROVE-01 | Generate migration preview | Critical |
| APPROVE-02 | Display preview to user | Critical |
| APPROVE-03 | Require explicit approval | Critical |
| APPROVE-04 | Allow modification before approval | High |

---

## Architecture

### New Step Flow

```
┌─────────────────────────────────────────────────────────────┐
│  ANALYSIS PHASE (Steps 1-4)                                 │
├─────────────────────────────────────────────────────────────┤
│  1. Detect GSD Structure                                    │
│  2. Create Backup                                           │
│  3. Validate Slack Connectivity                             │
│  4. Comprehensive Analysis (consolidate current Steps 8)    │
│     ├─ Extract phases from ROADMAP.md                       │
│     ├─ Analyze REQUIREMENTS.md for domains                  │
│     ├─ Map phases → concepts                                │
│     ├─ Suggest focus groups (clustering)                    │
│     └─ Store in data arrays                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  PREVIEW PHASE (NEW - Steps 5-7)                            │
├─────────────────────────────────────────────────────────────┤
│  5. Generate Preview                                        │
│     └─ Build formatted preview from data arrays             │
│  6. Display Preview to User                                 │
│     └─ Output comprehensive migration plan                  │
│  7. Approval Loop                                           │
│     ├─ Prompt: "yes" / "no" / "edit"                       │
│     ├─ If "no" → abort cleanly                             │
│     ├─ If "edit" → modification submenu → re-preview       │
│     └─ If "yes" → proceed to execution                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  EXECUTION PHASE (Steps 8-15, only if approved)             │
├─────────────────────────────────────────────────────────────┤
│  8. Create WGSD Directory Structure                         │
│  9. Transform PROJECT.md                                    │
│ 10. Create MASTER-ROADMAP.md                                │
│ 11. Create Focus Groups (from approved list)                │
│ 12. Create Concept Files (from approved list)               │
│ 13. Generate WGSD-CONFIG.md                                 │
│ 14. Create Slack Channels (from approved list)              │
│ 15. Update STATE.md                                         │
│ 16. Validate & Commit                                       │
│ 17. Report Success                                          │
└─────────────────────────────────────────────────────────────┘
```

### Data Structures

```bash
# Focus Groups (name:source)
FOCUS_GROUPS=("security:analysis" "api:analysis" "infrastructure:analysis")

# Concepts (slug:focus_group:source_phase:status)
CONCEPTS=("auth-v2:security:Phase 3:include" "webhook:api:Phase 4:include")

# Channels (name:type:visibility)
CHANNELS=("wgsd-dev:core:private" "wgsd-community:core:public" "wgsd-fg-security:focus-group:private")
```

---

## Execution Sequence

### Wave 1: Consolidate Analysis (Step 4)
**Duration:** ~30 min
**File:** `workflows/migrate-planning.md`

Consolidate all analysis into a single comprehensive step that collects all data before any execution.

#### Task 1.1: Add Analysis Data Structures

Insert after Step 3 (Slack validation):

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

# Count variables
FG_COUNT=0
CONCEPT_COUNT=0
```

#### Task 1.2: Extract Phases and Generate Concepts

```bash
# ─── Extract Phases from ROADMAP.md ───
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
  echo "   📝 Found $CONCEPT_COUNT concepts"
  echo ""
fi
```

#### Task 1.3: Analyze Domains for Focus Groups

```bash
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

# Add "core" if no focus groups or always as fallback
if [ ${#FOCUS_GROUPS[@]} -eq 0 ]; then
  FOCUS_GROUPS+=("core:default")
  echo "   🎯 core ← (default focus group)"
fi

FG_COUNT=${#FOCUS_GROUPS[@]}
echo "   📂 Suggested $FG_COUNT focus groups"
echo ""
```

#### Task 1.4: Assign Concepts to Focus Groups

```bash
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
    assigned_fg="security"
  elif echo "$concept_lower" | grep -qE "api|webhook|endpoint|integrat"; then
    assigned_fg="api"
  elif echo "$concept_lower" | grep -qE "infra|deploy|ci|cd|pipeline|docker"; then
    assigned_fg="infrastructure"
  elif echo "$concept_lower" | grep -qE "ui|frontend|component|design"; then
    assigned_fg="frontend"
  elif echo "$concept_lower" | grep -qE "onboard|setup|wizard"; then
    assigned_fg="onboarding"
  elif echo "$concept_lower" | grep -qE "billing|payment|subscription"; then
    assigned_fg="billing"
  fi
  
  # Verify assigned_fg exists, else use default
  if ! printf '%s\n' "${FG_NAMES[@]}" | grep -qx "$assigned_fg"; then
    assigned_fg="$DEFAULT_FG"
  fi
  
  UPDATED_CONCEPTS+=("$slug:$assigned_fg:$source:$status")
  echo "   💡 $slug → $assigned_fg"
done
CONCEPTS=("${UPDATED_CONCEPTS[@]}")

echo ""
```

#### Task 1.5: Build Channels List

```bash
# ─── Build Channels List ───
echo "📱 Planning Slack channels..."

CHANNELS=()

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

---

### Wave 2: Generate Preview (Step 5)
**Duration:** ~20 min
**File:** `workflows/migrate-planning.md`

#### Task 2.1: Create Preview Generation Function

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
      printf "  [%d] %-20s (EXCLUDED)\n" "$idx" "$slug"
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
      [ "$ch_type" = "core" ] || [ "$ch_type" = "community" ] && \
        printf "    • #%-25s (%s)\n" "$ch_name" "$ch_vis"
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

---

### Wave 3: Display Preview & Approval Loop (Steps 6-7)
**Duration:** ~40 min
**File:** `workflows/migrate-planning.md`

#### Task 3.1: Display Preview (Step 6)

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "📋 Step 5: Migration Preview"
echo "═══════════════════════════════════════════════════"

generate_preview
```

#### Task 3.2: Approval Loop (Step 7)

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
    yes|YES|Yes)
      echo ""
      echo "✅ Migration approved"
      APPROVED="true"
      ;;
      
    no|NO|No)
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
      
    edit|EDIT|Edit)
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
            
          done|DONE|Done)
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
            
          *)
            echo "   ❌ Unknown command. Type 'done' to finish editing."
            ;;
        esac
      done
      ;;
      
    *)
      echo "   Please enter 'yes', 'no', or 'edit'"
      ;;
  esac
done
```

---

### Wave 4: Update Execution to Use Approved Data
**Duration:** ~30 min
**File:** `workflows/migrate-planning.md`

#### Task 4.1: Create Focus Groups from Approved List

Update Step 11 (current Focus Group creation) to iterate over FOCUS_GROUPS array:

```bash
# Replace current focus group creation with:
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

#### Task 4.2: Create Concept Files from Approved List

Add new step for concept file creation:

```bash
echo ""
echo "💡 Creating approved concept files..."

for concept_entry in "${CONCEPTS[@]}"; do
  IFS=':' read -r slug fg source status <<< "$concept_entry"
  
  # Skip excluded concepts
  [ "$status" != "include" ] && continue
  
  concept_file=".planning/focus-groups/$fg/concepts/$slug.md"
  
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
done
```

#### Task 4.3: Create Channels from Approved List

Update Slack channel creation to use CHANNELS array:

```bash
if [ "$SLACK_ENABLED" = "true" ]; then
  echo ""
  echo "📱 Creating approved Slack channels..."
  
  for ch_entry in "${CHANNELS[@]}"; do
    IFS=':' read -r ch_name ch_type ch_vis <<< "$ch_entry"
    
    is_private="true"
    [ "$ch_vis" = "public" ] && is_private="false"
    
    echo "   Creating: #$ch_name ($ch_vis)"
    
    RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"name\":\"$ch_name\",\"is_private\":$is_private}")
    
    if [ "$(echo "$RESPONSE" | jq -r '.ok')" = "true" ]; then
      CH_ID=$(echo "$RESPONSE" | jq -r '.channel.id')
      echo "      ✅ Created: #$ch_name ($CH_ID)"
      CREATED_CHANNELS="$CREATED_CHANNELS $ch_name"
    elif [ "$(echo "$RESPONSE" | jq -r '.error')" = "name_taken" ]; then
      echo "      ℹ️  Already exists: #$ch_name"
      EXISTING_CHANNELS="$EXISTING_CHANNELS $ch_name"
    else
      echo "      ❌ Failed: $(echo "$RESPONSE" | jq -r '.error')"
      FAILED_CHANNELS="$FAILED_CHANNELS $ch_name"
    fi
    
    sleep 1  # Rate limiting
  done
fi
```

---

## File Changes Summary

| File | Type | Changes |
|------|------|---------|
| `workflows/migrate-planning.md` | Modify | Complete restructure with approval workflow |

**Single file modification** — all changes integrate into the existing migration workflow.

---

## Testing Strategy

### Test Case 1: Happy Path - Approve Immediately
1. Run migration on GSD project
2. Review preview
3. Type `yes` to approve
4. Verify all focus groups, concepts, channels created

### Test Case 2: Cancel Migration
1. Run migration
2. Review preview
3. Type `no` to cancel
4. Verify project unchanged
5. Verify backup cleaned up

### Test Case 3: Edit Before Approval
1. Run migration
2. Type `edit`
3. Rename a focus group
4. Move a concept
5. Exclude a concept
6. Type `done`
7. Verify updated preview shows changes
8. Approve and verify only approved items created

### Test Case 4: Add Custom Focus Group
1. Run migration
2. Type `edit`
3. `add-fg custom-group`
4. `move-concept 1 custom-group`
5. `done` → verify preview
6. Approve and verify custom focus group created

### Test Case 5: Remove Focus Group
1. Run migration with multiple suggested FGs
2. Type `edit`
3. `remove-fg 2`
4. `done` → verify concepts reassigned
5. Approve and verify

---

## Risk Mitigation

| Risk | Mitigation | Severity |
|------|------------|----------|
| Bash arrays complex | Use simple string format with : delimiter | Medium |
| Edit loop infinite | Clear exit with 'done' command | Low |
| Preview too long | Use compact format, scroll | Low |
| User typos in edit | Validate before applying, show confirmation | Low |

---

## Dependencies

- **Phase 7 Complete:** ✅ Concepts extracted correctly
- **Phase 8 Complete:** ✅ Slack channel automation in place
- **No External Dependencies:** All bash-based

---

## Success Criteria

- [ ] Preview shows all focus groups with numbers (APPROVE-01)
- [ ] Preview shows all concepts with mappings (APPROVE-01)
- [ ] Preview shows all planned channels (APPROVE-01)
- [ ] Preview formatted clearly with sections (APPROVE-02)
- [ ] User must type 'yes' to proceed (APPROVE-03)
- [ ] 'no' cancels cleanly (APPROVE-03)
- [ ] 'edit' allows modifications (APPROVE-04)
- [ ] Can rename focus groups (APPROVE-04)
- [ ] Can move concepts between FGs (APPROVE-04)
- [ ] Can exclude/include concepts (APPROVE-04)
- [ ] Can add custom focus groups (APPROVE-04)
- [ ] Updated preview reflects changes (APPROVE-04)
- [ ] Only approved items created (APPROVE-03)

---

## Execution Command

```
/gsd build-phase 9
```

Or manual execution:
1. Add consolidated analysis (Wave 1)
2. Add preview generation (Wave 2)
3. Add approval loop with edit mode (Wave 3)
4. Update execution to use approved data (Wave 4)
5. Test all scenarios

---

*Execution plan complete — Phase 9 ready for implementation*
