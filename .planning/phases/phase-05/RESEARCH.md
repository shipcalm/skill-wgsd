# Phase 5: Workflow Engine - Research

**Date:** 2026-02-22
**Phase:** 5 - Workflow Engine
**Requirements:** WORKFLOW-01 through WORKFLOW-08, INTEGRATE-07 (9 total)

---

## Domain Research

### Two-Track Development Model (WORKFLOW-01)

The WGSD two-track model separates concerns:

**Track 1: Collaborative Planning**
- Focus groups branches (`focus-groups/{name}`)
- Planning-only changes (docs, concepts, roadmaps)
- PRs merge to focus group branches, not develop
- Slower cadence, more discussion, social evolution
- No code changes allowed on planning branches

**Track 2: Trunk-Based Implementation**
- Implementation branches (`implementations/{name}`)
- Code changes only, minimal docs
- PRs merge directly to develop (trunk)
- Fast cadence (1-3 days), single owner
- Traditional GSD workflow applies

**Key Separation:**
- Planning PRs: `.planning/` changes → focus-groups/{fg} branch
- Code PRs: Source code changes → develop branch
- Never mix planning and code in same PR

### Focus Group Approval Process (WORKFLOW-02)

**Concept States:**
```
📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready for Implementation
```

**Approval Gates:**
1. **Draft → Exploring**: No approval needed (author decision)
2. **Exploring → Mature**: Focus group consensus (discussion-based)
3. **Mature → Ready**: Focus group approval vote
   - Minimum 2 approvals from focus group members
   - Approval via emoji reaction (👍 or ✅) or `/wgsd approve-concept`
   - 24-hour voting window (can be expedited)
4. **Ready → Implementation**: Owner assignment required

**Approval Tracking:**
```markdown
## Approval Status
**Current State:** 🔍 Exploring
**Votes to Advance:** 0/2
**Voters:** (none yet)
```

### Owner Assignment (WORKFLOW-03)

**Implementation Ownership:**
- Concept promotion requires explicit owner assignment
- Owner must be a focus group participant
- Owner commits to 1-3 day implementation timeline
- Owner responsible for daily status updates

**Assignment Flow:**
1. Concept reaches "Ready for Implementation"
2. `/wgsd assign-owner [concept] @user`
3. Owner accepts assignment
4. Implementation created with owner recorded

### Implementation Prioritization (WORKFLOW-04)

**Priority Factors:**
1. **Business Impact:** High/Medium/Low
2. **Technical Readiness:** Dependencies resolved?
3. **Focus Group Priority:** Voted priority
4. **Queue Position:** FIFO within priority tier

**Queue Visualization:**
```
## Implementation Queue

### 🔴 High Priority
1. auth-v2 (Security FG) - Owner: @alice - Ready
2. sso-integration (Security FG) - Awaiting owner

### 🟡 Medium Priority  
3. onboarding-wizard (Onboarding FG) - Owner: @bob - Ready

### 🟢 Low Priority
4. docs-refactor (Docs FG) - No owner yet
```

### Natural Scaling (WORKFLOW-05)

**Rule:** ~1 implementation per focus group

**Rationale:**
- Focus groups represent domain expertise
- 1 impl/FG prevents context switching
- Team naturally scales implementations
- Override available with explicit justification

**Enforcement:**
```bash
# Count active implementations per focus group
SECURITY_IMPLS=$(find .planning/active-implementations -name "concept-source.md" -exec grep -l "Security" {} \;)
if [ $(echo "$SECURITY_IMPLS" | wc -l) -gt 1 ]; then
    echo "⚠️ Security focus group already has active implementation"
    echo "Continue anyway? (requires justification)"
fi
```

### GitHub PR Integration (INTEGRATE-07)

**PR Types:**

**Planning PRs (Track 1):**
```yaml
Base: focus-groups/{focus-group}
Head: concept/{concept-name}
Title: "feat({fg}): {concept-name} concept update"
Labels: planning, {focus-group}
Reviewers: Focus group members
```

