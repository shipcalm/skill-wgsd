# Phase 4: Canvas Management System - Execution Plan

**Date:** 2026-02-22
**Phase:** 4 of 6
**Requirements:** 9 (CANVAS-01 through CANVAS-08, INTEGRATE-05)
**Duration:** 4-5 days estimated

---

## Requirements Mapping

| Req ID | Requirement | Wave | Deliverable |
|--------|-------------|------|-------------|
| INTEGRATE-05 | Canvas API integration | Wave 1 | `lib/canvas-api.md` |
| CANVAS-01 | Master dashboard Canvas | Wave 2 | `canvas-create.md` |
| CANVAS-08 | Implementation dashboard | Wave 2 | `canvas-create.md` |
| CANVAS-04 | Focus group Canvas | Wave 3 | `canvas-create.md` |
| CANVAS-06 | Bidirectional sync | Wave 3 | `canvas-sync.md` |
| CANVAS-03 | Auto-update on state change | Wave 3 | `canvas-sync.md` |
| CANVAS-07 | Community roadmap Canvas | Wave 4 | `canvas-create.md` |
| CANVAS-02 | Live roadmap visualization | Wave 4 | `canvas-sync.md` |
| CANVAS-05 | AI-managed enforcement | Wave 4 | `canvas-enforce.md` |

---

## Wave 1: Canvas API Foundation

### Deliverables

1. **`workflows/lib/canvas-api.md`** - Extended Canvas API library
   - `canvas_create_attached` - Create canvas attached to channel
   - `canvas_create_standalone` - Create standalone canvas
   - `canvas_edit` - Edit canvas content
   - `canvas_edit_section` - Edit specific section
   - `canvas_lookup_sections` - Find sections by content
   - `canvas_delete` - Delete canvas (for cleanup)

2. **`workflows/lib/canvas-registry.md`** - Canvas tracking system
   - `registry_init` - Initialize canvas registry
   - `registry_add` - Add canvas to registry
   - `registry_get` - Get canvas by type/key
   - `registry_update_sync` - Update last sync timestamp
   - `registry_list` - List all registered canvases

3. **`workflows/lib/canvas-templates.md`** - Template engine
   - `template_master_dashboard` - Master dashboard template
   - `template_focus_group` - Focus group template
   - `template_implementation` - Implementation dashboard template
   - `template_community` - Community roadmap template
   - `template_render` - Variable substitution engine

### Success Criteria
- [ ] Canvas API wrapper functions work with Slack API
- [ ] Registry tracks canvases in `.planning/CANVAS-REGISTRY.json`
- [ ] Templates render correctly with variable substitution

---

## Wave 2: Master Dashboard + Implementation Dashboard

### Deliverables

1. **`workflows/canvas-create.md`** - Canvas creation workflow
   - Create master dashboard in dev channel
   - Create implementation dashboard (same channel, different canvas)
   - Populate from .planning/ state
   - Register in canvas registry

2. **Template Content: Master Dashboard**
   - WGSD methodology guide section
   - Active focus groups list
   - Current implementations summary
   - Quick links to resources

3. **Template Content: Implementation Dashboard**
   - Active implementations with status
   - Implementation queue
   - Recently completed
   - Velocity metrics placeholder

### Success Criteria
- [ ] Master dashboard Canvas created in dev channel
- [ ] Implementation dashboard Canvas shows active/queued/completed
- [ ] Canvases registered in registry
- [ ] CANVAS-01, CANVAS-08, INTEGRATE-05 verified

---

## Wave 3: Focus Group Canvas + Sync

### Deliverables

1. **`workflows/canvas-sync.md`** - Bidirectional sync workflow
   - `sync_canvas_from_git` - Push git state to canvas
   - `sync_git_from_canvas` - Pull canvas to git (AI-only)
   - `sync_focus_group` - Sync specific focus group
   - `sync_all` - Full sync of all canvases

2. **Enhanced `workflows/create-focus-group.md`**
   - Auto-create focus group canvas on FG creation
   - Populate from migrated planning docs
   - Link to focus group STATE.md

3. **State Aggregation System**
   - Scan `.planning/focus-groups/*/STATE.md`
   - Parse concept files for status
   - Build aggregate state for dashboards

### Success Criteria
- [ ] Focus group canvas created with FG
- [ ] Canvas populated from .planning/ files
- [ ] Sync workflow updates canvas when state changes
- [ ] CANVAS-04, CANVAS-06, CANVAS-03 verified

---

## Wave 4: Community + AI Enforcement

### Deliverables

1. **Community Roadmap Canvas**
   - Sanitized public view (no internal details)
   - Focus areas without implementation details
   - Recently shipped features
   - Contribution invitation

2. **Live Roadmap Visualization**
   - Cross-focus-group aggregation
   - Visual status indicators (emoji-based)
   - Links between related items

3. **`workflows/canvas-enforce.md`** - AI-only enforcement
   - Detect canvas edits
   - Warn about direct editing policy
   - Revert unauthorized changes
   - Guide to proper workflow

### Success Criteria
- [ ] Community canvas shows public-safe content
- [ ] Roadmap visualization works across focus groups
- [ ] AI enforcement prevents human direct edits
- [ ] CANVAS-07, CANVAS-02, CANVAS-05 verified

---

## Execution Order

```
Wave 1 (Foundation)
├── lib/canvas-api.md
├── lib/canvas-registry.md
└── lib/canvas-templates.md

Wave 2 (Core Dashboards)
├── canvas-create.md
├── Master dashboard template
└── Implementation dashboard template

Wave 3 (Focus Group + Sync)
├── canvas-sync.md
├── create-focus-group.md enhancement
└── State aggregation

Wave 4 (Community + Enforcement)
├── Community canvas
├── Live roadmap
└── canvas-enforce.md
```

---

## File Inventory

### New Files
- `workflows/lib/canvas-api.md` (~350 lines)
- `workflows/lib/canvas-registry.md` (~250 lines)
- `workflows/lib/canvas-templates.md` (~400 lines)
- `workflows/canvas-create.md` (~400 lines)
- `workflows/canvas-sync.md` (~450 lines)
- `workflows/canvas-enforce.md` (~200 lines)

### Modified Files
- `workflows/create-focus-group.md` - Add canvas creation
- `workflows/create-implementation.md` - Trigger dashboard update
- `SKILL.md` - Add canvas routing

### New Config Files
- `.planning/CANVAS-REGISTRY.json` - Canvas tracking

### Total Estimated
- ~2050 lines new code
- ~100 lines modifications

---

## Verification Plan

### Unit Verification
- [ ] Canvas API functions execute without error
- [ ] Registry CRUD operations work
- [ ] Templates render with correct substitution

### Integration Verification
- [ ] Create canvas in test channel
- [ ] Edit canvas content via API
- [ ] Sync from git to canvas
- [ ] Focus group creation includes canvas

### End-to-End Verification
- [ ] Full workflow: create FG → canvas appears → edit state → canvas updates
- [ ] Community canvas shows sanitized content
- [ ] Human edit warning appears

---

*Plan created: 2026-02-22*
*Ready for execution*
