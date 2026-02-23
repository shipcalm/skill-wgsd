# Phase 17: Concept Independence Migration

**Status:** 📋 Planning  
**Priority:** P0 - Critical (foundation for v2.3)  
**Duration:** ~3-4 hours  
**Dependencies:** WGSD v2.2 complete

---

## Objective

Migrate from focus-group-owned concepts to independent concept architecture, eliminating duplication risk and enabling true many-to-many concept/focus-group relationships.

**Migration:** `.planning/focus-groups/{fg}/concepts/{concept}/` → `.planning/concepts/{concept}/`

---

## Requirements

| ID | Requirement | Status | Effort |
|----|-------------|--------|--------|
| CONCEPT-INDEP-01 | Directory structure migration | ⬜ | M |
| CONCEPT-INDEP-02 | Impact matrix schema updates | ⬜ | S |
| CONCEPT-INDEP-03 | Workflow path updates | ⬜ | L |
| CONCEPT-INDEP-04 | Backward compatibility layer | ⬜ | M |

---

## Current vs Target Architecture

### Current (v2.2)
```
.planning/focus-groups/
├── security/concepts/
│   ├── oauth-integration/
│   └── auth-v2/
├── api/concepts/
│   ├── versioning/
│   └── rate-limiting/
└── frontend/concepts/
    └── component-library/
```

### Target (v2.3)  
```
.planning/concepts/
├── oauth-integration/          # Was: security/concepts/oauth-integration/
├── auth-v2/                   # Was: security/concepts/auth-v2/
├── api-versioning/            # Was: api/concepts/versioning/
├── rate-limiting/             # Was: api/concepts/rate-limiting/
└── component-library/         # Was: frontend/concepts/component-library/
```

---

## Implementation Strategy

### Step 1: Discovery & Mapping
1. **Scan existing repositories** for current concept locations
2. **Build concept registry** with current paths and metadata
3. **Detect naming conflicts** (e.g., multiple "auth" concepts)
4. **Generate migration plan** with conflict resolution

### Step 2: Schema Updates
1. **Enhanced impact-matrix.md** - Add originating focus group metadata
2. **Concept ownership** - Track primary focus group for historical context
3. **Cross-reference updates** - Update all concept references

### Step 3: Migration Execution
1. **Create `.planning/concepts/` directory**
2. **Move concept directories** with full git history preservation
3. **Update all workflow files** to use new paths
4. **Rebuild focus group roadmaps** to reference independent concepts

### Step 4: Validation & Compatibility
1. **Backward compatibility layer** - Support old paths during transition
2. **Validation scripts** - Ensure all references updated correctly
3. **Migration rollback** - Complete reversal capability if needed

---

## Deliverables

### Core Migration
1. **workflows/migrate-to-independent-concepts.md** - Main migration workflow
2. **workflows/lib/concept-discovery.md** - Concept scanning and mapping
3. **templates/concept-directory-v23/** - Updated v2.3 concept templates

### Schema Enhancements  
1. **Enhanced impact-matrix.md template** - v2.3 format with originating FG
2. **Updated CONCEPT.md template** - Independent concept format
3. **Focus group roadmap templates** - Reference independent concepts

### Workflow Updates
1. **Updated create-concept.md** - Create in `.planning/concepts/`
2. **Updated declare-impact.md** - Work with independent concepts  
3. **Updated all workflow references** - Use new concept paths
4. **Compatibility shims** - Support old paths temporarily

---

## Conflict Resolution Strategy

### Naming Conflicts
**Problem:** Multiple focus groups have concepts with same name
```
security/concepts/auth/     ← Conflict!
api/concepts/auth/         ← Conflict!  
```

**Resolution:**
1. **Automatic prefixing:** `auth` → `security-auth` + `api-auth`
2. **Interactive merge prompt:** Offer to merge if concepts are similar
3. **Manual resolution:** Require explicit naming during migration

### Cross-References
**Problem:** Concepts reference other concepts by focus group path

**Resolution:**
1. **Automated path updates** in all `.md` files
2. **Validation scan** to ensure no broken references
3. **Reference registry** to track all concept interconnections

---

## Git Integration

### Branch Migration
**Current branches:**
```
concepts/security-auth-v2           # Was tied to security focus group
concepts/api-versioning            # Was tied to api focus group
```

**Target branches:**
```
concepts/auth-v2                   # Independent concept
concepts/api-versioning           # Independent concept  
```

### History Preservation
- **Git subtree/filter-branch** to preserve concept development history
- **Maintain original commit authors** and timestamps
- **Update commit messages** to reference new independent paths

---

## Rollback Strategy

**Complete rollback capability:**
1. **Snapshot current state** before migration begins
2. **Incremental rollback** - Roll back individual concepts if issues
3. **Full system rollback** - Return to v2.2 architecture if major issues
4. **Validation checkpoints** throughout migration process

---

## Testing Strategy

### Pre-Migration Validation
- [ ] All existing concepts discovered correctly
- [ ] No naming conflicts unresolved  
- [ ] All cross-references mapped
- [ ] Migration plan reviewed and approved

### Post-Migration Validation  
- [ ] All concepts accessible at new paths
- [ ] All workflows function with new architecture
- [ ] No broken concept references
- [ ] All git history preserved correctly
- [ ] Canvas integration still functional

### Compatibility Testing
- [ ] Old workflows still work (backward compatibility)
- [ ] New workflows work with independent concepts
- [ ] Migration can be rolled back successfully
- [ ] All Slack integrations function correctly

---

## Risks & Dependencies

### High Risk
- **Data loss during migration** - Comprehensive backup required
- **Broken workflow references** - Need compatibility layer
- **Git branch conflicts** - Careful branch management needed

### Medium Risk  
- **Canvas sync issues** - May need canvas regeneration
- **Complex concept relationships** - Impact matrix updates required
- **Team workflow disruption** - Clear communication needed

### Dependencies
- WGSD v2.2 fully deployed and stable
- All teams aware of upcoming architecture change
- Backup procedures tested and validated

---

## Definition of Done

- [ ] All concepts moved to `.planning/concepts/` directory
- [ ] Zero concept duplication across focus groups
- [ ] All workflows updated to use independent concept paths  
- [ ] Backward compatibility layer functional
- [ ] Migration rollback tested and working
- [ ] All existing functionality preserved
- [ ] Documentation updated for v2.3 architecture

---

**Phase 17 establishes the foundation for independent concept architecture, enabling all subsequent v2.3 merge/split operations.**

*Ready for detailed implementation planning and execution.*