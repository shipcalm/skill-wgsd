# Phase 12: Matrix-Based Approval System

**Status:** 🔵 Planned  
**Priority:** P0 - Critical  
**Duration:** ~3-4 hours  
**Dependencies:** Phase 10 (Concept Directories) ✅, Phase 11 (Cross-Cutting Impacts) ✅

---

## Objective

Transform approval from single focus group ownership to a sophisticated multi-stakeholder approval matrix. Each impacted focus group independently reviews and approves with their own priority, creating a comprehensive approval workflow that:
- Enables parallel approvals across multiple stakeholders
- Tracks per-focus-group status with SLA enforcement
- Detects blocking dependencies between focus groups
- Provides visual Canvas representation of approval state
- Supports admin overrides for urgent situations

---

## Requirements

| ID | Requirement | Status | Effort | Execution Order |
|----|-------------|--------|--------|-----------------|
| MATRIX-01 | Approval matrix data structure | ⬜ | M | 1 |
| MATRIX-02 | Per-focus-group approval | ⬜ | M | 2 |
| MATRIX-03 | Approval matrix Canvas widget | ⬜ | L | 4 |
| MATRIX-04 | Approval completion detection | ⬜ | M | 3 |
| MATRIX-05 | Blocked status handling | ⬜ | M | 5 |
| MATRIX-06 | Approval override (admin) | ⬜ | M | 6 |

---

## Detailed Execution Plan

### MATRIX-01: Approval Matrix Data Structure

**Goal:** Create machine-readable approval tracking per concept

**Design Decision:** Store approval state in impact-matrix.md YAML, not separate JSON
- Rationale: Single source of truth, leverages existing impact-parser
- Each impact entry already has `status`, `approver`, `approved_date` fields
- Add new fields for matrix-specific tracking

**Enhanced Schema:**
```yaml
impacts:
  - focus_group: security
    priority: P0
    type: breaking-change
    description: "Token validation changes"
    # Approval tracking (existing fields)
    status: pending | approved | rejected | blocked
    approver: "@alice" | null
    approved_date: "2026-02-23" | null
    # New matrix-specific fields
    approval_comment: "Reviewed token flow, LGTM" | null
    blocked_by: "api" | null          # Which FG is blocking this
    blocked_reason: "Waiting on API spec" | null
    sla_deadline: "2026-02-23T16:00:00Z" | null  # Based on priority
    overridden: false
    override_by: null
    override_date: null
    override_reason: null
```

**Files to Modify:**
- `workflows/lib/impact-parser.md` — Add approval matrix parsing functions
- `templates/concept-directory/impact-matrix.md.tmpl` — Add new fields

**New Functions:**
```bash
# Get approval matrix state for concept
approval_matrix_get(concept_path) → {
  "concept": "oauth-integration",
  "total_approvals": 4,
  "approved": 2,
  "pending": 1,
  "blocked": 1,
  "rejected": 0,
  "fully_approved": false,
  "blocking_fgs": ["api"],
  "details": [...]
}

# Calculate SLA deadline based on priority
approval_matrix_calculate_sla(priority, start_date) → deadline_timestamp

# Check if approval is overdue
approval_matrix_check_sla(concept_path) → {
  "overdue": ["security"],
  "approaching": ["api"],
  "ok": ["frontend", "docs"]
}
```

---

### MATRIX-02: Per-Focus-Group Approval

**Goal:** Enable independent approval from each impacted focus group

**Key Principle:** Each focus group's approval decision is independent
- Security approving doesn't affect API's decision
- Partial approval state is valid and visible
- Re-approval required when impact is modified (already implemented in Phase 11)

**Workflow: `workflows/matrix-approve.md`**

```yaml
Entry Points:
  - "/wgsd approve {concept} --fg {focus-group}"
  - "/wgsd approve oauth-integration --fg security"
  - Auto-triggered from Slack reaction/thread

Steps:
  1. Validate user has authority for focus group
  2. Load concept's impact matrix
  3. Update specific focus group's status
  4. Send confirmation notification
  5. Check for full approval (MATRIX-04)
  6. Update Canvas (MATRIX-03)
```

**Authority Validation:**
```bash
# Check if user can approve for focus group
approval_matrix_can_approve(user, focus_group) → boolean

# Get user's focus group affiliations
approval_matrix_get_user_fgs(user) → ["security", "api"]
```

**Approve Action:**
```bash
approval_matrix_approve(concept_path, focus_group, user, comment) {
  # 1. Validate authority
  # 2. Check current status (can't approve if already approved)
  # 3. Update impact-matrix.md
  # 4. Record timestamp and approver
  # 5. Clear any blocked_by this FG
  # 6. Notify concept channel
  # 7. Check for completion
}
```

**Reject Action:**
```bash
approval_matrix_reject(concept_path, focus_group, user, reason) {
  # 1. Validate authority
  # 2. Update status to "rejected"
  # 3. Record rejection reason
  # 4. Notify concept owner
  # 5. Block dependent FGs (optional cascade)
}
```

