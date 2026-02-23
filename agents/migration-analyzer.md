---
name: wgsd:migration-analyzer
description: Intelligent agent for analyzing GSD projects and suggesting focus group structure
---

# Migration Analyzer Agent

Intelligent agent for analyzing GSD projects and suggesting WGSD focus group structure based on roadmap phases and codebase structure.

---

## Objective

Analyze a GSD project to:
1. Detect current project state and migration readiness
2. Analyze roadmap phases for focus group suggestions
3. Analyze codebase structure for additional suggestions
4. Generate a migration plan with intelligent defaults
5. Preserve work-in-progress as implementations

---

## Analysis Capabilities

### 1. Roadmap Analysis

Extract concepts from GSD roadmap phases and cluster into focus groups:

```bash
# Usage: analyze_roadmap <path_to_roadmap>
# Returns: Concepts extracted from phases + suggested focus group clusters
analyze_roadmap() {
  local roadmap_file="$1"
  
  if [ ! -f "$roadmap_file" ]; then
    echo "NO_ROADMAP"
    return 1
  fi
  
  echo "📊 Extracting Concepts from Phases: $roadmap_file"
  echo ""
  
  # Extract phase names
  local phases=$(grep -E "^##? Phase [0-9]+:|^##? [0-9]+\." "$roadmap_file" | head -10)
  
  # Extract concepts from phases and assign to focus groups
  local suggested_concepts=""
  local fg_core="" fg_security="" fg_api="" fg_frontend="" fg_data="" 
  local fg_migration="" fg_workflow="" fg_community="" fg_visualization="" fg_channels=""
  
  while IFS= read -r phase; do
    [ -z "$phase" ] && continue
    local phase_lower=$(echo "$phase" | tr '[:upper:]' '[:lower:]')
    local fg=""
    
    # Generate concept slug from phase title
    local concept_slug=$(echo "$phase" | \
      sed 's/^##*[[:space:]]*//' | \
      sed 's/^Phase[ -]*[0-9]*[: -]*//' | \
      tr '[:upper:]' '[:lower:]' | \
      tr ' ' '-' | \
      tr -cd 'a-z0-9-' | \
      sed 's/--*/-/g' | \
      sed 's/^-//' | \
      sed 's/-$//' | \
      cut -c1-30)
    
    [ -z "$concept_slug" ] && concept_slug="phase-$(echo "$phase" | grep -oE '[0-9]+' | head -1)"
    
    # Determine focus group domain
    if echo "$phase_lower" | grep -qE "foundation|infrastructure|core|setup|init"; then
      fg="core"
      fg_core="$fg_core $concept_slug"
    elif echo "$phase_lower" | grep -qE "auth|security|permission|access|identity|login|password|session"; then
      fg="security"
      fg_security="$fg_security $concept_slug"
    elif echo "$phase_lower" | grep -qE "api|integration|endpoint|webhook|rest|graphql"; then
      fg="api"
      fg_api="$fg_api $concept_slug"
    elif echo "$phase_lower" | grep -qE "ui|frontend|component|interface|ux|design|dashboard"; then
      fg="frontend"
      fg_frontend="$fg_frontend $concept_slug"
    elif echo "$phase_lower" | grep -qE "data|database|schema|model|storage"; then
      fg="data"
      fg_data="$fg_data $concept_slug"
    elif echo "$phase_lower" | grep -qE "migration|wizard|transition|convert"; then
      fg="migration"
      fg_migration="$fg_migration $concept_slug"
    elif echo "$phase_lower" | grep -qE "workflow|process|automation|pipeline"; then
      fg="workflow"
      fg_workflow="$fg_workflow $concept_slug"
    elif echo "$phase_lower" | grep -qE "community|social|feedback"; then
      fg="community"
      fg_community="$fg_community $concept_slug"
    elif echo "$phase_lower" | grep -qE "canvas|visualization|display"; then
      fg="visualization"
      fg_visualization="$fg_visualization $concept_slug"
    elif echo "$phase_lower" | grep -qE "channel|slack|notification|messaging"; then
      fg="channels"
      fg_channels="$fg_channels $concept_slug"
    else
      fg="core"  # Default to core
      fg_core="$fg_core $concept_slug"
    fi
    
    suggested_concepts="$suggested_concepts $concept_slug"
    echo "   💡 $concept_slug → $fg ← $phase"
  done <<< "$phases"
  
  echo ""
  
  # Build focus groups from concept clusters
  local suggested_fgs=""
  local fg_details=""
  
  [ -n "$(echo $fg_core | xargs)" ] && { 
    suggested_fgs="$suggested_fgs core"
    fg_details="$fg_details\n   📂 core: $(echo $fg_core | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_security | xargs)" ] && {
    suggested_fgs="$suggested_fgs security"
    fg_details="$fg_details\n   📂 security: $(echo $fg_security | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_api | xargs)" ] && {
    suggested_fgs="$suggested_fgs api"
    fg_details="$fg_details\n   📂 api: $(echo $fg_api | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_frontend | xargs)" ] && {
    suggested_fgs="$suggested_fgs frontend"
    fg_details="$fg_details\n   📂 frontend: $(echo $fg_frontend | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_data | xargs)" ] && {
    suggested_fgs="$suggested_fgs data"
    fg_details="$fg_details\n   📂 data: $(echo $fg_data | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_migration | xargs)" ] && {
    suggested_fgs="$suggested_fgs migration"
    fg_details="$fg_details\n   📂 migration: $(echo $fg_migration | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_workflow | xargs)" ] && {
    suggested_fgs="$suggested_fgs workflow"
    fg_details="$fg_details\n   📂 workflow: $(echo $fg_workflow | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_community | xargs)" ] && {
    suggested_fgs="$suggested_fgs community"
    fg_details="$fg_details\n   📂 community: $(echo $fg_community | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_visualization | xargs)" ] && {
    suggested_fgs="$suggested_fgs visualization"
    fg_details="$fg_details\n   📂 visualization: $(echo $fg_visualization | xargs | tr ' ' ', ')"
  }
  [ -n "$(echo $fg_channels | xargs)" ] && {
    suggested_fgs="$suggested_fgs channels"
    fg_details="$fg_details\n   📂 channels: $(echo $fg_channels | xargs | tr ' ' ', ')"
  }
  
  echo "📂 Focus Group Clusters:"
  echo -e "$fg_details"
  echo ""
  echo "SUGGESTED_CONCEPTS:$(echo $suggested_concepts)"
  echo "SUGGESTED_FOCUS_GROUPS:$(echo $suggested_fgs)"
  return 0
}
```

