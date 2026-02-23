# Phase 7: Migration Logic Fix - Execution Plan

**Phase:** 7 - Migration Logic Fix
**Requirements:** 3 (MIG-FIX-01, MIG-FIX-02, MIG-FIX-03)
**Estimated Duration:** ~1 hour
**Plan Date:** 2026-02-23

---

## Overview

This phase fixes the incorrect Phase → Focus Group mapping throughout the migration system. The correct mapping is:

| GSD Construct | WGSD Equivalent |
|---------------|-----------------|
| Phase | Concept |
| Requirements | Requirements (preserved) |
| — | Focus Group (NEW: thematic container) |

---

## Execution Sequence

### Requirement 1: MIG-FIX-02 (First)

**File:** `agents/migration-analyzer.md`

Start here because analyzer output drives the rest of the system.

#### Changes Required

1. **Add `suggested_concepts[]` output**
   - Extract phase names → convert to concept names
   - Include source phase reference
   - Include initial focus group assignment

2. **Change `analyze_roadmap()` function**
   - Current: Maps each phase → one focus group
   - New: Extracts phases → concepts, then clusters into focus groups

3. **Add domain clustering logic**
   - Analyze concept keywords
   - Group related concepts by domain
   - Suggest focus groups based on clusters (not 1:1)

4. **Update output format**
   ```yaml
   ANALYSIS_RESULT:
     suggested_concepts:
       - name: phase-slug-here
         source: "Phase N: Title"
         focus_group: suggested-fg
     suggested_focus_groups:
       - name: fg-name
         concepts: [concept-1, concept-2]
         reason: "Why this grouping"
     focus_group_count: N
   ```

#### Code Diff Summary

```bash
# OLD: analyze_roadmap() → SUGGESTED_FOCUS_GROUPS
# NEW: analyze_roadmap() → SUGGESTED_CONCEPTS + DOMAIN_CLUSTERS

# OLD: full_analysis() → focus_groups: [fg1, fg2, ...]
# NEW: full_analysis() → concepts: [...], focus_groups: [...] 
```

---

### Requirement 2: MIG-FIX-03 (Second)

**File:** `agents/planning-migrator.md`

Update after analyzer so it can consume new output format.

#### Changes Required

1. **Add Phase→Concept transformation**
   - Parse ROADMAP.md phase entries
   - Create concept file from each phase
   - Preserve phase scope, goals, requirements

2. **Update concept file template**
   ```markdown
   # Concept: {phase-title}
   
   **Status:** Migrated
   **Focus Group:** {assigned-fg}
   **Source:** GSD Phase {N}
   **Migrated:** {date}
   
   ## Description
   {phase description}
   
   ## Acceptance Criteria
   {phase requirements as checklist}
   
   ## Implementation Notes
   *To be populated*
   ```

3. **Update domain detection logic**
   - Keep requirement classification
   - Add phase classification for focus group assignment

4. **Add focus group assignment logic**
   - Each concept gets assigned to best-match focus group
   - Log reasoning for assignments

---

### Requirement 3: MIG-FIX-01 (Third/Final)

**File:** `workflows/migrate.md`

Final integration after both agents are updated.

#### Changes Required

1. **Update Step 3 (Analysis)**
   - Rename: "Analyzing for Concept Suggestions" (not Focus Group)
   - Display concepts extracted from phases
   - Display suggested focus groups (as organizational containers)

2. **Update Step 5 (Create Structure)**
   - Create `.planning/focus-groups/{fg}/concepts/{concept}.md`
   - Create concept files using planning-migrator agent

3. **Update Step 6 (Configuration)**
   - WGSD-CONFIG.md lists both concepts and focus groups
   - Concepts section shows source phase mapping

4. **Update Step 10 (Success Report)**
   - Report: "Created N concepts across M focus groups"
   - Show concept→focus group assignments

#### Specific Line Changes

```bash
# OLD Step 3:
echo "📊 Step 2: Analyzing for Focus Group Suggestions"

# NEW Step 3:
echo "📊 Step 2: Extracting Concepts from Phases"
echo ""
echo "📄 Analyzing ROADMAP.md..."
echo "   Found N phases → creating N concepts"
echo ""
echo "📂 Suggested Focus Groups:"
echo "   • core (3 concepts)"
echo "   • migration (2 concepts)"
```

---

## Wave Execution

### Wave 1: Analyzer Update (MIG-FIX-02)
**Duration:** ~25 min

| Task | Action |
|------|--------|
| 1.1 | Update `analyze_roadmap()` to extract concepts |
| 1.2 | Add domain clustering logic |
| 1.3 | Update `full_analysis()` output format |
| 1.4 | Update interactive mode output |

### Wave 2: Migrator Update (MIG-FIX-03)
**Duration:** ~15 min

| Task | Action |
|------|--------|
| 2.1 | Add Phase→Concept transformation function |
| 2.2 | Update concept file template |
| 2.3 | Add focus group assignment logic |
| 2.4 | Update migration mode output |

### Wave 3: Workflow Integration (MIG-FIX-01)
**Duration:** ~20 min

| Task | Action |
|------|--------|
| 3.1 | Update Step 3 to show concepts |
| 3.2 | Update Step 5 to create concept files |
| 3.3 | Update WGSD-CONFIG.md generation |
| 3.4 | Update success report |

---

## File Inventory

| File | Action | Changes Est. | Status |
|------|--------|--------------|--------|
| `agents/migration-analyzer.md` | Modify | ~150 lines | ⏳ |
| `agents/planning-migrator.md` | Modify | ~100 lines | ⏳ |
| `workflows/migrate.md` | Modify | ~80 lines | ⏳ |

**Total:** ~330 lines of changes

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Breaking existing migration | Test with fresh GSD project |
| Incorrect domain clustering | Keep fallback to "core" focus group |
| Config format change | Ensure backward compatibility |

---

## Dependencies

- **Blocking:** None
- **Blocked by this:** Phase 9 (Approval) needs correct mappings

---

## Success Criteria

- [ ] `wgsd analyze` shows "Suggested Concepts" (not Focus Groups)
- [ ] `wgsd migrate` creates concept files in focus group directories
- [ ] Focus Groups are 2-4 thematic containers (not 1:1 with phases)
- [ ] Each Concept links back to source Phase
- [ ] WGSD-CONFIG.md lists both concepts and focus groups

---

## Execution Command

```
/gsd build-phase 7
```

Or manually:
1. Update `agents/migration-analyzer.md`
2. Update `agents/planning-migrator.md`
3. Update `workflows/migrate.md`
4. Run verification

---

*Execution plan complete — Phase 7 ready for implementation*
