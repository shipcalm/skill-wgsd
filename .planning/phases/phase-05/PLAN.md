# Phase 5: Workflow Engine - Execution Plan

**Date:** 2026-02-22
**Phase:** 5 - Workflow Engine
**Requirements:** 9 (WORKFLOW-01 through WORKFLOW-08, INTEGRATE-07)
**Estimated Duration:** 4-5 days
**Execution Method:** Wave-based sequential execution

---

## Requirements Mapping

| ID | Requirement | Wave | Deliverable |
|----|-------------|------|-------------|
| WORKFLOW-01 | Two-track development | Wave 3 | concept-development.md + implementation-workflow.md |
| WORKFLOW-02 | Focus group approval process | Wave 1+2 | approval-system.md + promote-concept.md |
| WORKFLOW-03 | Promotion with owner assignment | Wave 2 | promote-concept.md (enhanced) |
| WORKFLOW-04 | Implementation prioritization | Wave 1+4 | priority-queue.md + implementation-status.md |
| WORKFLOW-05 | Enforce ~1 impl per focus group | Wave 2 | create-implementation.md (enhanced) |
| WORKFLOW-06 | Concept development workflow | Wave 3 | concept-development.md |
| WORKFLOW-07 | Implementation workflow | Wave 3 | implementation-workflow.md |
| WORKFLOW-08 | Status tracking and visualization | Wave 4 | implementation-status.md |
| INTEGRATE-07 | GitHub PR workflow integration | Wave 1 | github-pr.md |

---

## Wave 1: Core Libraries

**Goal:** Build foundational libraries for PR management, approval tracking, and prioritization

### Deliverables

#### 1.1 workflows/lib/github-pr.md (~400 lines)
GitHub PR integration for both development tracks

**Sections:**
- PR Creation: Create PRs with correct base branch and labels
- PR Detection: List open PRs for project
- PR Status: Check merge status
- PR Merge: Merge with squash option
- Track Routing: Auto-detect correct base branch

**Key Functions:**
```markdown
## Create Planning PR
Create PR for concept changes to focus group branch

## Create Implementation PR
Create PR for code changes to develop branch

## Detect PR Type
Analyze branch name to determine track

## Get PR Status
Check if PR is open, merged, or closed
```

#### 1.2 workflows/lib/approval-system.md (~350 lines)
Approval voting and tracking system

**Sections:**
- Vote Recording: Track approvals by user
- Vote Counting: Calculate approval status
- State Transitions: Manage concept state changes
- Notification: Alert team of pending approvals

**Approval States:**
```
pending: Awaiting minimum votes
approved: Minimum votes reached
rejected: Explicit rejection (optional)
expired: Voting window closed
```

#### 1.3 workflows/lib/priority-queue.md (~300 lines)
Implementation queue management

**Sections:**
- Queue Structure: Priority tiers and ordering
- Add to Queue: Insert with priority
- Queue Reordering: Adjust priorities
- Queue Display: Formatted output for dashboard

**Priority Tiers:**
```
critical: Blocking other work
high: Business priority
medium: Standard priority
low: Nice to have
```

### Wave 1 Success Criteria
- [ ] GitHub PR library can create PRs with correct base branch
- [ ] Approval system tracks votes and calculates status
- [ ] Priority queue maintains ordered list with tiers

---

## Wave 2: Enhanced Workflows

**Goal:** Enhance existing workflows with approval gates, owner assignment, and scaling enforcement

### Deliverables

#### 2.1 workflows/promote-concept.md (~500 lines) - NEW
Complete rewrite of concept promotion with approval gates

**Major Enhancements:**
- Check approval status before promotion
- Require minimum 2 approvals
- Prompt for owner assignment
- Validate owner is focus group member
- Add to implementation priority queue
- Trigger notifications

**New Flow:**
```
1. Validate concept exists and is mature
2. Check approval votes (min 2)
3. Prompt for owner assignment
4. Owner accepts assignment
5. Add to implementation queue
6. Update concept status to "Ready"
7. Notify focus group
```

#### 2.2 workflows/create-implementation.md Enhancement (~200 lines added)
Enhance existing workflow with scaling enforcement

**Enhancements:**
- Check focus group's active implementation count
- Warn if focus group already has active impl
- Require justification for override
- Log override decisions
- Update priority queue

**New Sections:**
```markdown
## Scaling Check
Verify natural scaling (~1 impl per focus group)

## Override Handling
Process override with justification
```

### Wave 2 Success Criteria
- [ ] Concept cannot promote without minimum approvals
- [ ] Owner assignment required before implementation
- [ ] Scaling warning appears for >1 impl per focus group
- [ ] Override requires explicit justification

---

## Wave 3: Development Tracks

**Goal:** Implement both development tracks with proper PR integration

### Deliverables

#### 3.1 workflows/concept-development.md (~450 lines) - NEW
Planning track: Concepts evolve through planning PRs

**Sections:**
- Create Planning Branch: Branch from focus group base
- Edit Concept: Update concept files
- Create Planning PR: PR to focus group branch
- Review Process: Focus group reviews planning changes
- Merge and Sync: Merge triggers canvas sync