---

### 2. Codebase Analysis

Analyze directory structure for additional focus group suggestions:

```bash
# Usage: analyze_codebase <repo_path>
# Returns: Additional focus group suggestions from code structure
analyze_codebase() {
  local repo_path="${1:-.}"
  
  echo "📁 Analyzing Codebase Structure: $repo_path"
  echo ""
  
  local suggested_fgs=""
  
  # Check for common directory patterns
  if [ -d "$repo_path/api" ] || [ -d "$repo_path/src/api" ] || [ -d "$repo_path/routes" ]; then
    if ! echo "$suggested_fgs" | grep -q "api"; then
      suggested_fgs="$suggested_fgs api"
      echo "   📂 api ← Found api/ or routes/ directory"
    fi
  fi
  
  if [ -d "$repo_path/components" ] || [ -d "$repo_path/src/components" ] || [ -d "$repo_path/ui" ]; then
    if ! echo "$suggested_fgs" | grep -q "frontend"; then
      suggested_fgs="$suggested_fgs frontend"
      echo "   📂 frontend ← Found components/ or ui/ directory"
    fi
  fi
  
  if [ -d "$repo_path/lib" ] || [ -d "$repo_path/src/lib" ] || [ -d "$repo_path/core" ]; then
    if ! echo "$suggested_fgs" | grep -q "core"; then
      suggested_fgs="$suggested_fgs core"
      echo "   📂 core ← Found lib/ or core/ directory"
    fi
  fi
  
  if [ -d "$repo_path/auth" ] || [ -d "$repo_path/src/auth" ] || [ -d "$repo_path/security" ]; then
    if ! echo "$suggested_fgs" | grep -q "security"; then
      suggested_fgs="$suggested_fgs security"
      echo "   📂 security ← Found auth/ or security/ directory"
    fi
  fi
  
  if [ -d "$repo_path/db" ] || [ -d "$repo_path/models" ] || [ -d "$repo_path/migrations" ]; then
    if ! echo "$suggested_fgs" | grep -q "data"; then
      suggested_fgs="$suggested_fgs data"
      echo "   📂 data ← Found db/, models/, or migrations/ directory"
    fi
  fi
  
  if [ -d "$repo_path/tests" ] || [ -d "$repo_path/__tests__" ] || [ -d "$repo_path/spec" ]; then
    if ! echo "$suggested_fgs" | grep -q "testing"; then
      suggested_fgs="$suggested_fgs testing"
      echo "   📂 testing ← Found tests/ directory"
    fi
  fi
  
  if [ -d "$repo_path/docs" ] || [ -d "$repo_path/wiki" ] || [ -d "$repo_path/documentation" ]; then
    if ! echo "$suggested_fgs" | grep -q "docs"; then
      suggested_fgs="$suggested_fgs docs"
      echo "   📂 docs ← Found docs/ directory"
    fi
  fi
  
  if [ -d "$repo_path/scripts" ] || [ -d "$repo_path/tools" ] || [ -d "$repo_path/infra" ]; then
    if ! echo "$suggested_fgs" | grep -q "infrastructure"; then
      suggested_fgs="$suggested_fgs infrastructure"
      echo "   📂 infrastructure ← Found scripts/, tools/, or infra/ directory"
    fi
  fi
  
  if [ -d "$repo_path/workflows" ] || [ -d "$repo_path/agents" ] || [ -d "$repo_path/skills" ]; then
    if ! echo "$suggested_fgs" | grep -q "ai"; then
      suggested_fgs="$suggested_fgs ai"
      echo "   📂 ai ← Found workflows/ or agents/ directory"
    fi
  fi
  
  if [ -d "$repo_path/plugins" ] || [ -d "$repo_path/extensions" ]; then
    if ! echo "$suggested_fgs" | grep -q "extensibility"; then
      suggested_fgs="$suggested_fgs extensibility"
      echo "   📂 extensibility ← Found plugins/ or extensions/ directory"
    fi
  fi
  
  echo ""
  echo "CODEBASE_SUGGESTIONS:$(echo $suggested_fgs)"
  return 0
}
```

