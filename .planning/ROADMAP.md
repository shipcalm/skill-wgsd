# WGSD v2.2 Roadmap

**Milestone:** Matrix-Based Social Development Architecture  
**Created:** 2026-02-23  
**Total Phases:** 7 (Phases 10-16, continuing from v2.1)
**Total Requirements:** 31
**Status:** 🔵 **PLANNING COMPLETE**

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WGSD v2.2 ROADMAP                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    FOUNDATION LAYER (P0)                            │   │
│   │  Phase 10         Phase 11          Phase 12                        │   │
│   │  ┌─────────┐     ┌─────────┐       ┌─────────┐                      │   │
│   │  │ Concept │     │ Cross-  │       │ Matrix  │                      │   │
│   │  │  Dirs   │────►│ Cutting │──────►│Approval │                      │   │
│   │  └─────────┘     │ Impact  │       └────┬────┘                      │   │
│   │                  └─────────┘            │                           │   │
│   └─────────────────────────────────────────┼───────────────────────────┘   │
│                                             │                               │
│   ┌─────────────────────────────────────────┼───────────────────────────┐   │
│   │                 INTEGRATION LAYER (P1)  │                           │   │
│   │                                         ▼                           │   │
│   │  Phase 14        Phase 13           Phase 15                        │   │
│   │  ┌─────────┐    ┌─────────┐        ┌─────────┐                      │   │
│   │  │ Slack   │───►│ Roadmap │───────►│ Canvas  │                      │   │
│   │  │Approval │    │ Branch  │        │Dev Info │                      │   │
│   │  └─────────┘    └─────────┘        └─────────┘                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    OPERATIONS LAYER (P2)                            │   │
│   │  Phase 16 (Independent - can run parallel)                          │   │
│   │  ┌─────────────┐                                                    │   │
│   │  │  Emergency  │                                                    │   │
│   │  │   Hotfix    │                                                    │   │
│   │  └─────────────┘                                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 10: Concept Directory Architecture

**Goal:** Transform concepts from single files to full directories with multiple artifacts

**Priority:** P0 - Critical  
**Duration:** ~2-3 hours  
**Dependencies:** None (foundation phase)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| CONCEPT-DIR-01 | Create concept as directory | M |
| CONCEPT-DIR-02 | Concept directory structure template | M |
| CONCEPT-DIR-03 | Independent concept branches | M |
| CONCEPT-DIR-04 | Concept worktree support | L |
| CONCEPT-DIR-05 | Multi-artifact concept sync | M |

### Deliverables

