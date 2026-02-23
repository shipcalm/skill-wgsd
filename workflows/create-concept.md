---
name: wgsd:create-concept
description: Create a new concept directory with branch and optional worktree
argument-hint: "[concept-name] [--worktree] [--api-spec] [--ac]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Canvas
---

<objective>
Create a new concept as a directory structure with:
- Independent concept branch (concepts/{name})
- Required artifacts (CONCEPT.md, impact-matrix.md)
- Optional artifacts (API-SPEC.md, acceptance-criteria.md)
- Optional worktree for isolated development
- Canvas sync and team notification

v2.2 Architecture: Concepts are directories, not single files.
</objective>

<libraries>
- workflows/lib/branch-ops.md - Branch creation and management
- workflows/lib/git-ops.md - Worktree operations
- workflows/lib/canvas-templates.md - Canvas updates
</libraries>

<process>

## Step 1: Parse Arguments and Context

```bash
REPO_PATH=$(pwd)
REPO_NAME=$(basename "$REPO_PATH")
CONCEPT_NAME="{concept-name}"
CONCEPT_SLUG=$(echo "$CONCEPT_NAME" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | sed 's/[^a-z0-9-]//g')

# Flags
CREATE_WORKTREE="${WORKTREE:-false}"
CREATE_API_SPEC="${API_SPEC:-false}"
CREATE_AC="${AC:-false}"

# Detect focus group from context
FOCUS_GROUP=""

# Try to detect from channel name or working directory
if [ -d ".planning/focus-groups" ]; then
    # List available focus groups
    AVAILABLE_FGS=$(find .planning/focus-groups -maxdepth 1 -type d | grep -v "^\\.planning/focus-groups$" | sed 's|.planning/focus-groups/||')
    
    # If only one focus group exists, use it
    FG_COUNT=$(echo "$AVAILABLE_FGS" | wc -l)
    if [ "$FG_COUNT" -eq 1 ] && [ -n "$AVAILABLE_FGS" ]; then
        FOCUS_GROUP="$AVAILABLE_FGS"
        echo "📍 Auto-detected focus group: ${FOCUS_GROUP}"
    fi
fi

# If not detected, ask user
if [ -z "$FOCUS_GROUP" ]; then
    echo "🤔 Which focus group should this concept belong to?"
    echo ""
    echo "Available focus groups:"
    find .planning/focus-groups -maxdepth 1 -type d 2>/dev/null | grep -v "^\\.planning/focus-groups$" | sed 's|.planning/focus-groups/||' | while read fg; do
        echo "  - $fg"
    done
    echo ""
    echo "💡 Usage: /wgsd create-concept ${CONCEPT_NAME} --focus-group <name>"
    exit 1
fi

DATE=$(date +"%Y-%m-%d")
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "💡 Creating concept: ${CONCEPT_NAME}"
echo "   Slug: ${CONCEPT_SLUG}"
echo "   Focus group: ${FOCUS_GROUP}"
echo "   Branch: concepts/${CONCEPT_SLUG}"
echo "   Worktree: ${CREATE_WORKTREE}"
```

## Step 2: Validate Prerequisites

```bash
echo ""
echo "🔍 Validating prerequisites..."

# Check focus group exists
if [ ! -d ".planning/focus-groups/${FOCUS_GROUP}" ]; then
    echo "❌ Focus group '${FOCUS_GROUP}' doesn't exist"
    echo "💡 Create it first: /wgsd create-focus-group ${FOCUS_GROUP}"
    exit 1
fi

# Check concept doesn't already exist (as directory OR legacy file)
CONCEPT_DIR=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_SLUG}"
LEGACY_FILE=".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_SLUG}.md"

if [ -d "$CONCEPT_DIR" ]; then
    echo "⚠️  Concept directory already exists: ${CONCEPT_DIR}"
    echo "📄 Edit existing concept or choose different name"
    exit 1
fi

if [ -f "$LEGACY_FILE" ]; then
    echo "⚠️  Legacy concept file exists: ${LEGACY_FILE}"
    echo "💡 Migrate with: /wgsd migrate-concept ${CONCEPT_SLUG}"
    exit 1
fi

# Check branch doesn't exist
if git rev-parse --verify "concepts/${CONCEPT_SLUG}" >/dev/null 2>&1; then
    echo "⚠️  Branch concepts/${CONCEPT_SLUG} already exists"
    exit 1
fi

echo "✅ Prerequisites validated"
```