---

### 3. Requirements Analysis

Parse REQUIREMENTS.md to extract domain groupings:

```bash
# Usage: analyze_requirements <path_to_requirements>
# Returns: Focus groups derived from requirement categories
analyze_requirements() {
  local req_file="$1"
  
  if [ ! -f "$req_file" ]; then
    echo "NO_REQUIREMENTS"
    return 1
  fi
  
  echo "📋 Analyzing Requirements: $req_file"
  echo ""
  
  local suggested_fgs=""
  
  # Look for section headers that indicate domains
  local sections=$(grep -E "^##? [A-Z]" "$req_file" | head -15)
  
  while IFS= read -r section; do
    local section_lower=$(echo "$section" | tr '[:upper:]' '[:lower:]')
    local fg=""
    
    # Map section to focus group
    case "$section_lower" in
      *migration*|*migrate*) fg="migration" ;;
      *channel*|*slack*|*communication*) fg="channels" ;;
      *canvas*|*dashboard*|*visual*) fg="visualization" ;;
      *workflow*|*process*|*automation*) fg="workflow" ;;
      *community*|*social*|*feedback*) fg="community" ;;
      *integrat*|*api*|*connect*) fg="api" ;;
      *security*|*auth*|*permission*) fg="security" ;;
    esac
    
    if [ -n "$fg" ] && ! echo "$suggested_fgs" | grep -q "$fg"; then
      suggested_fgs="$suggested_fgs $fg"
      echo "   📂 $fg ← Section: $section"
    fi
  done <<< "$sections"
  
  echo ""
  echo "REQUIREMENTS_SUGGESTIONS:$(echo $suggested_fgs)"
  return 0
}
```

