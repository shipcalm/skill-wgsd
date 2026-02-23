# WGSD v2.1 Roadmap ✅ COMPLETE

**Milestone:** Migration Experience Improvements  
**Created:** 2026-02-23  
**Completed:** 2026-02-23
**Total Phases:** 3 (Phases 7-9, continuing from v2.0)
**Status:** ✅ **ALL PHASES COMPLETE**

---

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    WGSD v2.1 ROADMAP                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 7              Phase 8              Phase 9         │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐      │
│   │ Logic   │         │ Slack   │         │Approval │      │
│   │  Fix    │────┬───►│ Auto    │────────►│Workflow │      │
│   └─────────┘    │    └─────────┘         └─────────┘      │
│                  │                              ▲           │
│                  └──────────────────────────────┘           │
│                                                             │
│   Parallel:        Sequential:      Depends on both         │
│   7 & 8            After 7 or 8     7 & 8 complete          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 7: Migration Logic Fix

**Goal:** Correct Phase → Concept mapping throughout migration system

**Duration:** ~1 hour  
**Priority:** P0 - Critical (blocks accurate migration)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| MIG-FIX-01 | Update migrate.md (Phase → Concept) | M |
| MIG-FIX-02 | Update migration-analyzer.md | S |
| MIG-FIX-03 | Update planning-migrator.md | S |

### Deliverables

1. **workflows/migrate.md** — Updated mapping logic
2. **agents/migration-analyzer.md** — Suggest Concepts from Phases
3. **agents/planning-migrator.md** — Transform Phase → Concept format

### Success Criteria

- [ ] migrate.md creates Concepts from Phases
- [ ] Focus Groups are suggested thematically (not 1:1)
- [ ] All references to "Phase → Focus Group" removed
- [ ] Migration produces correct WGSD structure

### Dependencies

- None (can start immediately)

---

## Phase 8: Slack Channel Automation

**Goal:** Auto-create all WGSD Slack channels during migration

**Duration:** ~1.5 hours  
**Priority:** P1 - High (manual work reduction)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| SLACK-AUTO-01 | Auto-create {stub}-dev | S |
| SLACK-AUTO-02 | Auto-create {stub}-fg-{focus} channels | M |
| SLACK-AUTO-03 | Auto-create {stub}-community | S |
| SLACK-AUTO-04 | Integrate into migrate.md flow | M |

### Deliverables

1. **workflows/migrate.md** — Integrated channel creation
2. Channel creation step with error handling
3. Rollback support for channel creation failures

### Success Criteria

- [ ] {stub}-dev channel created automatically
- [ ] One {stub}-fg-{name} channel per Focus Group
- [ ] {stub}-community public channel created
- [ ] All channels have appropriate Canvases
- [ ] Failures handled gracefully

### Dependencies

- Depends on existing `workflows/lib/slack-api.md` (from v2.0)
- Depends on existing `workflows/setup-core-channels.md` (from v2.0)

---

## Phase 9: Approval Workflow ✅ COMPLETE

**Goal:** Add interactive preview and approval before migration execution

**Duration:** ~1.5 hours  
**Priority:** P1 - High (user confidence)
**Status:** ✅ **COMPLETE** (2026-02-23)

### Requirements

| ID | Requirement | Effort | Status |
|----|-------------|--------|--------|
| APPROVE-01 | Generate migration preview | M | ✅ Done |
| APPROVE-02 | Display preview to user | S | ✅ Done |
| APPROVE-03 | Require explicit approval | S | ✅ Done |
| APPROVE-04 | Allow modification before approval | M | ✅ Done |

### Deliverables

1. **workflows/migrate-planning.md** — Full preview generation and approval workflow ✅
2. Approval gate with yes/no/edit confirmation ✅
3. Edit mode with rename-fg, move-concept, exclude-concept, add-fg commands ✅

### Success Criteria

- [x] Preview shows all proposed changes clearly
- [x] User must explicitly approve to proceed
- [x] User can modify Focus Group names
- [x] User can reassign Concepts
- [x] "No" cleanly aborts migration

### Dependencies

- **Phase 7**: Preview must show correct mappings (Concepts, not Focus Groups) ✅
- **Phase 8**: Preview must show channel names that will be created ✅

---

## Execution Plan

### Recommended Order

```
Day 1:
├── Phase 7: Logic Fix (parallel start)
└── Phase 8: Slack Automation (parallel start)

Day 1 (cont):
└── Phase 9: Approval Workflow (after 7 & 8)
```

### Parallel Execution Notes

Phases 7 and 8 modify different aspects of `migrate.md`:
- Phase 7: Mapping logic (which WGSD entities to create)
- Phase 8: Slack integration (channel creation calls)

They can be planned and developed in parallel, but final merge requires coordination to avoid conflicts.

---

## Risk Assessment

| Risk | Mitigation | Impact |
|------|------------|--------|
| migrate.md merge conflicts | Plan sequential execution if needed | Low |
| Slack API rate limits | Use existing rate limiting in slack-api.md | Low |
| User confusion on approval | Clear preview formatting | Medium |

---

## Verification Strategy

### Phase 7 Verification
- Run migrate analysis on test GSD project
- Verify output shows Concepts (not Focus Groups)
- Check generated structure matches WGSD model

### Phase 8 Verification
- Run migration with Slack integration
- Verify all channels created
- Verify Canvases attached correctly

### Phase 9 Verification
- Run migration and verify preview appears
- Test approval flow (yes/no)
- Test modification flow
- Verify modified values used in execution

---

## Summary

| Phase | Name | Requirements | Effort | Dependencies | Status |
|-------|------|--------------|--------|--------------|--------|
| 7 | Migration Logic Fix | 3 | ~1h | None | ✅ Complete |
| 8 | Slack Channel Automation | 4 | ~1.5h | v2.0 libs | ✅ Complete |
| 9 | Approval Workflow | 4 | ~1.5h | Phases 7 & 8 | ✅ Complete |
| **Total** | | **11** | **~4h** | | **✅ ALL DONE** |

---

## Next Steps

```
/gsd plan-phase 7
```

Or plan phases 7 & 8 in parallel:
```
/gsd plan-phase 7
/gsd plan-phase 8
```

---

*Roadmap for WGSD v2.1 — Migration Experience Improvements*