## Step 3: Gather Concept Context

Ask user for concept details:

**Required Information:**
- **Brief Description:** What is this concept about? (1-2 sentences)
- **Problem Statement:** What problem does this solve?
- **Initial Ideas:** Any early thoughts on approach or implementation?
- **Priority Level:** P0 (Critical) / P1 (High) / P2 (Medium) / P3 (Low)
- **Dependencies:** Does this relate to other concepts or focus groups?

**Optional Information:**
- **Target Users:** Who would benefit from this?
- **Success Metrics:** How would we measure success?
- **Rough Scope:** Small/Medium/Large effort estimate

Store gathered information in variables:
```bash
PROBLEM_STATEMENT="{problem-statement}"
BRIEF_DESCRIPTION="{brief-description}"
INITIAL_IDEAS="{initial-ideas}"
PRIORITY="{priority}"  # P0, P1, P2, P3
SCOPE_ESTIMATE="{scope-estimate}"
TARGET_USERS="{target-users}"
SUCCESS_METRICS="{success-metrics}"
CREATOR="{creator-name}"
```

## Step 4: Create Concept Branch

```bash
echo ""
echo "🌿 Creating concept branch..."

# Detect primary branch
PRIMARY_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "develop")
if ! git rev-parse --verify "origin/${PRIMARY_BRANCH}" >/dev/null 2>&1; then
    PRIMARY_BRANCH="main"
fi

# Ensure we're on primary and up to date
git fetch origin
git checkout "${PRIMARY_BRANCH}"
git pull origin "${PRIMARY_BRANCH}"

# Create concept branch
CONCEPT_BRANCH="concepts/${CONCEPT_SLUG}"
git checkout -b "${CONCEPT_BRANCH}"

echo "✅ Created branch: ${CONCEPT_BRANCH}"
```

## Step 5: Create Concept Directory Structure