---

### 4. Combined Analysis

Merge all suggestions with concept extraction and focus group clustering:

```bash
# Usage: full_analysis <repo_path>
# Returns: Complete migration analysis with concepts and focus group recommendations
full_analysis() {
  local repo_path="${1:-.}"
  local planning_dir="$repo_path/.planning"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "🔍 WGSD Migration Analysis"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  echo "📁 Project: $(basename $(realpath $repo_path))"
  echo "📅 Date: $(date -I)"
  echo ""
  
  # Collect concepts and focus groups
  local all_concepts=""
  local all_fgs=""
  
  # Roadmap analysis - extract concepts from phases
  if [ -f "$planning_dir/ROADMAP.md" ]; then
    echo "─────────────────────────────────────────────────────────"
    roadmap_result=$(analyze_roadmap "$planning_dir/ROADMAP.md")
    echo "$roadmap_result"
    roadmap_concepts=$(echo "$roadmap_result" | grep "SUGGESTED_CONCEPTS:" | cut -d: -f2)
    roadmap_fgs=$(echo "$roadmap_result" | grep "SUGGESTED_FOCUS_GROUPS:" | cut -d: -f2)
    all_concepts="$all_concepts $roadmap_concepts"
    all_fgs="$all_fgs $roadmap_fgs"
    echo ""
  fi
  
  # Requirements analysis
  if [ -f "$planning_dir/REQUIREMENTS.md" ]; then
    echo "─────────────────────────────────────────────────────────"
    req_result=$(analyze_requirements "$planning_dir/REQUIREMENTS.md")
    echo "$req_result"
    req_fgs=$(echo "$req_result" | grep "REQUIREMENTS_SUGGESTIONS:" | cut -d: -f2)
    all_fgs="$all_fgs $req_fgs"
    echo ""
  fi
  
  # Codebase analysis
  echo "─────────────────────────────────────────────────────────"
  code_result=$(analyze_codebase "$repo_path")
  echo "$code_result"
  code_fgs=$(echo "$code_result" | grep "CODEBASE_SUGGESTIONS:" | cut -d: -f2)
  all_fgs="$all_fgs $code_fgs"
  echo ""
  
  # Deduplicate concepts and focus groups
  local unique_concepts=$(echo $all_concepts | tr ' ' '\n' | sort -u | tr '\n' ' ')
  local concept_count=$(echo $unique_concepts | wc -w | tr -d ' ')
  local unique_fgs=$(echo $all_fgs | tr ' ' '\n' | sort -u | tr '\n' ' ')
  local fg_count=$(echo $unique_fgs | wc -w | tr -d ' ')
  
  echo "─────────────────────────────────────────────────────────"
  echo "📊 Analysis Summary"
  echo ""
  
  if [ "$concept_count" -gt 0 ]; then
    echo "📝 Suggested Concepts ($concept_count) - from GSD Phases:"
    for concept in $unique_concepts; do
      [ -n "$concept" ] && echo "   💡 $concept"
    done
    echo ""
  fi
  
  echo "📂 Recommended Focus Groups ($fg_count) - thematic containers:"
  for fg in $unique_fgs; do
    [ -n "$fg" ] && echo "   ✅ $fg"
  done
  echo ""
  
  # Provide migration command
  echo "─────────────────────────────────────────────────────────"
  echo "🚀 Ready to Migrate"
  echo ""
  echo "Run: wgsd migrate $repo_path"
  echo ""
  echo "The migration wizard will:"
  echo "   1. Create WGSD workspace structure"
  echo "   2. Create $concept_count concepts from GSD phases"
  echo "   3. Organize concepts into $fg_count focus groups"
  echo "   4. Preserve any work-in-progress"
  echo "   5. Generate team announcement"
  echo ""
  echo "═══════════════════════════════════════════════════════════"
  
  # Export for use by migration workflow
  echo ""
  echo "ANALYSIS_RESULT:"
  echo "  suggested_concepts: [$unique_concepts]"
  echo "  concept_count: $concept_count"
  echo "  suggested_focus_groups: [$unique_fgs]"
  echo "  focus_group_count: $fg_count"
  echo "  has_roadmap: $([ -f "$planning_dir/ROADMAP.md" ] && echo true || echo false)"
  echo "  has_requirements: $([ -f "$planning_dir/REQUIREMENTS.md" ] && echo true || echo false)"
  echo "  migration_ready: true"
  
  return 0
}
```