**Key Flow:**
```
1. User: "I want to update the auth-v2 concept"
2. Create branch: concept/auth-v2-update from focus-groups/security
3. Edit .planning/focus-groups/security/concepts/auth-v2.md
4. Create PR: concept/auth-v2-update → focus-groups/security
5. Focus group reviews
6. Merge PR
7. Canvas syncs automatically
```

#### 3.2 workflows/implementation-workflow.md (~450 lines) - NEW
Implementation track: Code PRs to develop

**Sections:**
- Implementation Start: Verify concept is ready
- Code Development: Traditional GSD workflow
- Create Code PR: PR to develop branch
- Code Review: Standard review process
- Merge to Develop: Complete implementation
- Cleanup: Archive implementation artifacts

**Key Flow:**
```
1. Implementation starts from ready concept
2. Owner works in implementations/{name}/ worktree
3. Create PR: implementations/{name} → develop
4. Code review by team
5. Merge to develop (trunk)
6. Archive implementation channel
7. Mark concept as implemented
```

### Wave 3 Success Criteria
- [ ] Planning changes go to focus group branches only
- [ ] Code changes go to develop branch only
- [ ] PRs are created with correct labels and reviewers
- [ ] Canvas syncs on planning PR merge

---

## Wave 4: Status and Dashboard

**Goal:** Implement status tracking and dashboard visualization

### Deliverables

#### 4.1 workflows/implementation-status.md (~400 lines) - NEW
Progress tracking and reporting

**Sections:**
- Status Query: Get implementation status
- Progress Update: Owner updates progress
- Daily Summary: Generate daily status report
- Blocker Management: Track and escalate blockers
- Completion: Mark implementation complete

**Status Fields:**
```yaml
implementation: auth-v2
focus_group: security
owner: @alice
status: implementing
progress: 60%
phase: "Phase 2 of 3"
days_active: 2
blockers: []
last_update: "2026-02-22T10:30:00Z"
```

#### 4.2 Canvas Template Updates
Enhance canvas-templates.md with dashboard views

**New Templates:**
```markdown
## Implementation Queue Dashboard
Renders priority queue with status indicators

## Active Implementations Grid
Shows all active implementations with progress bars

## Focus Group Overview
Shows implementation count per focus group
```

### Wave 4 Success Criteria
- [ ] Implementation status can be queried
- [ ] Progress updates reflect in dashboard
- [ ] Daily summary available
- [ ] Blockers are tracked and visible

---

## SKILL.md Updates

Add new routing entries:

```markdown
| "promote concept", "approve concept" | workflows/promote-concept.md |
| "concept pr", "planning pr" | workflows/concept-development.md |
| "implementation pr", "code pr" | workflows/implementation-workflow.md |
| "implementation status", "impl status" | workflows/implementation-status.md |
| "approve", "vote" | workflows/promote-concept.md |
| "claim", "assign owner" | workflows/promote-concept.md |
```

---

## File Summary

### New Files (8)
| File | Lines | Purpose |
|------|-------|---------|
| workflows/lib/github-pr.md | ~400 | GitHub PR integration |
| workflows/lib/approval-system.md | ~350 | Voting/approval tracking |
| workflows/lib/priority-queue.md | ~300 | Implementation queue |
| workflows/promote-concept.md | ~500 | Complete rewrite with gates |
| workflows/concept-development.md | ~450 | Planning track workflow |
| workflows/implementation-workflow.md | ~450 | Implementation track workflow |
| workflows/implementation-status.md | ~400 | Progress tracking |
| .planning/phases/phase-05/VERIFICATION.md | ~200 | Verification results |

### Enhanced Files (2)
| File | Changes |
|------|---------|
| workflows/create-implementation.md | +200 lines: scaling enforcement |
| workflows/lib/canvas-templates.md | +150 lines: dashboard templates |

### Total New Lines: ~3,400

---

## Execution Sequence

```
Wave 1: github-pr.md → approval-system.md → priority-queue.md
        ↓
Wave 2: promote-concept.md → create-implementation.md (enhance)
        ↓
Wave 3: concept-development.md → implementation-workflow.md
        ↓
Wave 4: implementation-status.md → canvas-templates.md (enhance)
        ↓
Verification: Test all 9 requirements
```

---

## Success Verification

After all waves complete, verify:

1. **WORKFLOW-01**: Two tracks are clearly separated
2. **WORKFLOW-02**: Approval gates prevent premature promotion
3. **WORKFLOW-03**: Owner assignment is mandatory
4. **WORKFLOW-04**: Priority queue is visible in dashboard
5. **WORKFLOW-05**: Scaling warnings appear appropriately
6. **WORKFLOW-06**: Planning PRs go to focus group branches
7. **WORKFLOW-07**: Code PRs go to develop
8. **WORKFLOW-08**: Status is trackable and visible
9. **INTEGRATE-07**: GitHub PR integration works end-to-end

---

*Plan created: 2026-02-22*
