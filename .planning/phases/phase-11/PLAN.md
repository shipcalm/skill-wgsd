# Phase 11: Cross-Cutting Impact System

**Status:** ✅ COMPLETE  
**Priority:** P0 - Critical  
**Duration:** ~2-3 hours  
**Dependencies:** Phase 10 (Complete)  
**Planned:** 2026-02-23  
**Completed:** 2026-02-23

---

## Objective

Enable concepts to declare and track impact across multiple focus groups, supporting the matrix-based approval workflow introduced in Phase 12.

**Core Value:** A single concept like "OAuth Integration" can declare impacts on Security, API, and Frontend focus groups, triggering notifications and creating approval requirements for each.

---

## Research Findings

### Cross-Cutting Concern Patterns
- **First-Class Citizens:** Cross-cutting concerns should be linked to features and traced like functional requirements (ScienceDirect)
- **Multi-Stakeholder Evaluation:** Stakeholders evaluate against their functional criteria (financial, legal, operational, compliance, strategic)
- **Notification-Driven:** AI/automation summarizes incoming feedback for coordinators

### Applied to WGSD
- Each focus group impact becomes an approval requirement
- Notifications route to affected focus group channels
- Impact changes trigger re-notification for updated review

---

## Requirements

| ID | Requirement | Status | Effort | Files |
|----|-------------|--------|--------|-------|
| IMPACT-01 | Impact matrix file format | ✅ Complete | S | templates/concept-directory/impact-matrix.md.tmpl, workflows/lib/impact-parser.md |
| IMPACT-02 | Impact declaration workflow | ✅ Complete | M | workflows/declare-impact.md |
| IMPACT-03 | Automatic focus group notifications | ✅ Complete | M | workflows/lib/impact-notifications.md |
| IMPACT-04 | Impact change tracking | ✅ Complete | M | workflows/update-impact.md |

---

## Execution Plans

### IMPACT-01: Impact Matrix File Format

**Goal:** Formalize the YAML frontmatter schema and make the template machine-parseable.

**Current State:**
- `templates/concept-directory/impact-matrix.md.tmpl` exists with basic structure
- Uses tables for human readability
- Missing: strict YAML schema, parsing utilities

**Tasks:**
1. **Enhance YAML frontmatter schema** (~15 min)
   - Define strict schema with validation rules
   - Add `impacts[]` array in frontmatter (structured, machine-parseable)
   - Keep Markdown tables for human-readable view
   
2. **Create impact schema spec** (~15 min)
   - Document valid priority values: P0, P1, P2, P3
   - Document impact types: breaking-change, api-change, documentation, integration, testing, behavior
   - Document status values: pending, approved, rejected, blocked