1. **workflows/create-concept.md** — Directory-based concept creation
2. **templates/concept-directory/** — Standard directory structure
3. **workflows/lib/branch-ops.md** — Concept branch operations
4. **workflows/lib/git-ops.md** — Worktree support

### Success Criteria

- [ ] New concepts create as directories
- [ ] Directory includes CONCEPT.md + impact-matrix.md
- [ ] Independent branch created per concept
- [ ] Optional worktree setup working
- [ ] Canvas displays multi-artifact concepts

---

## Phase 11: Cross-Cutting Impact System

**Goal:** Enable concepts to declare and track impact across multiple focus groups

**Priority:** P0 - Critical  
**Duration:** ~2 hours  
**Dependencies:** Phase 10 (concept directories must exist)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| IMPACT-01 | Impact matrix file format | S |
| IMPACT-02 | Impact declaration workflow | M |
| IMPACT-03 | Automatic focus group notifications | M |
| IMPACT-04 | Impact change tracking | M |

### Deliverables

1. **templates/concept-directory/impact-matrix.md** — Impact format specification
2. **workflows/declare-impact.md** — Interactive impact declaration
3. **workflows/update-impact.md** — Impact modification workflow
4. **workflows/lib/slack-api.md** — Impact notification integration

### Success Criteria

- [ ] Concepts can declare multiple focus group impacts
- [ ] Each impact has priority and description
- [ ] Focus groups notified when impacted
- [ ] Impact changes trigger notifications
- [ ] Canvas shows impact matrix

---

## Phase 12: Matrix-Based Approval

**Goal:** Transform approval from single owner to multi-stakeholder matrix

**Priority:** P0 - Critical  
**Duration:** ~3 hours  
**Dependencies:** Phases 10 & 11 (need concept structure and impact declarations)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| MATRIX-01 | Approval matrix data structure | M |
| MATRIX-02 | Per-focus-group approval | M |
| MATRIX-03 | Approval matrix Canvas widget | L |
| MATRIX-04 | Approval completion detection | M |
| MATRIX-05 | Blocked status handling | M |
| MATRIX-06 | Approval override (admin) | M |

### Deliverables

1. **workflows/lib/approval-system.md** — Matrix-based approval logic
2. **workflows/lib/canvas-templates.md** — Approval matrix widget
3. **workflows/approval-override.md** — Admin override workflow

### Success Criteria

- [ ] Approval matrix tracks per-FG status
- [ ] Independent approvals per focus group
- [ ] Canvas shows matrix with color coding
- [ ] System detects fully approved state
- [ ] Blocked status with dependencies working
- [ ] Admin override functional

---

## Phase 13: Roadmap Branch Architecture

**Goal:** Establish roadmap branch as source of truth for approved concepts

**Priority:** P1 - High  
**Duration:** ~2-3 hours  
**Dependencies:** Phase 12 (need approval completion to trigger merge)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| ROADMAP-01 | Roadmap branch creation | S |
| ROADMAP-02 | Automatic concept → roadmap merge | L |
| ROADMAP-03 | Implementation branches from roadmap | M |
| ROADMAP-04 | Roadmap Canvas view | M |
| ROADMAP-05 | Roadmap sync to develop | M |

### Deliverables

1. **workflows/init.md** — Roadmap branch creation on project init
2. **workflows/promote-concept.md** — Concept to roadmap merge
3. **workflows/create-implementation.md** — Branch from roadmap
4. **workflows/roadmap-sync.md** — Roadmap → develop sync
5. **workflows/lib/canvas-templates.md** — Roadmap view

### Success Criteria

- [ ] Roadmap branch created and protected
- [ ] Approved concepts merge to roadmap automatically
- [ ] Implementations branch from roadmap
- [ ] Canvas shows roadmap contents
- [ ] Sync mechanism keeps develop updated

---

## Phase 14: Slack-Native Approval Workflow

**Goal:** Conversational approvals in Slack channels

**Priority:** P1 - High  
**Duration:** ~2-3 hours  
**Dependencies:** Phase 12 (need approval matrix structure)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| SLACK-APPROVE-01 | Conversational approval prompts | M |
| SLACK-APPROVE-02 | Approval via slash command | M |
| SLACK-APPROVE-03 | Discussion thread for approval | M |
| SLACK-APPROVE-04 | Approval status summary | M |
| SLACK-APPROVE-05 | Rejection with feedback | M |

### Deliverables

1. **workflows/conversational-approve.md** — Slack approval commands
2. **workflows/concept-status.md** — Status query workflow
3. **workflows/lib/slack-api.md** — Approval message patterns
4. **workflows/lib/approval-system.md** — Rejection handling

### Success Criteria

- [ ] Approval prompts appear in FG channels
- [ ] `/wgsd approve {concept}` working
- [ ] Discussion threads linked
- [ ] Status summaries accurate
- [ ] Rejection requires and stores feedback

---

## Phase 15: Canvas Developer Info

**Goal:** Show developers the git information they need

**Priority:** P2 - Medium  
**Duration:** ~1-2 hours  
**Dependencies:** Phase 13 (roadmap branch structure needed)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| CANVAS-DEV-01 | Display current branch | S |
| CANVAS-DEV-02 | Display worktree path | S |
| CANVAS-DEV-03 | Git status quick reference | M |

### Deliverables

1. **workflows/lib/canvas-templates.md** — Developer info widgets
2. **workflows/canvas-sync.md** — Git status integration

### Success Criteria

- [ ] Canvas shows branch name
- [ ] Worktree path visible when applicable
- [ ] Basic git status in Canvas
- [ ] Last commit summary displayed

---

## Phase 16: Emergency Hotfix Bypass

**Goal:** Allow urgent fixes to bypass approval matrix

**Priority:** P2 - Medium  
**Duration:** ~1-2 hours  
**Dependencies:** Phase 10 (concept structure for optional retroactive tracking)

### Requirements

| ID | Requirement | Effort |
|----|-------------|--------|
| HOTFIX-01 | Emergency hotfix workflow | M |
| HOTFIX-02 | Hotfix completion and merge | M |
| HOTFIX-03 | Post-hotfix concept creation | S |

### Deliverables

1. **workflows/emergency-hotfix.md** — Complete hotfix workflow

### Success Criteria

- [ ] Hotfix starts from develop directly
- [ ] No approval required
- [ ] Fast merge path to develop/main
- [ ] Optional concept creation afterward
- [ ] Audit trail maintained

---

## Execution Plan

### Recommended Execution Order

```
Week 1: Foundation (Critical Path)
├── Day 1-2: Phase 10 - Concept Directory Architecture
├── Day 2-3: Phase 11 - Cross-Cutting Impact System
└── Day 3-4: Phase 12 - Matrix-Based Approval

Week 2: Integration  
├── Day 1-2: Phase 14 - Slack-Native Approval (parallel with 13)
├── Day 1-2: Phase 13 - Roadmap Branch Architecture
├── Day 2-3: Phase 15 - Canvas Developer Info
└── Day 3: Phase 16 - Emergency Hotfix (can run parallel)
```

### Parallel Execution Opportunities

| Phase | Can Run Parallel With |
|-------|----------------------|
| 10 | 16 (after 10 core complete) |
| 11 | 16 |
| 13 | 14 (same dependencies) |
| 15 | 16 |

### Critical Path

```
Phase 10 → Phase 11 → Phase 12 → Phase 13/14 → Phase 15
```

Phase 16 is independent and can be executed at any time after Phase 10.

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Concept directory migration complexity | Medium | High | Backward compat for single-file concepts |
| Matrix approval UI complexity | Medium | Medium | Start with text-based, enhance later |
| Roadmap branch conflicts | Low | Medium | Clear merge policies, automated checks |
| Slack API rate limits | Low | Low | Existing rate limiting in place |
| Scope creep | Medium | High | Strict phase boundaries, defer to v2.3 |

---

## Verification Strategy

### Phase 10 Verification
- Create new concept, verify directory structure
- Verify branch creation and worktree
- Test Canvas sync with multi-artifact concept

### Phase 11 Verification  
- Declare concept with 3+ focus group impacts
- Verify all FG channels receive notifications
- Update impact, verify change notifications

### Phase 12 Verification
- Approve concept from different FG perspectives
- Test partial approval visibility
- Verify completion detection triggers
- Test admin override

### Phase 13 Verification
- Verify roadmap branch creation
- Complete approval, verify auto-merge to roadmap
- Create implementation, verify branches from roadmap
- Test sync to develop

### Phase 14 Verification
- Test approval via slash command
- Verify discussion threads
- Test rejection flow with feedback
- Query status, verify accuracy

### Phase 15 Verification
- View Canvas, verify branch display
- Set up worktree, verify path shown
- Check git status integration

### Phase 16 Verification
- Start emergency hotfix
- Complete and merge without approval
- Test optional concept creation

---

## Summary

| Phase | Name | Requirements | Effort | Priority | Dependencies |
|-------|------|--------------|--------|----------|--------------|
| 10 | Concept Directory Architecture | 5 | ~2-3h | P0 | None |
| 11 | Cross-Cutting Impact System | 4 | ~2h | P0 | Phase 10 |
| 12 | Matrix-Based Approval | 6 | ~3h | P0 | Phases 10, 11 |
| 13 | Roadmap Branch Architecture | 5 | ~2-3h | P1 | Phase 12 |
| 14 | Slack-Native Approval Workflow | 5 | ~2-3h | P1 | Phase 12 |
| 15 | Canvas Developer Info | 3 | ~1-2h | P2 | Phase 13 |
| 16 | Emergency Hotfix Bypass | 3 | ~1-2h | P2 | Phase 10 |
| **Total** | | **31** | **~15-18h** | | |

---

## Milestone Success Metrics

| Metric | Target |
|--------|--------|
| Requirements Completed | 31/31 |
| Multi-FG Concept Tested | 1+ |
| Approval Matrix Functional | Yes |
| Roadmap Branch Operational | Yes |
| Emergency Bypass Working | Yes |

---

## Next Steps

```
/gsd plan-phase 10
```

Begin with Phase 10: Concept Directory Architecture as the foundation for all subsequent phases.

---

*Roadmap for WGSD v2.2 — Matrix-Based Social Development Architecture*

---

## Phase 17: Migration Auto-Channel Creation

**Dependencies:** None (bug fix)  
**Estimated Effort:** 3-5 hours  
**Priority:** High (bug fix)  
**Status:** 🔵 Planned  

### Problem

WGSD migration workflow currently requires manual Slack channel creation after migration completes. This breaks the "one-step migration" promise and creates friction for teams adopting WGSD.

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| REQ-17-01 | Automatic dev channel creation during migration | Must Have |
| REQ-17-02 | Automatic focus group channel creation | Must Have |
| REQ-17-03 | Automatic implementation channel creation | Must Have |
| REQ-17-04 | Automatic team member invitation | Should Have |
| REQ-17-05 | Graceful error handling for channel failures | Must Have |

### Success Criteria

Migration workflow automatically creates all required Slack channels and invites team members without manual intervention.

### Implementation Notes

- **Target:** `workflows/migrate.md` Step 11 (after commit)
- **Integration:** Use existing `create-channel.md` workflow  
- **Error Handling:** Warn on failures but don't abort migration
- **Testing:** Verify with fresh Slack workspace

---

## Phase 18: Migration Auto-Canvas Creation

**Dependencies:** Phase 17 (channels must exist first)  
**Estimated Effort:** 4-6 hours  
**Priority:** High (missing feature)  
**Status:** 🔵 Planned  

### Problem

WGSD migration workflow does not create essential roadmap canvases, leaving the visual collaboration aspect unavailable until manual setup. This defeats the purpose of "complete workflow migration."

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| REQ-18-01 | Automatic master dashboard creation | Must Have |
| REQ-18-02 | Automatic focus group canvas creation | Must Have |
| REQ-18-03 | Automatic implementation canvas creation | Should Have |
| REQ-18-04 | Canvas registry initialization | Must Have |
| REQ-18-05 | Dynamic content population from .planning/ | Must Have |
| REQ-18-06 | Graceful canvas failure handling (non-fatal) | Must Have |

### Success Criteria

Migration workflow automatically creates and populates all essential roadmap canvases, making WGSD visual collaboration immediately available.

### Implementation Notes

- **Target:** `workflows/migrate.md` Step 12 (after channels)
- **Integration:** Use existing `canvas-create.md` workflows
- **Content:** Populate from current `.planning/` roadmap data
- **Error Handling:** Non-fatal failures with manual fallback documentation


---

## Phase 19: Canvas Auto-Sync Implementation

**Dependencies:** Canvas infrastructure (existing)  
**Estimated Effort:** 6-8 hours  
**Priority:** High (missing core feature)  
**Status:** 🔵 Planned  

### Problem

WGSD canvas sync is entirely manual despite extensive documentation suggesting automatic triggers should exist. Throughout the codebase, canvas sync calls are commented out with "would invoke" statements.

**Impact:** Created canvases become stale immediately after planning changes, defeating the purpose of live visual collaboration.

### Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| REQ-19-01 | Automatic concept merge sync to focus group + master canvases | Must Have |
| REQ-19-02 | Automatic branch update sync for focus group changes | Must Have |  
| REQ-19-03 | Automatic implementation state sync for dashboard updates | Must Have |
| REQ-19-04 | Automatic approval matrix sync for focus group canvases | Should Have |
| REQ-19-05 | Manual override capability preserved | Must Have |
| REQ-19-06 | Sync failure resilience (non-blocking) | Must Have |

### Success Criteria

Canvas content stays current automatically without manual intervention. All commented "would invoke" statements replaced with actual working sync calls.

### Implementation Notes

- **Pattern:** Replace `# Would invoke canvas-sync` with actual `/wgsd sync-canvas` calls
- **Locations:** 8+ files with commented sync triggers need implementation
- **Risk Mitigation:** Non-blocking calls, error logging, async execution
- **Testing:** Verify canvas updates on concept merges, branch changes, implementation state changes

### Evidence

Marvin migration revealed this bug - 7 canvases created but won't update automatically when concepts change or branches are updated.

