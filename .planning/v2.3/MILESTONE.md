# WGSD v2.3 - Independent Concept Architecture

**Status:** 📋 Planning  
**Started:** 2026-02-23  
**Dependencies:** WGSD v2.2 (Complete)

---

## Vision

Transform from focus-group-owned concepts to independent concept management with merge/split operations for both concepts and focus groups.

**Current (v2.2):** `.planning/focus-groups/{fg}/concepts/{concept}/`  
**Target (v2.3):** `.planning/concepts/{concept}/` (independent)

---

## Problems Solved

1. **Concept Duplication** - No accidental copying across focus groups
2. **Cross-Cutting Clarity** - Concepts naturally span multiple focus groups  
3. **Focus Group Evolution** - Ability to merge/split focus groups as teams evolve
4. **Concept Lifecycle** - Merge related concepts, split complex ones

---

## Architecture Changes

### New Structure
```
{repo}/.planning/
├── concepts/                       # ✨ Independent concept directory
│   ├── oauth-integration/          # ✨ Concept lives independently
│   │   ├── CONCEPT.md             
│   │   ├── impact-matrix.md        # Declares impacts on multiple FGs
│   │   ├── API-SPEC.md            
│   │   └── acceptance-criteria.md  
│   ├── user-onboarding/
│   └── billing-v3/
├── focus-groups/                   # Focus groups reference concepts
│   ├── security/
│   │   ├── ROADMAP.md             # References independent concepts
│   │   ├── STATE.md               
│   │   └── CHANNELS.md            
│   ├── api/
│   └── frontend/
├── active-implementations/         # Unchanged
├── MASTER-ROADMAP.md              # Cross-concept aggregation
└── WGSD-CONFIG.md                 # Updated concept mappings
```

### Git Branch Changes
```
main/develop (primary)
├── roadmap                        # Approved concepts (unchanged)
├── concepts/                      # ✨ Independent concept branches
│   ├── oauth-integration          # No focus group ownership
│   ├── user-onboarding           
│   └── billing-v3                
├── focus-groups/                  # Focus group planning only
│   ├── security                   # No concept ownership
│   ├── api                       
│   └── frontend                  
└── implementations/               # Implementation branches (unchanged)
```

---

## Phase Breakdown

| Phase | Name | Requirements | Effort | Priority |
|-------|------|--------------|--------|----------|
| 17 | Concept Independence Migration | 4 | M | P0 |
| 18 | Concept Merge Operations | 3 | M | P1 |
| 19 | Concept Split Operations | 3 | M | P1 |
| 20 | Focus Group Merge Operations | 4 | L | P2 |
| 21 | Focus Group Split Operations | 4 | L | P2 |
| 22 | Migration & Compatibility | 3 | M | P0 |

**Total:** 21 requirements, ~10-15 hours

---

## Phase 17: Concept Independence Migration

**Objective:** Move concepts from focus-group ownership to independent management.

### Requirements
- **CONCEPT-INDEP-01:** Migrate existing `.planning/focus-groups/{fg}/concepts/` → `.planning/concepts/`
- **CONCEPT-INDEP-02:** Update impact-matrix.md schema for independent concepts
- **CONCEPT-INDEP-03:** Update all workflows to use independent concept paths
- **CONCEPT-INDEP-04:** Maintain backward compatibility during transition

### Deliverables
- Migration script for existing repositories
- Updated workflow templates
- Enhanced impact matrix format

---

## Phase 18: Concept Merge Operations

**Objective:** Combine related concepts into unified concepts.

### Requirements
- **CONCEPT-MERGE-01:** `/wgsd merge-concepts source-concept target-concept`
- **CONCEPT-MERGE-02:** Impact matrix consolidation logic
- **CONCEPT-MERGE-03:** Git history preservation for merged concepts

### Use Cases
- Merge "oauth-v1" + "oauth-v2" → "oauth-integration"
- Combine duplicate concepts discovered across teams
- Consolidate related security concepts

---

## Phase 19: Concept Split Operations

**Objective:** Split complex concepts into focused sub-concepts.

### Requirements
- **CONCEPT-SPLIT-01:** `/wgsd split-concept parent-concept child1-concept child2-concept`
- **CONCEPT-SPLIT-02:** Impact inheritance distribution
- **CONCEPT-SPLIT-03:** Approval status distribution logic

### Use Cases  
- Split "user-management" → "user-auth" + "user-profile" + "user-permissions"
- Break complex concepts for parallel development
- Separate concerns for better approval matrix

---

## Phase 20: Focus Group Merge Operations

**Objective:** Consolidate related focus groups as teams evolve.

### Requirements
- **FG-MERGE-01:** `/wgsd merge-focus-groups source-fg target-fg`
- **FG-MERGE-02:** Channel consolidation workflow  
- **FG-MERGE-03:** Concept impact reassignment
- **FG-MERGE-04:** Canvas and roadmap consolidation

### Use Cases
- Merge "api-v1" + "api-v2" → "api"
- Consolidate "frontend" + "mobile" → "client"
- Team restructuring scenarios

---

## Phase 21: Focus Group Split Operations  

**Objective:** Split broad focus groups into specialized teams.

### Requirements
- **FG-SPLIT-01:** `/wgsd split-focus-group parent-fg child1-fg child2-fg`
- **FG-SPLIT-02:** Concept impact redistribution
- **FG-SPLIT-03:** Channel creation for new focus groups
- **FG-SPLIT-04:** Member assignment workflows

### Use Cases
- Split "backend" → "api" + "database" + "infrastructure"  
- Separate "security" → "auth" + "compliance" + "infra-security"
- Growing team specialization

---

## Phase 22: Migration & Compatibility

**Objective:** Smooth transition from v2.2 to v2.3 architecture.

### Requirements
- **MIGRATE-V23-01:** Automated v2.2 → v2.3 migration command
- **MIGRATE-V23-02:** Backward compatibility layer for existing workflows  
- **MIGRATE-V23-03:** Validation and rollback mechanisms

---

## Success Metrics

- [ ] Zero concept duplication across focus groups
- [ ] Clean concept → focus group many-to-many relationships  
- [ ] Successful merge/split operations with full audit trails
- [ ] Migration from v2.2 completes without data loss
- [ ] All existing workflows continue to function

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Data loss during migration | HIGH | Comprehensive backup + rollback procedures |
| Broken workflows | MEDIUM | Backward compatibility layer + extensive testing |
| Complex approval matrix | MEDIUM | Enhanced visualization tools |
| Git branch conflicts | LOW | Careful branch management + worktree updates |

---

**v2.3 represents a fundamental architectural evolution toward true many-to-many concept/focus-group relationships with powerful merge/split operations.**

*Ready for Greg's approval to begin Phase 17 planning and execution.*