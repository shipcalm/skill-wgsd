# Phase 19: Canvas Auto-Sync Implementation

**Status:** Planned  
**Created:** 2026-02-23  
**Dependencies:** Canvas infrastructure (existing)  
**Priority:** High (missing core feature)  

---

## Problem Statement

WGSD v2.2 canvas sync functionality is entirely manual, despite extensive documentation suggesting automatic triggers should exist. Throughout the codebase, there are commented-out canvas sync calls and "would invoke" statements where actual automation should be.

**Current State:** All canvas sync triggers are commented out or stubbed:
- `# Would invoke canvas-sync workflow`
- `# This would invoke workflows/canvas-sync.md` 
- `# /wgsd sync-canvas ${FOCUS_GROUP}`
- 8+ locations with non-functional sync triggers

**Impact:** Created canvases become stale immediately after planning changes, defeating the purpose of live visual collaboration.

**Evidence:** Marvin migration created 7 canvases that won't update automatically when concepts are merged or branches updated.

---

## Requirements

### REQ-19-01: Automatic Concept Merge Sync
**Priority:** Must Have  
When concepts are merged to focus group branches, automatically trigger canvas sync for affected focus group and master dashboard.

**Implementation:** Replace commented `# Would invoke canvas-sync` with actual calls.

### REQ-19-02: Automatic Branch Update Sync  
**Priority:** Must Have  
When focus group branches are updated (via direct commits or merges), automatically sync related canvases.

**Implementation:** Git hooks or webhook handlers to detect branch changes.

### REQ-19-03: Automatic Implementation State Sync
**Priority:** Must Have  
When implementations change state (started, completed, archived), automatically update implementation and master dashboard canvases.

**Implementation:** Hook into implementation workflow completion.

### REQ-19-04: Automatic Approval Matrix Sync
**Priority:** Should Have  
When concepts receive approvals or rejections, automatically update focus group canvases to reflect current approval state.

**Implementation:** Hook into matrix approval completion.

### REQ-19-05: Manual Override Capability
**Priority:** Must Have  
Preserve manual `/wgsd sync-canvas` commands for debugging and force-refresh scenarios.

**Implementation:** Ensure manual commands work alongside automatic triggers.

### REQ-19-06: Sync Failure Resilience  
**Priority:** Must Have  
Canvas sync failures should not block the underlying workflow (concept merges, etc.). Failed syncs should be logged and retryable.

**Implementation:** Non-blocking sync calls with error logging.

---

## Success Criteria

- [ ] Concept merges automatically update focus group + master dashboard canvases
- [ ] Focus group branch updates automatically sync related canvases
- [ ] Implementation state changes automatically update dashboards  
- [ ] Approval matrix changes automatically update focus group canvases
- [ ] Manual `/wgsd sync-canvas` commands continue to work
- [ ] Sync failures are non-blocking and logged for retry
- [ ] All commented "would invoke" statements replaced with actual calls
- [ ] Canvas content stays current without manual intervention

---

## Technical Implementation

### Locations Requiring Fixes

1. **workflows/concept-development.md** - Line 452-453: Replace commented sync with actual call
2. **workflows/lib/github-pr.md** - Line 449: Replace commented sync with actual call  
3. **workflows/matrix-approve.md** - Line 531-532: Replace commented sync with actual call
4. **workflows/promote-concept.md** - Line 361: Replace commented sync with actual call
5. **workflows/lib/approval-system.md** - Line 566: Uncomment and implement sync call
6. **workflows/create-implementation.md** - Line 523: Replace commented sync with actual call
7. **workflows/implementation-workflow.md** - Add automatic sync on state change
8. **workflows/complete-implementation.md** - Add automatic sync on completion

### Implementation Pattern

Replace this pattern:
```bash
# Would invoke canvas-sync workflow
# /wgsd sync-canvas ${FOCUS_GROUP}
```

With this pattern:
```bash
# Trigger canvas sync for updated planning
echo "🔄 Auto-syncing canvas for ${FOCUS_GROUP}..."
/wgsd sync-canvas fg "${FOCUS_GROUP}" || echo "⚠️ Canvas sync failed (non-blocking)"
```

---

## Risk Assessment

- **🟡 Medium Risk:** Canvas API calls can fail, must not block primary workflows
- **🟡 Medium Risk:** Performance impact of frequent canvas updates
- **🟢 Low Risk:** Canvas sync workflow already exists and is tested

## Mitigation Strategies

- **Non-blocking calls:** Canvas sync failures won't abort concept merges
- **Async execution:** Canvas updates happen in background where possible  
- **Rate limiting:** Batch multiple changes to avoid API rate limits
- **Error logging:** Failed syncs logged for manual retry

---

## Dependencies

- **Canvas Sync Workflow:** `workflows/canvas-sync.md` (exists)
- **Canvas Registry:** Canvas tracking system (exists)  
- **Slack Canvas API:** Permissions and API access (exists)

---

*Phase created to implement missing automatic canvas sync triggers identified during Marvin deployment*