3. **Add parsing helper to lib/** (~20 min)
   - Create `workflows/lib/impact-parser.md`
   - Functions to read/write impact-matrix.md
   - Extract impacts array from frontmatter
   - Validate against schema

**Updated Template Structure:**
```yaml
---
concept: {{CONCEPT_SLUG}}
created: {{DATE}}
updated: {{DATE}}
impacts:
  - focus_group: api
    priority: P0
    type: breaking-change
    description: "Auth endpoint signature changes"
    status: pending
    approver: null
    approved_date: null
  - focus_group: security
    priority: P1
    type: integration
    description: "New token validation flow"
    status: pending
    approver: null
    approved_date: null
---
```

**Files Modified:**
- `templates/concept-directory/impact-matrix.md.tmpl` (enhance)
- `workflows/lib/impact-parser.md` (NEW)

**Acceptance Criteria:**
- [ ] YAML frontmatter has strict `impacts[]` array schema
- [ ] Schema supports priority, type, description, status, approver fields
- [ ] Parser can extract impacts from any impact-matrix.md
- [ ] Parser validates against schema

---

### IMPACT-02: Impact Declaration Workflow

**Goal:** Interactive workflow to declare which focus groups a concept impacts.

**Tasks:**
1. **Create declare-impact.md workflow** (~30 min)
   - Prompt for focus group selection (from known focus groups)
   - Prompt for priority per focus group
   - Prompt for impact type and description
   - Generate/update impact-matrix.md

2. **Focus group discovery** (~15 min)
   - List available focus groups from `.planning/focus-groups/`
   - Present as interactive menu
   - Support batch declaration (multiple FGs at once)

3. **Integration with create-concept** (~15 min)
   - Option to declare impacts during concept creation
   - Or run separately: `/wgsd declare-impact {concept}`

**Workflow Flow:**
```
1. User: /wgsd declare-impact oauth-integration
2. System: Lists available focus groups: [api, security, frontend, billing]
3. User: Selects: api, security
4. System: For 'api' - What priority? What type? Description?
5. User: P0, breaking-change, "Auth endpoint changes"
6. System: For 'security' - What priority? What type? Description?
7. User: P1, integration, "New token flow"
8. System: Generates impact-matrix.md, triggers notifications
```

**Files Created:**
- `workflows/declare-impact.md` (NEW)

**Acceptance Criteria:**
- [ ] Workflow lists available focus groups
- [ ] User can select multiple focus groups
- [ ] Each impact has priority, type, description
- [ ] impact-matrix.md generated/updated
- [ ] Backward compatible with manual editing

---

### IMPACT-03: Automatic Focus Group Notifications

**Goal:** Notify affected focus groups when a concept declares impact on them.

**Tasks:**
1. **Create notification library** (~30 min)
   - `workflows/lib/impact-notifications.md`
   - Function to send impact notification to channel
   - Format message with concept summary, priority, impact description
   - Include action prompts (review, approve, discuss)

2. **Channel resolution** (~15 min)
   - Map focus group name → Slack channel
   - Use existing `workflows/lib/channel-registry.md`
   - Handle missing channels gracefully

3. **Notification message format** (~15 min)
   - Rich formatting with priority badge
   - Link to concept (Canvas or file)
   - Action buttons/commands for approval

**Notification Template:**
```
🔔 *Impact Declaration: oauth-integration*

A new concept declares impact on *#mvn-security*:

*Priority:* 🔴 P0 - Critical
*Type:* Breaking Change
*Impact:* Auth endpoint signature changes

📋 *Next Steps:*
• Review concept: `/wgsd concept oauth-integration`
• Approve: `/wgsd approve oauth-integration`
• Discuss: Thread below or in #mvn-dev

_From: @author | Focus Group: api_
```

**Files Created:**
- `workflows/lib/impact-notifications.md` (NEW)

**Files Modified:**
- `workflows/declare-impact.md` (integrate notifications)

**Acceptance Criteria:**
- [ ] Notification sent to each impacted focus group channel
- [ ] Message includes priority, type, description
- [ ] Message includes approval action prompt
- [ ] Channel resolution handles edge cases

---

### IMPACT-04: Impact Change Tracking

**Goal:** Track changes to impact declarations and re-notify when updated.

**Tasks:**
1. **Create update-impact workflow** (~25 min)
   - `/wgsd update-impact {concept}`
   - Detect existing impacts
   - Allow add/remove/modify
   - Track changes in change log

2. **Change detection** (~15 min)
   - Compare old vs new impacts
   - Identify: added FGs, removed FGs, priority changes
   - Generate change summary

3. **Re-notification on change** (~15 min)
   - Notify newly impacted focus groups (full notification)
   - Notify existing FGs of priority/scope changes (update notification)
   - Notify removed FGs (impact removed notification)

4. **Change log in impact-matrix.md** (~10 min)
   - Append changes to Change Log table
   - Include date, change description, author

**Change Notification Types:**
- **NEW_IMPACT:** Full notification (same as IMPACT-03)
- **UPDATED_IMPACT:** "⚠️ Impact Updated: Priority changed P1→P0"
- **REMOVED_IMPACT:** "✅ Impact Removed: oauth-integration no longer impacts security"

**Files Created:**
- `workflows/update-impact.md` (NEW)

**Files Modified:**
- `workflows/lib/impact-notifications.md` (add change notifications)

**Acceptance Criteria:**
- [ ] Changes to impacts tracked in change log
- [ ] New impacts trigger full notification
- [ ] Priority/scope changes trigger update notification
- [ ] Removed impacts trigger removal notification
- [ ] Canvas updated with impact changes

---

## File Summary

### New Files
| File | Purpose |
|------|---------|
| `workflows/declare-impact.md` | Interactive impact declaration workflow |
| `workflows/update-impact.md` | Impact modification workflow |
| `workflows/lib/impact-parser.md` | Parse/validate impact-matrix.md |
| `workflows/lib/impact-notifications.md` | Slack notifications for impacts |

### Modified Files
| File | Changes |
|------|---------|
| `templates/concept-directory/impact-matrix.md.tmpl` | Enhanced YAML schema |
| `workflows/create-concept.md` | Optional impact declaration integration |
| `SKILL.md` | Add declare-impact, update-impact to routing |

---

## Integration Points

### Phase 10 (Complete)
- Uses concept directory structure from Phase 10
- impact-matrix.md is part of concept directory template

### Phase 12 (Next)
- Impact declarations become approval requirements
- `impacts[].status` drives approval matrix
- Phase 12 builds approval UI on top of impact data

### Slack Integration
- Uses existing `workflows/lib/slack-api.md` for channel operations
- Uses existing `workflows/lib/channel-registry.md` for FG→channel mapping

---

## Definition of Done

- [ ] impact-matrix.md has strict YAML schema with `impacts[]` array
- [ ] `declare-impact` workflow interactively declares impacts
- [ ] Focus groups receive notifications on impact declaration
- [ ] `update-impact` workflow modifies existing impacts
- [ ] Impact changes trigger appropriate re-notifications
- [ ] Canvas displays impact matrix
- [ ] All 4 requirements pass acceptance criteria

---

## Estimated Timeline

| Task | Effort |
|------|--------|
| IMPACT-01: Format spec + parser | 50 min |
| IMPACT-02: Declaration workflow | 60 min |
| IMPACT-03: Notifications | 60 min |
| IMPACT-04: Change tracking | 65 min |
| **Total** | **~4 hours** |

*Note: Slightly higher than original 2h estimate due to notification system complexity*

---

## Execution Order

1. **IMPACT-01** first (format foundation)
2. **IMPACT-03** second (notifications needed by IMPACT-02)
3. **IMPACT-02** third (declaration workflow)
4. **IMPACT-04** last (builds on all above)

---

*Phase 11 planned and ready for execution — 2026-02-23*