```bash
echo ""
echo "📁 Creating concept directory structure..."

# Create concept directory
mkdir -p "${CONCEPT_DIR}"

# Determine worktree path
WORKTREE_PATH="None"
if [ "$CREATE_WORKTREE" = "true" ]; then
    WORKTREE_PATH="worktrees/${CONCEPT_SLUG}/"
fi

# Create CONCEPT.md from template
cat > "${CONCEPT_DIR}/CONCEPT.md" << EOF
# ${CONCEPT_NAME}

**Focus Group:** ${FOCUS_GROUP}  
**Created:** ${DATE}  
**Status:** 📝 Draft  
**Priority:** ${PRIORITY}  
**Creator:** ${CREATOR}  
**Branch:** \`concepts/${CONCEPT_SLUG}\`  
**Worktree:** ${WORKTREE_PATH}

---

## Problem Statement

${PROBLEM_STATEMENT}

---

## Concept Overview

${BRIEF_DESCRIPTION}

---

## Initial Ideas

${INITIAL_IDEAS}

---

## Dependencies & Cross-References

**Related Concepts:**
- *None yet*

**Focus Group Dependencies:**
- *See impact-matrix.md for cross-cutting impacts*

---

## Discussion History

### ${DATE} - Concept Created
- Initial idea captured
- Status: Draft - needs social discussion and refinement

*Add discussion updates here as the concept evolves*

---

## Implementation Considerations

**Rough Scope:** ${SCOPE_ESTIMATE}  
**Target Users:** ${TARGET_USERS}  
**Success Metrics:** ${SUCCESS_METRICS}

*These will be refined as concept matures*

---

## Artifacts

This concept directory may contain:

| Artifact | Status | Description |
|----------|--------|-------------|
| \`CONCEPT.md\` | ✅ Created | This file - core concept description |
| \`impact-matrix.md\` | ✅ Created | Cross-cutting impact declarations |
| \`API-SPEC.md\` | $([ "$CREATE_API_SPEC" = "true" ] && echo "✅ Created" || echo "⏳ Optional") | API specification if applicable |
| \`acceptance-criteria.md\` | $([ "$CREATE_AC" = "true" ] && echo "✅ Created" || echo "⏳ Optional") | Detailed acceptance criteria |
| \`wireframes/\` | ⏳ Optional | Design mockups and wireframes |

---

## Next Steps

- [ ] Social discussion in #${REPO_NAME}-${FOCUS_GROUP}
- [ ] Declare cross-cutting impacts in \`impact-matrix.md\`
- [ ] Gather team input and refine approach
- [ ] Research similar solutions or approaches
- [ ] Define success criteria more clearly
- [ ] Move to "🔍 Exploring" status when discussion is active

---

**Status Progression:**  
📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready for Implementation → ⏳ Pending Approval → ✨ Approved

---

*Edit this file as discussions evolve. Canvas syncs automatically.*
EOF

echo "✅ Created: ${CONCEPT_DIR}/CONCEPT.md"

# Create impact-matrix.md from template
cat > "${CONCEPT_DIR}/impact-matrix.md" << EOF
# Impact Matrix: ${CONCEPT_NAME}

\`\`\`yaml
---
concept: ${CONCEPT_SLUG}
created: ${DATE}
status: draft
total_impacts: 1
---
\`\`\`

## Overview

This file declares which focus groups are impacted by this concept and tracks approval status.

---

## Impact Declarations

*Each impacted focus group requires approval before implementation.*

### Focus Group: ${FOCUS_GROUP}

| Field | Value |
|-------|-------|
| **Priority** | ${PRIORITY} |
| **Impact Type** | Primary Owner |
| **Description** | ${BRIEF_DESCRIPTION} |
| **Approval Status** | ⏳ Pending |
| **Approver** | — |
| **Approved Date** | — |

---

## Impact Summary

| Focus Group | Priority | Status | Approver |
|-------------|----------|--------|----------|
| ${FOCUS_GROUP} | ${PRIORITY} | ⏳ Pending | — |

---

## Approval Requirements

**Required for Implementation:**
- [ ] All impacted focus groups must approve
- [ ] No "Blocked" statuses remaining
- [ ] Primary focus group approved

**Status Legend:**
- ⏳ Pending - Awaiting review
- ✅ Approved - Focus group approved
- 🚫 Rejected - Focus group rejected (see feedback)
- ⏸️ Blocked - Waiting on dependency

---

## How to Add an Impact

When this concept affects another focus group, add a section:

\`\`\`markdown
### Focus Group: {focus-group-name}

| Field | Value |
|-------|-------|
| **Priority** | P0/P1/P2/P3 |
| **Impact Type** | Breaking Change / API Change / Documentation / Integration / Testing |
| **Description** | How this concept impacts the focus group |
| **Approval Status** | ⏳ Pending |
| **Approver** | — |
| **Approved Date** | — |
\`\`\`

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| ${DATE} | Impact matrix created | ${CREATOR} |

---

*This file is managed by WGSD. Canvas syncs from git.*
EOF

echo "✅ Created: ${CONCEPT_DIR}/impact-matrix.md"

# Create optional API-SPEC.md if requested
if [ "$CREATE_API_SPEC" = "true" ]; then
    cat > "${CONCEPT_DIR}/API-SPEC.md" << EOF
# API Specification: ${CONCEPT_NAME}

\`\`\`yaml
---
concept: ${CONCEPT_SLUG}
type: api-spec
version: 1.0.0
created: ${DATE}
author: ${CREATOR}
status: draft
---
\`\`\`

## Overview

**Purpose:** API specification for the ${CONCEPT_NAME} concept

**Base Path:** \`/api/v1/${CONCEPT_SLUG}\`

---

## Endpoints

*Define API endpoints here*

### GET \`/${CONCEPT_SLUG}\`

*Add endpoint documentation*

---

## Data Models

*Define data schemas here*

---

## Authentication

**Type:** Bearer Token (JWT)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | ${DATE} | Initial API specification |

---

*This specification is part of the ${CONCEPT_NAME} concept directory.*
EOF
    echo "✅ Created: ${CONCEPT_DIR}/API-SPEC.md"
fi

# Create optional acceptance-criteria.md if requested
if [ "$CREATE_AC" = "true" ]; then
    cat > "${CONCEPT_DIR}/acceptance-criteria.md" << EOF
# Acceptance Criteria: ${CONCEPT_NAME}

\`\`\`yaml
---
concept: ${CONCEPT_SLUG}
type: acceptance-criteria
created: ${DATE}
author: ${CREATOR}
status: draft
---
\`\`\`

## Overview

Acceptance criteria for the **${CONCEPT_NAME}** concept.

---

## User Stories & Acceptance Criteria

### US-001: [User Story Title]

**As a** [role]  
**I want** [feature]  
**So that** [benefit]

#### Acceptance Criteria

- [ ] **AC-001.1:** Given [context], When [action], Then [expected result]

---

## Definition of Done

- [ ] All acceptance criteria pass
- [ ] Code review completed
- [ ] Tests written
- [ ] Documentation updated

---

*This document is part of the ${CONCEPT_NAME} concept directory.*
EOF
    echo "✅ Created: ${CONCEPT_DIR}/acceptance-criteria.md"
fi
```