**Implementation PRs (Track 2):**
```yaml
Base: develop
Head: implementations/{concept-name}
Title: "feat: {concept-name} implementation"
Labels: implementation, {focus-group}
Reviewers: Code reviewers
```

**PR Workflow:**
1. Create PR with appropriate base branch
2. Request reviews from relevant team
3. Merge after approval
4. Trigger canvas sync on merge

### Status Tracking (WORKFLOW-08)

**Implementation Status Model:**
```yaml
Status:
  - initializing: Just created
  - planning: Requirements/roadmap phase
  - implementing: Active coding
  - review: PR open, awaiting merge
  - merging: Merging to develop
  - complete: Merged and cleaned up
  - blocked: Waiting on dependency

Progress:
  percentage: 0-100
  phase: "Phase X of Y"
  blockers: []
  last_update: timestamp
```

**Dashboard Format:**
```
| Implementation | Focus Group | Owner | Status | Progress | Days |
|----------------|-------------|-------|--------|----------|------|
| auth-v2 | Security | @alice | implementing | 60% | 2 |
| sso | Security | @bob | planning | 10% | 1 |
```

---

## Design Decisions

### Decision 1: Approval via Reaction vs Command
**Choice:** Support both
- Emoji reaction (👍) for quick approval
- `/wgsd approve [concept]` for explicit approval with comment
- Both are recorded in concept file

### Decision 2: Owner Self-Assignment
**Choice:** Allow self-assignment with confirmation
- `/wgsd claim [concept]` for self-assignment
- `/wgsd assign [concept] @user` for delegation
- Owner must confirm with 24h acceptance window

### Decision 3: Scaling Enforcement Level
**Choice:** Soft enforcement with override
- Warn when >1 impl per focus group
- Require justification string to override
- Log override decisions for review

### Decision 4: PR Automation Level
**Choice:** Semi-automated
- Auto-create PR with correct base/labels
- Manual review request (team varies)
- Auto-update canvas on merge (webhook or poll)

### Decision 5: Progress Calculation
**Choice:** Phase-based + manual override
- Default: (current_phase / total_phases) * 100
- Manual: Owner can set exact percentage
- Both displayed in dashboard

---

## Implementation Approach

### Wave Structure

**Wave 1: Core Libraries**
- `workflows/lib/github-pr.md` - PR creation and management
- `workflows/lib/approval-system.md` - Voting and approval tracking
- `workflows/lib/priority-queue.md` - Queue management

**Wave 2: Enhanced Workflows**
- `workflows/promote-concept.md` - Enhanced with approval gates
- `workflows/create-implementation.md` - Enhanced with scaling enforcement

**Wave 3: Development Tracks**
- `workflows/concept-development.md` - Planning PR workflow (Track 1)
- `workflows/implementation-workflow.md` - Code PR workflow (Track 2)

**Wave 4: Status and Dashboard**
- `workflows/implementation-status.md` - Progress tracking
- Canvas template updates for queue visualization

---

## Dependencies

### Existing Assets to Enhance
- `workflows/promote-concept.md` - Base exists, needs approval gates
- `workflows/create-concept.md` - Already creates concepts
- `workflows/create-implementation.md` - Already exists, needs scaling check
- `workflows/lib/canvas-templates.md` - Needs dashboard templates

### External Dependencies
- GitHub API for PR operations
- Slack reactions API for approval votes
- Canvas sync system (Phase 4) for dashboard updates

---

## Testing Strategy

### Unit Tests
1. Approval counting logic
2. Priority queue sorting
3. Scaling violation detection
4. PR base branch selection

### Integration Tests
1. Full concept promotion flow with approval
2. Implementation creation with scaling warning
3. PR creation and merge detection
4. Dashboard canvas update after state change

### End-to-End Tests
1. Concept → Approval → Implementation → Merge flow
2. Multi-focus-group parallel development
3. Scaling override with justification

---

*Research complete: 2026-02-22*