---

### 5. Implementation Name Generation

Generate implementation names from WIP context:

```bash
# Usage: generate_impl_name <stub> <phase_info> <branch_name>
# Returns: Validated implementation name
generate_impl_name() {
  local stub="$1"
  local phase_info="$2"
  local branch="$3"
  
  local base_name=""
  
  # Try to extract from phase info
  if [ -n "$phase_info" ]; then
    # "Phase 2: Security Implementation" → "security"
    base_name=$(echo "$phase_info" | \
      sed 's/^[Pp]hase[ -]*[0-9]*[: -]*//' | \
      tr '[:upper:]' '[:lower:]' | \
      tr ' ' '-' | \
      tr -cd 'a-z0-9-' | \
      sed 's/--*/-/g' | \
      sed 's/^-//' | \
      sed 's/-$//' | \
      cut -c1-20)
  fi
  
  # Fall back to branch name
  if [ -z "$base_name" ] && [ -n "$branch" ]; then
    # "phase-02-auth" → "auth"
    base_name=$(echo "$branch" | \
      sed 's/^phase-[0-9]*-//' | \
      sed 's/^phases\///' | \
      tr '[:upper:]' '[:lower:]' | \
      tr -cd 'a-z0-9-' | \
      cut -c1-20)
  fi
  
  # Default if nothing found
  if [ -z "$base_name" ]; then
    base_name="wip-$(date +%Y%m%d)"
  fi
  
  # Generate full implementation name
  local impl_name="${stub}-impl-${base_name}"
  
  echo "$impl_name"
  return 0
}
```

---

## Interactive Mode

When invoked interactively, the agent guides the user:

```markdown
📊 **Migration Analysis Complete**

I've analyzed your GSD project and have some recommendations.

**Detected State:**
- Current Phase: Phase 2 - Security Implementation
- Status: In Progress (75% complete)
- Uncommitted Changes: 3 files

**Suggested Concepts (from GSD Phases):**
1. 💡 **foundation-setup** ← Phase 1
2. 💡 **security-implementation** ← Phase 2 (current)
3. 💡 **api-development** ← Phase 3

**Recommended Focus Groups (thematic containers):**
1. 📂 **core** (1 concept: foundation-setup)
2. 📂 **security** (1 concept: security-implementation)
3. 📂 **api** (1 concept: api-development)

**Note:** Each GSD Phase becomes a Concept within a Focus Group.
Focus Groups are organizational containers (2-4 typical), not 1:1 with phases.

**Work-in-Progress:**
Your current phase has uncommitted work. I recommend:
- Converting it to: `mvn-impl-security-auth`
- This preserves your progress as an active implementation

**Next Steps:**
1. Review the concept and focus group suggestions above
2. Run `wgsd migrate` to start the wizard
3. The wizard will create concept files in appropriate focus groups

Would you like to proceed with the migration?
```

---

## Usage Examples

```bash
# Full analysis of a project
full_analysis /path/to/gsd-project

# Just roadmap analysis
analyze_roadmap /path/to/.planning/ROADMAP.md

# Just codebase analysis
analyze_codebase /path/to/project

# Generate implementation name
generate_impl_name "mvn" "Phase 2: Security" "phase-02-auth"
```

---

*Agent enhanced for WGSD Phase 2 - MIGRATE-02, MIGRATE-03, MIGRATE-05*