---

### MATRIX-03: Approval Matrix Canvas Widget

**Goal:** Visual representation of approval status in Canvas

**Widget Template:**
```markdown
## 📊 Approval Matrix

| Focus Group | Priority | Status | Approver | Date | SLA |
|-------------|----------|--------|----------|------|-----|
| 🔐 Security | 🔴 P0 | ✅ Approved | @alice | Feb 23 | ✓ |
| 🔌 API | 🟠 P1 | ⏳ Pending | — | — | ⚠️ 4h |
| 🎨 Frontend | 🟡 P2 | ⏸️ Blocked | — | — | Waiting on API |
| 📖 Docs | 🟢 P3 | ⏳ Pending | — | — | ✓ |

**Overall Status:** 🟡 Partially Approved (1/4)
**Blocking Issues:** API approval required before Frontend can proceed
```

**Color Coding:**
- ✅ Green = Approved
- ⏳ Yellow = Pending
- 🚫 Red = Rejected
- ⏸️ Gray = Blocked

**Files to Modify:**
- `workflows/lib/canvas-templates.md` — Add `template_approval_matrix` function
- `workflows/canvas-sync.md` — Include approval matrix in concept canvas

**New Functions:**
```bash
# Generate approval matrix widget
template_approval_matrix(concept_path) → markdown_string

# Format status with emoji
format_approval_status(status) → "✅ Approved" | "⏳ Pending" | ...

# Format priority with color
format_priority_badge(priority) → "🔴 P0" | "🟠 P1" | ...

# Format SLA status
format_sla_status(deadline, status) → "✓" | "⚠️ 4h" | "⛔ OVERDUE"
```

---

### MATRIX-04: Approval Completion Detection

**Goal:** Automatically detect when all required approvals are complete

**Completion Criteria:**
1. All impacts have status = "approved" (no pending, rejected, blocked)
2. No blocking dependencies remain
3. Primary owner focus group must be approved

**Detection Logic:**
```python
def is_fully_approved(impacts):
    required_approved = all(
        i['status'] == 'approved' 
        for i in impacts 
        if i.get('required', True)  # Future: optional approvals
    )
    no_rejections = not any(i['status'] == 'rejected' for i in impacts)
    no_blocked = not any(i['status'] == 'blocked' for i in impacts)
    
    return required_approved and no_rejections and no_blocked
```

**On Full Approval:**
1. Update concept status to "Ready for Implementation"
2. Notify concept channel
3. Trigger roadmap merge workflow (Phase 13)
4. Update master Canvas

**Files to Create/Modify:**
- `workflows/lib/approval-system.md` — Add completion detection
- `workflows/promote-concept.md` — Hook into completion trigger

**New Functions:**
```bash
# Check if concept is fully approved
approval_matrix_is_complete(concept_path) → boolean

# Get blocking issues preventing completion
approval_matrix_get_blockers(concept_path) → [
  {"fg": "api", "status": "pending", "reason": "Awaiting review"},
  {"fg": "frontend", "status": "blocked", "blocked_by": "api"}
]

# Trigger completion actions
approval_matrix_on_complete(concept_path) {
  # 1. Update concept status
  # 2. Send celebration notification 🎉
  # 3. Queue for roadmap merge (Phase 13)
}
```

---

### MATRIX-05: Blocked Status Handling

**Goal:** Handle dependencies between focus group approvals

**Use Cases:**
1. **Explicit Block:** FG declares they're blocked waiting on another FG
2. **Cascading Block:** When one FG rejects, dependent FGs auto-block
3. **Auto-Unblock:** When blocker approves, blocked FG becomes pending

**Block Declaration:**
```bash
# User action: "We can't approve until API finalizes their changes"
/wgsd block oauth-integration --fg frontend --blocked-by api --reason "Waiting on API spec"
```

**Blocked Status Schema:**
```yaml
- focus_group: frontend
  status: blocked
  blocked_by: api
  blocked_reason: "Cannot review UI until API endpoints are finalized"
  blocked_date: "2026-02-23"
```

**Auto-Unblock Logic:**
```bash
# When API approves, auto-unblock Frontend
on_approval(concept, focus_group) {
  blocked_fgs = get_fgs_blocked_by(concept, focus_group)
  for fg in blocked_fgs:
    if fg.blocked_by == focus_group:
      fg.status = "pending"
      fg.blocked_by = null
      fg.blocked_reason = null
      notify_fg_unblocked(fg)
}
```

**Files to Create/Modify:**
- `workflows/lib/approval-system.md` — Blocking logic
- `workflows/matrix-block.md` — Block declaration workflow

