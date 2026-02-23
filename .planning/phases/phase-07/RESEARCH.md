# Phase 7: Migration Logic Fix - Research

**Phase:** 7 - Migration Logic Fix
**Research Date:** 2026-02-23
**Researcher:** Jarvis (via gsd:plan-phase)

---

## Problem Statement

The current WGSD migration system incorrectly maps GSD constructs to WGSD constructs:

| Current (Wrong) | Correct |
|-----------------|---------|
| GSD Phase → Focus Group | GSD Phase → Concept |
| Phase 1:1 Focus Group | Focus Groups are thematic containers |
| No Concept layer | Concepts are specific deliverables |

---

## Research: Industry Patterns

### Atlassian/Jira Hierarchy Model

From Atlassian documentation ([source](https://www.atlassian.com/agile/project-management/epics-stories-themes)):

```
Theme → Initiative → Epic → Story/Task → Subtask
```

| Level | Purpose | Longevity |
|-------|---------|-----------|
| **Theme** | Strategic area/domain | Permanent |
| **Initiative** | Collection of related epics | Long-lived |
| **Epic** | Discrete deliverable | Medium-lived |
| **Story/Task** | Individual work item | Short-lived |

### Mapping to WGSD

| Jira Construct | WGSD Equivalent | Notes |
|----------------|-----------------|-------|
| Theme | Focus Group | Both are organizational domains |
| Epic | Concept | Both are discrete deliverables |
| Story/Task | Requirement | Both are individual work items |

### Mapping to GSD

| GSD Construct | WGSD Equivalent | Notes |
|---------------|-----------------|-------|
| Phase | Concept | Both are deliverables with scope |
| Requirement | Requirement | Direct mapping |
| — | Focus Group | NEW: thematic container |

---

## Analysis: Current Implementation Bugs

### Bug 1: migration-analyzer.md

**Location:** `agents/migration-analyzer.md`

**Current Behavior:**
```bash
# Maps each phase to a focus group
if echo "$phase_lower" | grep -qE "auth|security"; then
  fg="security"
```

**Problem:** Creates 1:1 mapping of phases to focus groups. If there are 6 phases, it creates 6 focus groups.

**Correct Behavior:** Phases should become Concepts. Focus Groups should be suggested based on domain clustering, not individual phase content.

### Bug 2: migrate.md

**Location:** `workflows/migrate.md`

**Current Behavior (Step 3):**
```bash
echo "📊 Step 2: Analyzing for Focus Group Suggestions"
# ...extracts phases and maps each to a focus group
```

**Problem:** Creates focus group structure from phases instead of concepts.

**Correct Behavior:**
1. Extract phases → create Concept list
2. Analyze domain patterns → suggest Focus Groups
3. Assign Concepts to appropriate Focus Groups

### Bug 3: planning-migrator.md

**Location:** `agents/planning-migrator.md`

**Current Behavior:**
- Creates concept files from requirements (partially correct)
- Missing: Transform Phase descriptions to Concept format

**Correct Behavior:**
- Transform Phase → Concept (preserve scope, goals, requirements)
- Assign each Concept to a Focus Group
- Requirements become acceptance criteria within Concepts

---

## Proposed Fix: Correct Mapping Model

### Before (Current Wrong Model)

```
GSD:
├── Phase 1: Foundation Infrastructure
├── Phase 2: Migration Wizard  
├── Phase 3: Channel Structure
└── ...6 phases total

→ Creates 6 Focus Groups (WRONG)
```

### After (Correct Model)

```
GSD:
├── Phase 1: Foundation Infrastructure
├── Phase 2: Migration Wizard
├── Phase 3: Channel Structure  
└── ...6 phases total

→ Creates 6 Concepts:
    - foundation-infrastructure
    - migration-wizard
    - channel-structure
    - ...

→ Suggests 2-4 Focus Groups based on DOMAIN:
    - core (phases 1, 3, 4)
    - migration (phases 2)
    - community (phases 5, 6)
```

---

## Domain Clustering Strategy

### Approach: Thematic Grouping

Instead of 1:1 phase→focus-group, cluster phases by domain:

| Domain Keywords | Focus Group | Typical Phases |
|-----------------|-------------|----------------|
| foundation, core, infrastructure, setup | core | Architecture phases |
| auth, security, permission, access | security | Security phases |
| api, integration, endpoint | api | API-related phases |
| channel, slack, notification | channels | Communication phases |
| workflow, process, automation | workflow | Process phases |
| community, social, feedback | community | User-facing phases |
| migration, wizard, convert | migration | Migration phases |
| canvas, dashboard, visual | visualization | UI phases |

### Minimum Focus Group Threshold

- Suggest focus groups only when 2+ concepts would belong
- Avoid creating single-concept focus groups
- Default unmatched concepts to "core" focus group

---

## Implementation Strategy

### Step 1: Fix migration-analyzer.md

**Changes:**
1. Add `suggested_concepts[]` array (from phases)
2. Change focus group logic to domain clustering
3. Output both concepts AND focus groups separately

**Output Format (New):**
```yaml
ANALYSIS_RESULT:
  suggested_concepts:
    - name: foundation-infrastructure
      source: Phase 1
      focus_group: core
    - name: migration-wizard
      source: Phase 2
      focus_group: migration
  suggested_focus_groups:
    - name: core
      concepts: [foundation-infrastructure, channel-structure]
      reason: "Infrastructure-related phases"
    - name: migration
      concepts: [migration-wizard]
      reason: "Migration-related phases"
```

### Step 2: Fix migrate.md

**Changes:**
1. Update Step 3 to show "Suggested Concepts" (not Focus Groups)
2. Update Step 5 to create concept files in `.planning/focus-groups/{fg}/concepts/`
3. Update Step 6 to reference concepts in WGSD-CONFIG.md

### Step 3: Fix planning-migrator.md

**Changes:**
1. Add Phase→Concept transformation logic
2. Update concept file format to include phase origin
3. Map Phase requirements → Concept acceptance criteria

---

## Verification Approach

### Test Case 1: 6-Phase GSD Project

**Input:** Project with 6 distinct phases
**Expected:**
- 6 Concepts created
- 2-4 Focus Groups suggested (not 6!)
- Each Concept assigned to appropriate Focus Group

### Test Case 2: Single-Domain Project

**Input:** Project with 3 API-related phases
**Expected:**
- 3 Concepts created
- 1 Focus Group suggested ("api")
- All 3 Concepts in "api" Focus Group

### Test Case 3: Mixed Domain Project

**Input:** Project with security + UI + data phases
**Expected:**
- Multiple Concepts
- 3 Focus Groups (security, frontend, data)
- Proper Concept→Focus Group assignment

---

## References

1. Atlassian Agile Hierarchy: https://www.atlassian.com/agile/project-management/epics-stories-themes
2. TitanApps Jira Themes: https://titanapps.io/blog/jira-themes-initiatives
3. Software Migration Best Practices: https://sam-solutions.com/blog/software-migration/

---

## Research Conclusion

The fix is conceptually simple but requires careful updates to three files:
1. **migration-analyzer.md** — Change output model
2. **migrate.md** — Change creation logic
3. **planning-migrator.md** — Add Phase→Concept transform

The key insight is that **Focus Groups are NOT derived from Phases** — they are thematic containers that HOLD the Concepts derived from Phases.

---

*Research complete — proceed to PLAN.md for execution details*