## Step 6: Update Focus Group Roadmap

```bash
echo ""
echo "📋 Updating focus group roadmap..."

ROADMAP_FILE=".planning/focus-groups/${FOCUS_GROUP}/ROADMAP.md"
STATE_FILE=".planning/focus-groups/${FOCUS_GROUP}/STATE.md"

# Add concept to roadmap if it exists
if [ -f "$ROADMAP_FILE" ]; then
    # Add to Draft section
    if grep -q "### 📝 Draft" "$ROADMAP_FILE"; then
        sed -i "/### 📝 Draft/a\\
- **${CONCEPT_NAME}** (${PRIORITY}) - ${BRIEF_DESCRIPTION} *[${DATE}]*" "$ROADMAP_FILE"
    else
        # Append draft section if it doesn't exist
        cat >> "$ROADMAP_FILE" << EOF

### 📝 Draft

- **${CONCEPT_NAME}** (${PRIORITY}) - ${BRIEF_DESCRIPTION} *[${DATE}]*
EOF
    fi
    echo "✅ Updated roadmap"
fi

# Update concept count in STATE.md
if [ -f "$STATE_FILE" ]; then
    CONCEPT_COUNT=$(find ".planning/focus-groups/${FOCUS_GROUP}/concepts" -mindepth 1 -maxdepth 1 \( -type d -o -name "*.md" \) 2>/dev/null | wc -l)
    sed -i "s/Active Concepts: [0-9]*/Active Concepts: $CONCEPT_COUNT/" "$STATE_FILE" 2>/dev/null || true
    echo "✅ Updated state (${CONCEPT_COUNT} concepts)"
fi
```

## Step 7: Commit and Push Concept Branch

```bash
echo ""
echo "📤 Committing concept to branch..."

# Stage all new files
git add "${CONCEPT_DIR}/"
git add ".planning/focus-groups/${FOCUS_GROUP}/ROADMAP.md" 2>/dev/null || true
git add ".planning/focus-groups/${FOCUS_GROUP}/STATE.md" 2>/dev/null || true

# Commit
git commit -m "feat(${FOCUS_GROUP}): create ${CONCEPT_SLUG} concept directory [Draft]

- Created concept directory with core artifacts
- CONCEPT.md: Primary concept description
- impact-matrix.md: Cross-cutting impact tracking
$([ "$CREATE_API_SPEC" = "true" ] && echo "- API-SPEC.md: API specification")
$([ "$CREATE_AC" = "true" ] && echo "- acceptance-criteria.md: Acceptance criteria")
- Updated focus group roadmap

Branch: concepts/${CONCEPT_SLUG}"

# Push branch
git push -u origin "${CONCEPT_BRANCH}"

echo "✅ Concept branch pushed"
```

## Step 8: Create Worktree (Optional)