**New Functions:**
```bash
# Block a focus group's approval
approval_matrix_block(concept_path, focus_group, blocked_by, reason)

# Unblock a focus group
approval_matrix_unblock(concept_path, focus_group)

# Get all blocked FGs and their blockers
approval_matrix_get_blocked(concept_path) → [
  {"fg": "frontend", "blocked_by": "api", "reason": "..."},
  {"fg": "docs", "blocked_by": "api", "reason": "..."}
]

# Check if FG can be unblocked (blocker approved)
approval_matrix_can_unblock(concept_path, focus_group) → boolean
```

---

### MATRIX-06: Approval Override (Admin)

**Goal:** Allow admins to force-approve stuck approvals for urgent situations

**Override Criteria:**
- User must have admin role
- Override must include reason
- Audit trail maintained
- Original focus group notified

**Override Workflow: `workflows/approval-override.md`**

```yaml
Entry Point:
  - "/wgsd override {concept} --fg {focus-group} --reason {reason}"
  - "/wgsd override oauth-integration --fg api --reason 'Critical security fix needed'"

Steps:
  1. Validate user is admin
  2. Confirm override with warning
  3. Update status to "approved" with override flag
  4. Record override metadata
  5. Notify original focus group
  6. Check for full approval
```

**Override Schema:**
```yaml
- focus_group: api
  status: approved
  overridden: true
  override_by: "@greg"
  override_date: "2026-02-23T14:30:00Z"
  override_reason: "Critical security fix, API team unavailable"
  # Original fields preserved for audit
  original_approver: null
  original_status: pending
```

**Admin Detection:**
```bash
# Check if user can perform admin override
approval_matrix_is_admin(user) → boolean

# Get admin list from config
approval_matrix_get_admins() → ["@greg", "@alice"]
```

**Notification:**
```markdown
⚠️ **Approval Override: oauth-integration**

The API focus group approval has been overridden by @greg.

**Reason:** Critical security fix, API team unavailable

This concept will now proceed to roadmap merge.
Please review when available: `/wgsd concept oauth-integration`

_Override timestamp: 2026-02-23 14:30 UTC_
```

---

## Deliverables

| File | Action | Description |
|------|--------|-------------|
| `workflows/lib/approval-system.md` | Enhance | Add matrix functions to existing file |
| `workflows/lib/canvas-templates.md` | Enhance | Add approval matrix widget |
| `workflows/matrix-approve.md` | Create | Per-FG approval workflow |
| `workflows/matrix-block.md` | Create | Block declaration workflow |
| `workflows/approval-override.md` | Create | Admin override workflow |
| `workflows/lib/impact-parser.md` | Enhance | Add SLA calculation |
| `templates/concept-directory/impact-matrix.md.tmpl` | Enhance | Add new approval fields |

---

## Execution Order

```
1. MATRIX-01 (Data Structure)
   └─ Foundation: Enhanced schema, parser functions
   
2. MATRIX-02 (Per-FG Approval)
   └─ Depends on: MATRIX-01
   └─ Core workflow: matrix-approve.md
   
3. MATRIX-04 (Completion Detection)
   └─ Depends on: MATRIX-01, MATRIX-02
   └─ Enables: Phase 13 trigger
   
4. MATRIX-03 (Canvas Widget)
   └─ Depends on: MATRIX-01
   └─ Visual layer, can parallelize

5. MATRIX-05 (Blocked Status)
   └─ Depends on: MATRIX-02
   └─ Advanced feature
   
6. MATRIX-06 (Admin Override)
   └─ Depends on: MATRIX-02, MATRIX-05
   └─ Emergency mechanism
```

---

## Integration Points

### With Phase 11 (Cross-Cutting Impacts)
- Approval matrix lives inside impact-matrix.md
- Uses impact-parser for reading/writing
- Notifications reuse impact-notifications infrastructure

### With Phase 13 (Roadmap Branch Architecture)
- MATRIX-04 completion triggers roadmap merge
- Fully approved concepts → `roadmap` branch

### With Phase 14 (Slack-Native Approvals)
- MATRIX-02 provides backend for Slack commands
- MATRIX-03 renders to Canvas for visibility

---

## Definition of Done

- [ ] Impact-matrix.md schema enhanced with approval tracking
- [ ] Per-focus-group approval via `/wgsd approve {concept} --fg {fg}`
- [ ] Canvas shows color-coded approval matrix widget
- [ ] Full approval auto-detected and triggers completion workflow
- [ ] Blocked status with dependencies working
- [ ] Admin override functional with audit trail
- [ ] All notifications sent appropriately
- [ ] Unit tests for completion detection logic

---

## Estimated Effort

| Requirement | Estimated | Notes |
|-------------|-----------|-------|
| MATRIX-01 | 30 min | Schema + parser enhancements |
| MATRIX-02 | 45 min | Core approval workflow |
| MATRIX-03 | 30 min | Canvas widget template |
| MATRIX-04 | 30 min | Detection logic |
| MATRIX-05 | 45 min | Blocking dependencies |
| MATRIX-06 | 30 min | Override workflow |
| **Total** | **~3.5 hours** | |

---

*Phase 12 detailed planning complete — ready for execution*
