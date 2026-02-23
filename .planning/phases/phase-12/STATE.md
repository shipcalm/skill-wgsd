# Phase 12: Matrix-Based Approval System — State

**Status:** 🔵 **PLANNED — Ready for Execution**  
**Updated:** 2026-02-23  
**Depends On:** Phase 10 ✅, Phase 11 ✅

---

## Quick Status

| Requirement | Status | Notes |
|-------------|--------|-------|
| MATRIX-01: Approval Matrix Data Structure | ⬜ Ready | Foundation, execute first |
| MATRIX-02: Per-Focus-Group Approval | ⬜ Ready | Core workflow |
| MATRIX-03: Approval Matrix Canvas Widget | ⬜ Ready | Visual layer |
| MATRIX-04: Approval Completion Detection | ⬜ Ready | Enables Phase 13 |
| MATRIX-05: Blocked Status Handling | ⬜ Ready | Advanced dependencies |
| MATRIX-06: Approval Override (Admin) | ⬜ Ready | Emergency mechanism |

**Progress:** 0/6 requirements (0%)

---

## Execution Checklist

### Pre-Execution
- [x] Phase 10 complete (concept directories)
- [x] Phase 11 complete (impact system with notifications)
- [x] Impact-parser library ready
- [x] Impact-notifications library ready
- [x] Detailed plan created

### Execution Order

**Step 1: MATRIX-01 (Data Structure)**
- [ ] Enhance impact-matrix.md.tmpl with new fields
- [ ] Add approval matrix parsing to impact-parser.md
- [ ] Add SLA calculation function
- [ ] Test parsing with sample data

**Step 2: MATRIX-02 (Per-FG Approval)**
- [ ] Create workflows/matrix-approve.md
- [ ] Implement approval_matrix_approve function
- [ ] Implement approval_matrix_reject function
- [ ] Add authority validation
- [ ] Send approval notifications

**Step 3: MATRIX-04 (Completion Detection)**
- [ ] Implement approval_matrix_is_complete function
- [ ] Implement approval_matrix_get_blockers function
- [ ] Add completion trigger to approval workflow
- [ ] Send celebration notification on complete

**Step 4: MATRIX-03 (Canvas Widget)**
- [ ] Add template_approval_matrix to canvas-templates.md
- [ ] Add format_approval_status helper
- [ ] Add format_priority_badge helper
- [ ] Add format_sla_status helper
- [ ] Integrate into concept canvas sync

**Step 5: MATRIX-05 (Blocked Status)**
- [ ] Create workflows/matrix-block.md
- [ ] Implement approval_matrix_block function
- [ ] Implement auto-unblock on blocker approval
- [ ] Add blocked dependencies to Canvas widget

**Step 6: MATRIX-06 (Admin Override)**
- [ ] Create workflows/approval-override.md
- [ ] Implement admin detection
- [ ] Add override fields to schema
- [ ] Send override notification
- [ ] Preserve audit trail

---

## Key Design Decisions

1. **Single Source of Truth**: Approval state lives in impact-matrix.md, not separate JSON
2. **Independent Approvals**: Each FG's decision doesn't affect others
3. **SLA Enforcement**: Priority-based deadlines (P0=4h, P1=24h, P2=72h, P3=7d)
4. **Cascade Blocking**: Optional cascade when FG rejects
5. **Override Audit**: Full audit trail for admin overrides

---

## Files to Create/Modify

| File | Action | Size |
|------|--------|------|
| `workflows/matrix-approve.md` | Create | ~200 lines |
| `workflows/matrix-block.md` | Create | ~100 lines |
| `workflows/approval-override.md` | Create | ~100 lines |
| `workflows/lib/approval-system.md` | Enhance | +150 lines |
| `workflows/lib/canvas-templates.md` | Enhance | +100 lines |
| `workflows/lib/impact-parser.md` | Enhance | +50 lines |
| `templates/concept-directory/impact-matrix.md.tmpl` | Enhance | +20 lines |

---

## Test Scenarios

1. **Happy Path**: Concept with 3 FGs, all approve → fully approved
2. **Partial Approval**: 2/3 approved → pending state shown
3. **Rejection**: 1 FG rejects → concept returns to draft
4. **Blocking**: Frontend blocked by API → auto-unblocks when API approves
5. **Admin Override**: API team unavailable → admin overrides
6. **SLA Warning**: P0 impact pending >3h → warning shown

---

## Enables

- **Phase 13**: Roadmap Branch Architecture (approved concepts merge to roadmap)
- **Phase 14**: Slack-Native Approval Workflow (backend for `/wgsd approve`)

---

## Notes

*Execution notes will be added during implementation.*

---

*To execute: `/gsd execute-phase 12`*