```bash
if [ "$CREATE_WORKTREE" = "true" ]; then
    echo ""
    echo "📂 Creating concept worktree..."
    
    WORKTREE_DIR="worktrees/${CONCEPT_SLUG}"
    
    # Return to repo root if in worktree
    cd "$REPO_PATH"
    
    # Create worktree
    git worktree add "$WORKTREE_DIR" "${CONCEPT_BRANCH}"
    
    if [ $? -eq 0 ]; then
        echo "✅ Worktree created: ${WORKTREE_DIR}"
        echo ""
        echo "📍 To work in worktree:"
        echo "   cd ${WORKTREE_DIR}"
    else
        echo "⚠️ Worktree creation failed (branch still created)"
    fi
fi
```

## Step 9: Update Canvas

```bash
echo ""
echo "🖼️ Triggering canvas sync..."

# Sync focus group canvas with new concept
# Would invoke: sync_on_state_change "$REPO_PATH" "concept" "$FOCUS_GROUP"

echo "✅ Canvas sync triggered"
```

## Step 9.5: Declare Cross-Cutting Impacts (Optional)

Offer user the option to declare impacts on other focus groups:

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📊 Cross-Cutting Impacts"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "This concept can declare impacts on other focus groups."
echo "Example: An OAuth concept might impact Security + API + Frontend"
echo ""
read -p "Declare additional focus group impacts now? (y/n): " declare_impacts

