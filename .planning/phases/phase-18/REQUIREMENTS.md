# Phase 18: Migration Auto-Canvas Creation

**Status:** Planned  
**Created:** 2026-02-23  
**Dependencies:** Phase 17 (channels must exist first)  
**Priority:** High  

---

## Problem Statement

WGSD v2.2 migration workflow does not create Slack canvases automatically. Users must manually run canvas creation commands after migration, leaving the essential visual collaboration aspect of WGSD unavailable until manual intervention.

**Missing Canvases:**
- Master dashboard in dev channel (aggregated roadmap)
- Focus group canvases in fg channels (roadmaps per focus group)  
- Implementation canvases in impl channels (active work tracking)

**Evidence:** Marvin migration (2026-02-23) had no canvases created, requiring manual setup.

---

## Requirements

### REQ-18-01: Automatic Master Dashboard Creation
**Priority:** Must Have  
Migration workflow must automatically create master dashboard canvas in the dev channel with aggregated project roadmap.

### REQ-18-02: Automatic Focus Group Canvas Creation
**Priority:** Must Have  
Migration workflow must automatically create focus group canvases in each fg channel with relevant roadmap data.

### REQ-18-03: Automatic Implementation Canvas Creation  
**Priority:** Should Have  
Migration workflow should automatically create implementation canvases for any active implementations preserved during migration.

### REQ-18-04: Canvas Registry Initialization
**Priority:** Must Have  
Migration workflow must initialize and populate the canvas registry (`CANVAS-REGISTRY.json`) for ongoing canvas management.

### REQ-18-05: Dynamic Content Population
**Priority:** Must Have  
Auto-created canvases must be populated with current roadmap data from `.planning/` structure, not empty templates.

### REQ-18-06: Graceful Canvas Failures
**Priority:** Must Have  
Canvas creation failures must be non-fatal to migration process (warn but continue).

---

## Success Criteria

- [ ] Migration creates master dashboard canvas in dev channel
- [ ] Migration creates focus group canvases in all fg channels  
- [ ] Migration creates implementation canvas for active work
- [ ] Canvas registry is initialized and populated
- [ ] Canvases contain relevant roadmap data from migration
- [ ] Canvas creation failures are handled gracefully (non-fatal)
- [ ] Migration success report shows created canvases with IDs

---

## Technical Notes

- Integration point: After Phase 17 (channel creation) in `workflows/migrate.md`
- Use existing `canvas-create.md` workflows for consistency
- Dependency: Slack Canvas API access and permissions
- Content source: Current `.planning/` structure data  
- Registry: Initialize `CANVAS-REGISTRY.json` automatically

---

## Risk Assessment

- **Medium Risk:** Canvas API is newer Slack feature, potential for API failures
- **Mitigation:** Non-fatal failures, graceful degradation to manual canvas creation
- **Fallback:** Document manual canvas creation in success report if auto-creation fails

---

*Phase created to address missing visual collaboration setup in migration workflow*
