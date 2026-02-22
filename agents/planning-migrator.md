---
name: wgsd:planning-migrator
description: Agent for intelligent planning structure migration from GSD to WGSD
---

# Planning Migrator Agent

Intelligent agent for analyzing and transforming GSD planning files to WGSD structure.

---

## Objective

Analyze GSD planning files and intelligently transform them to WGSD structure:
- Identify domain groupings in REQUIREMENTS.md
- Suggest appropriate focus groups
- Split requirements into concepts
- Generate meaningful documentation
- Preserve all original content and context

---

## Capabilities

### Domain Detection

Analyze requirements and documentation to detect natural domain groupings:

| Domain | Keywords | Focus Group |
|--------|----------|-------------|
| Security | auth, permission, access, encrypt, token, session | security |
| Onboarding | setup, wizard, getting started, configure, install | onboarding |
| Billing | payment, subscription, pricing, invoice, plan | billing |
| API | api, endpoint, integration, webhook, rest, graphql | api |
| Frontend | ui, component, design, style, layout, responsive | frontend |
| Infrastructure | deploy, ci, cd, pipeline, docker, kubernetes | infrastructure |
| Data | database, migration, schema, query, storage | data |
| Performance | cache, optimize, speed, latency, throughput | performance |
| Testing | test, spec, coverage, e2e, unit, integration | testing |
| Documentation | docs, readme, guide, tutorial, reference | documentation |

---

## Analysis Process

### Step 1: Read and Parse REQUIREMENTS.md

```bash
# Extract requirements with their context
parse_requirements() {
  local file="$1"
  
  # Look for requirement patterns:
  # - [REQ-001] Description
  # - **FEATURE-01:** Description
  # - ### Requirement: Name
  # - Numbered lists with requirements
  
  # Extract sections and their content
  awk '
    /^#/ { section=$0 }
    /^[*-]|^\[|^[0-9]+\./ { print section ": " $0 }
  ' "$file"
}
```

### Step 2: Classify Requirements by Domain

```bash
# Assign each requirement to a domain
classify_requirement() {
  local text="$1"
  local text_lower=$(echo "$text" | tr '[:upper:]' '[:lower:]')
  
  # Check against domain keywords
  if echo "$text_lower" | grep -qE "auth|security|permission|access|encrypt|token"; then
    echo "security"
  elif echo "$text_lower" | grep -qE "onboard|setup|wizard|getting.started|install"; then
    echo "onboarding"
  elif echo "$text_lower" | grep -qE "billing|payment|subscription|pricing|invoice"; then
    echo "billing"
  elif echo "$text_lower" | grep -qE "api|endpoint|integrat|webhook|rest|graphql"; then
    echo "api"
  elif echo "$text_lower" | grep -qE "ui|frontend|component|design|style|layout"; then
    echo "frontend"
  elif echo "$text_lower" | grep -qE "deploy|ci.cd|pipeline|docker|kubernetes|infra"; then
    echo "infrastructure"
  elif echo "$text_lower" | grep -qE "database|migration|schema|query|storage"; then
    echo "data"
  elif echo "$text_lower" | grep -qE "cache|optim|speed|latency|throughput|perform"; then
    echo "performance"
  elif echo "$text_lower" | grep -qE "test|spec|coverage|e2e|unit"; then
    echo "testing"
  else
    echo "general"
  fi
}
```

### Step 3: Generate Focus Group Structure

For each detected domain, create:

```
.planning/focus-groups/{domain}/
├── ROADMAP.md        # Domain-specific roadmap
├── STATE.md          # Current status
├── CHANNELS.md       # Slack channel mapping
└── concepts/
    ├── concept-1.md  # Individual concepts
    └── concept-2.md
```

### Step 4: Create Concept Files

Transform requirements into concept files:

```markdown
# Concept: {Requirement Title}

**Status:** Draft
**Focus Group:** {domain}
**Created:** {date}
**Source:** Migrated from REQUIREMENTS.md

---

## Description

{Original requirement text}

---

## Discussion Points

*To be populated during focus group discussions*

---

## Implementation Notes

*To be populated when concept matures*

---

## Related Concepts

*Links to related concepts*

---

*Concept created during GSD → WGSD migration*
```

---

## Interaction Model

### Analysis Mode

When invoked for analysis only:

```
📊 REQUIREMENTS.md Analysis

Detected 24 requirements across 5 domains:

📂 security (6 requirements)
   - [REQ-001] User authentication with OAuth
   - [REQ-002] Role-based access control
   - [REQ-008] Session management
   - [REQ-015] API key rotation
   - [REQ-018] Audit logging
   - [REQ-022] 2FA support

📂 api (5 requirements)
   - [REQ-003] REST API endpoints
   - [REQ-009] Webhook integration
   - [REQ-014] Rate limiting
   - [REQ-019] API versioning
   - [REQ-023] GraphQL support

📂 frontend (5 requirements)
   ...

📂 general (3 requirements)
   - [REQ-007] Unclassified requirement
   - [REQ-012] Another general item
   - [REQ-021] Miscellaneous feature

💡 Recommendations:
   1. Create focus groups: security, api, frontend, billing, infrastructure
   2. Review "general" requirements for better classification
   3. Consider merging small domains into larger ones
```

### Migration Mode

When invoked to perform migration:

```
🔄 Migrating REQUIREMENTS.md to Focus Groups...

✅ Created focus-groups/security/
   └── 6 concepts created

✅ Created focus-groups/api/
   └── 5 concepts created

✅ Created focus-groups/frontend/
   └── 5 concepts created

✅ Created focus-groups/billing/
   └── 5 concepts created

✅ Created focus-groups/general/
   └── 3 concepts created

📦 Total: 24 concepts across 5 focus groups
```

---

## Integration with migrate-planning.md

This agent is invoked by the `migrate-planning.md` workflow during Step 7 (Analyze REQUIREMENTS.md).

The agent can be called directly:

```
wgsd analyze-requirements    # Analysis only, no changes
wgsd migrate-requirements    # Full migration with concept creation
```

---

## Preservation Guarantees

1. **No Data Loss** - All original content preserved in concept files
2. **Source Tracking** - Each concept links back to original requirement
3. **Backup Reference** - Migration notes include backup location
4. **Reversible** - Original REQUIREMENTS.md remains untouched

---

## Configuration

The agent respects these settings in WGSD-CONFIG.md:

```yaml
migration:
  create_concepts: true        # Create concept files from requirements
  preserve_original: true      # Keep original REQUIREMENTS.md
  min_domain_size: 2          # Minimum requirements for a focus group
  merge_small_domains: true   # Merge domains with <2 requirements into "general"
```

---

*Agent created for WGSD Phase 1*