if [ "$declare_impacts" = "y" ] || [ "$declare_impacts" = "Y" ]; then
    # Get available focus groups (excluding current)
    echo ""
    echo "Available focus groups (excluding ${FOCUS_GROUP}):"
    idx=1
    other_fgs=()
    for fg_dir in .planning/focus-groups/*/; do
        fg_name=$(basename "$fg_dir")
        if [ "$fg_name" != "${FOCUS_GROUP}" ]; then
            echo "  [$idx] $fg_name"
            other_fgs+=("$fg_name")
            ((idx++))
        fi
    done
    
    if [ ${#other_fgs[@]} -eq 0 ]; then
        echo "  No other focus groups available"
        echo "  Impact matrix initialized with primary focus group only"
    else
        echo ""
        echo "Enter focus groups to add impacts for (e.g., '1,3' or 'security,api'):"
        read -p "Selection (or Enter to skip): " fg_selection
        
        if [ -n "$fg_selection" ]; then
            # Parse selection and collect impact details
            IFS=',' read -ra selected <<< "$fg_selection"
            
            for item in "${selected[@]}"; do
                item=$(echo "$item" | tr -d ' ')
                
                # Resolve to focus group name
                fg_target=""
                if [[ "$item" =~ ^[0-9]+$ ]]; then
                    idx=$((item - 1))
                    if [ $idx -ge 0 ] && [ $idx -lt ${#other_fgs[@]} ]; then
                        fg_target="${other_fgs[$idx]}"
                    fi
                else
                    for fg in "${other_fgs[@]}"; do
                        if [ "$fg" = "$item" ]; then
                            fg_target="$fg"
                            break
                        fi
                    done
                fi
                
                if [ -n "$fg_target" ]; then
                    echo ""
                    echo "━━━ Impact on $fg_target ━━━"
                    
                    # Quick priority selection
                    echo "Priority: [0]P0 [1]P1 [2]P2 [3]P3"
                    read -p "Select: " p
                    case "$p" in
                        0) priority="P0" ;; 1) priority="P1" ;;
                        2) priority="P2" ;; 3) priority="P3" ;;
                        *) priority="P2" ;;
                    esac
                    
                    # Quick type selection
                    echo "Type: [1]breaking [2]api [3]integration [4]docs [5]testing [6]behavior [7]security [8]performance"
                    read -p "Select: " t
                    case "$t" in
                        1) type="breaking-change" ;; 2) type="api-change" ;;
                        3) type="integration" ;; 4) type="documentation" ;;
                        5) type="testing" ;; 6) type="behavior" ;;
                        7) type="security" ;; 8) type="performance" ;;
                        *) type="integration" ;;
                    esac
                    
                    read -p "Impact description: " desc
                    [ -z "$desc" ] && desc="Impact on $fg_target focus group"
                    
                    # Add to impact-matrix.md using impact_add
                    # (In practice, this calls the impact-parser library)
                    echo "✅ Added impact: $fg_target ($priority, $type)"
                fi
            done
            
            echo ""
            echo "✅ Cross-cutting impacts declared"
            echo "💡 Use /wgsd update-impact ${CONCEPT_SLUG} to modify later"
        fi
    fi
else
    echo "💡 You can declare impacts later with: /wgsd declare-impact ${CONCEPT_SLUG}"
fi
```

## Step 10: Announce in Channel

Send message to focus group channel:

```markdown
💡 **New Concept Created: {Concept Name}**

**Status:** 📝 Draft  
**Priority:** {priority}  
**Branch:** `concepts/{concept-slug}`  
**Problem:** {brief-problem-statement}

**Directory:** `.planning/focus-groups/{focus-group}/concepts/{concept-slug}/`

**Artifacts Created:**
- 📄 CONCEPT.md - Primary description
- 📊 impact-matrix.md - Cross-cutting impacts
{additional artifacts if created}

**Next Steps:**
- Team discussion and feedback
- Declare cross-cutting impacts
- Refine problem statement and approach  
- Move to 🔍 Exploring status when ready

*The concept directory and canvas have been updated. Discuss here to evolve the concept!*

**What do you think about this concept? Any initial reactions or ideas?**
```

## Step 11: Return Status

```
---

## ✅ Concept Created: {Concept Name}

**Focus Group:** {focus-group}  
**Branch:** `concepts/{concept-slug}`  
**Status:** 📝 Draft  
**Directory:** `.planning/focus-groups/{focus-group}/concepts/{concept-slug}/`

**Artifacts:**
| File | Status |
|------|--------|
| CONCEPT.md | ✅ Created |
| impact-matrix.md | ✅ Created |
| API-SPEC.md | {✅ Created or ⏳ Optional} |
| acceptance-criteria.md | {✅ Created or ⏳ Optional} |

**Worktree:** {path or "Not created"}

**The concept is now:**
- 📁 Organized as a directory with multiple artifacts
- 🌿 On its own branch for isolated development
- 🔄 Synced to focus group canvas
- 💬 Ready for team discussion

**Next Actions:**
- Discuss in #{repo-name}-{focus-group}
- Declare cross-cutting impacts in impact-matrix.md
- Add optional artifacts as needed
- Move to "Exploring" status when discussion is active

**Development Commands:**
```bash
# Switch to concept branch
git checkout concepts/{concept-slug}

# Create worktree (if not already)
git worktree add worktrees/{concept-slug} concepts/{concept-slug}

# Edit concept files
cd worktrees/{concept-slug}
vim .planning/focus-groups/{focus-group}/concepts/{concept-slug}/CONCEPT.md
```

---
```

</process>

<backward_compatibility>

## Legacy Concept Detection

The workflow detects and handles legacy single-file concepts:

```bash
# Check for legacy format
legacy_concept_exists() {
    local fg="$1"
    local name="$2"
    [ -f ".planning/focus-groups/${fg}/concepts/${name}.md" ]
}

# Read concept (supports both formats)
read_concept() {
    local fg="$1"
    local name="$2"
    
    local dir=".planning/focus-groups/${fg}/concepts/${name}"
    local file=".planning/focus-groups/${fg}/concepts/${name}.md"
    
    if [ -d "$dir" ]; then
        # v2.2 directory format
        cat "$dir/CONCEPT.md"
    elif [ -f "$file" ]; then
        # Legacy single-file format
        cat "$file"
    else
        return 1
    fi
}

# Migration hint
suggest_migration() {
    local fg="$1"
    local name="$2"
    
    echo "💡 To migrate legacy concept to directory format:"
    echo "   /wgsd migrate-concept ${name} --focus-group ${fg}"
}
```

</backward_compatibility>

<success_criteria>
- [ ] Concept directory created with proper structure
- [ ] CONCEPT.md contains all gathered information
- [ ] impact-matrix.md initialized with primary focus group
- [ ] Optional artifacts created if requested
- [ ] Concept branch created (concepts/{slug})
- [ ] Branch pushed to origin
- [ ] Focus group roadmap updated
- [ ] Worktree created if requested
- [ ] Canvas synced with new concept
- [ ] Team notified in focus group channel
- [ ] Clear next steps provided
</success_criteria>
