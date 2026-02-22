# Phase 6: Community Integration - Execution Plan

**Date:** 2026-02-22
**Phase:** 6 - Community Integration (FINAL PHASE)
**Requirements:** 7 (COMMUNITY-01 through COMMUNITY-07)
**Estimated Duration:** 3-4 days
**Execution Method:** Wave-based sequential execution

---

## Requirements Mapping

| ID | Requirement | Wave | Deliverable |
|----|-------------|------|-------------|
| COMMUNITY-01 | Public feedback collection | Wave 1 | moderate-feedback.md |
| COMMUNITY-02 | Moderation triage workflow | Wave 1 | moderate-feedback.md |
| COMMUNITY-03 | Auto-invite contributors | Wave 2 | invite-contributor.md |
| COMMUNITY-04 | Access request system | Wave 2 | access-request.md |
| COMMUNITY-05 | Attribution in Canvas | Wave 3 | community-visibility.md |
| COMMUNITY-06 | Moderation commands | Wave 1+2 | multiple workflows |
| COMMUNITY-07 | Transparent visibility | Wave 3 | community-visibility.md |

---

## Wave 1: Feedback Collection & Moderation

**Goal:** Enable community feedback collection and moderation triage

### Deliverables

#### 1.1 workflows/moderate-feedback.md (~500 lines)
Feedback collection and triage workflow

**Sections:**
- Feedback Collection: Record feedback from community
- Triage Queue: List pending feedback items
- Move to Focus Group: Transfer feedback as concept
- Acknowledge: Post triage result to community

**Commands:**
```markdown
| Command | Action |
|---------|--------|
| collect [message-link] | Record as feedback |
| triage | Show triage queue |
| move-to-focus-group [id] [fg] | Move feedback to focus group |
| acknowledge [id] | Post acknowledgment to community |
```

**Key Features:**
- Emoji-based categorization (💡 feature, 🐛 bug, etc.)
- Smart focus group suggestion based on keywords
- Original message preservation with link
- Duplicate detection

### Wave 1 Success Criteria
- [ ] Feedback can be collected from community channel
- [ ] Triage queue shows pending feedback
- [ ] Move-to-focus-group creates concept with attribution
- [ ] Community receives acknowledgment

---

## Wave 2: Contributor Management

**Goal:** Manage contributor invitations and access requests

### Deliverables

#### 2.1 workflows/invite-contributor.md (~350 lines)
Contributor invitation workflow

**Sections:**
- Auto-Invite: Invite contributor when feedback becomes concept
- Manual Invite: Explicit invitation command
- Track Invitation: Record invitation status
- Handle Response: Process accept/decline

**Commands:**
```markdown
| Command | Action |
|---------|--------|
| invite-contributor @user [focus-group] | Send invitation |
| invitation-status @user | Check invitation status |
| pending-invites | List pending invitations |
```

#### 2.2 workflows/access-request.md (~350 lines)
Access request workflow

**Sections:**
- Request Access: Community member requests access
- Review Request: Post to focus group for review
- Approve Request: Process approval
- Deny Request: Process denial with reason

**Commands:**
```markdown
| Command | Action |
|---------|--------|
| request-access [focus-group] | Submit request |
| review-requests | Show pending requests |
| approve-access [id] | Approve request |
| deny-access [id] [reason] | Deny request |
```

### Wave 2 Success Criteria
- [ ] Contributors auto-invited when feedback becomes concept
- [ ] Manual invitation command works
- [ ] Access requests can be submitted
- [ ] Approval/denial process functional

---

## Wave 3: Visibility & Attribution

**Goal:** Provide transparent development visibility with attribution

### Deliverables

#### 3.1 workflows/community-visibility.md (~400 lines)
Community-facing visibility workflow

**Sections:**
- Generate Community Canvas: Create sanitized roadmap view
- Attribution Display: Show community contributors
- Progress Updates: Update community on development
- Sanitization Rules: What to show vs hide

**Commands:**
```markdown
| Command | Action |
|---------|--------|
| update-community-canvas | Refresh community roadmap |
| attribute @user [concept] [note] | Add attribution |
| community-status | Show engagement stats |
| publish-update [type] | Post progress update |
```

**Community Canvas Template:**
```markdown
# {Project} Development Roadmap

## What We're Building
[Sanitized list of active work]

## Community Contributions
[List of attributed community members]

## Recently Shipped
[Recent releases]

## How to Contribute
[Instructions for feedback]
```

### Wave 3 Success Criteria
- [ ] Community canvas shows sanitized roadmap
- [ ] Community contributors displayed with attribution
- [ ] Progress updates can be published
- [ ] Sensitive information properly hidden

---

## SKILL.md Updates

Add new routing entries:

```markdown
| "collect feedback", "record feedback" | workflows/moderate-feedback.md |
| "triage", "triage queue" | workflows/moderate-feedback.md |
| "move to focus group", "move-to-focus-group" | workflows/moderate-feedback.md |
| "invite contributor", "invite-contributor" | workflows/invite-contributor.md |
| "request access", "request-access" | workflows/access-request.md |
| "approve access", "approve-access" | workflows/access-request.md |
| "deny access", "deny-access" | workflows/access-request.md |
| "community status", "community-status" | workflows/community-visibility.md |
| "update community canvas" | workflows/community-visibility.md |
| "attribute", "attribution" | workflows/community-visibility.md |
```

---

## File Summary

### New Files (4)
| File | Lines | Purpose |
|------|-------|---------|
| workflows/moderate-feedback.md | ~500 | Feedback collection + triage |
| workflows/invite-contributor.md | ~350 | Contributor invitations |
| workflows/access-request.md | ~350 | Access request system |
| workflows/community-visibility.md | ~400 | Visibility + attribution |

### Total New Lines: ~1,600

---

## Execution Sequence

```
Wave 1: moderate-feedback.md (feedback + triage + move command)
        ↓
Wave 2: invite-contributor.md + access-request.md
        ↓
Wave 3: community-visibility.md (attribution + sanitized canvas)
        ↓
SKILL.md updates
        ↓
Verification: Test all 7 requirements
```

---

## Success Verification

After all waves complete, verify:

1. **COMMUNITY-01**: Feedback collected from community channel
2. **COMMUNITY-02**: Feedback can be triaged to focus groups
3. **COMMUNITY-03**: Contributors auto-invited when relevant
4. **COMMUNITY-04**: Access requests submitted and processed
5. **COMMUNITY-05**: Attribution displayed in focus group canvas
6. **COMMUNITY-06**: All moderation commands functional
7. **COMMUNITY-07**: Community has transparent development view

---

*Plan created: 2026-02-22*
