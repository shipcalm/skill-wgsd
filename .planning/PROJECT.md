# WGSD v2.2 - Matrix-Based Social Development Architecture

## What This Is

A major architectural evolution that transforms WGSD from single focus group ownership to a sophisticated multi-stakeholder approval matrix. Concepts become first-class citizens with their own branches, directories, and cross-cutting impact declarations. The roadmap branch becomes the authoritative source for approved concepts, with implementations branching from roadmap rather than develop.

## Core Value

Enable complex enterprise concepts that impact multiple focus groups to be properly planned, prioritized across stakeholders, and approved through conversational Slack-native workflows. Developers get clear visibility into git branches and worktree paths. Emergency hotfixes bypass the full approval matrix when speed matters.

---

## Current Milestone: v2.2 Matrix-Based Social Development

**Goal:** Transform from single focus group ownership to cross-cutting concept approval matrix  
**Total Requirements:** 31  
**Phases:** 7 (Phases 10-16)

### Phase Status

| Phase | Name | Requirements | Status |
|-------|------|--------------|--------|
| 10 | Concept Directory Architecture | 5 | ✅ **Complete** |
| 11 | Cross-Cutting Impact System | 4 | ✅ **Complete** |
| 12 | Matrix-Based Approval | 6 | 🟡 **Planned** |
| 13 | Roadmap Branch Architecture | 5 | ⚪ Not Started |
| 14 | Slack-Native Approval Workflow | 5 | ⚪ Not Started |
| 15 | Canvas Developer Info | 3 | ⚪ Not Started |
| 16 | Emergency Hotfix Bypass | 3 | ⚪ Not Started |

### Phase 12 Execution Plan

**Location:** `.planning/phases/phase-12/PLAN.md`

**Current Phase - Matrix-Based Approval System:**
1. ⬜ MATRIX-01: Approval Matrix Data Structure
2. ⬜ MATRIX-02: Per-Focus-Group Approval  
3. ⬜ MATRIX-03: Approval Matrix Canvas Widget
4. ⬜ MATRIX-04: Approval Completion Detection
5. ⬜ MATRIX-05: Blocked Status Handling
6. ⬜ MATRIX-06: Approval Override (Admin)

**Deliverables:**
- `workflows/matrix-approve.md` (NEW)
- `workflows/matrix-block.md` (NEW)
- `workflows/approval-override.md` (NEW)
- Enhanced `workflows/lib/approval-system.md`
- Enhanced `workflows/lib/canvas-templates.md`

**Estimated Time:** ~3-4 hours  
**Dependencies:** Phase 10 ✅, Phase 11 ✅  
**Enables:** Phase 13 (Roadmap Branch), Phase 14 (Slack-Native Approvals)

---

**Target Architecture Changes:**
1. **Cross-cutting concept support** - concepts impact multiple focus groups
2. **Canvas developer info** - git branch + worktree paths displayed
3. **Concept-centric architecture** - independent concept branches + directories (not single .md files)
4. **Matrix-based approval system** - multi-stakeholder approval with per-focus-group prioritization
5. **Roadmap branch architecture** - approved concepts merge to roadmap, implementations branch from roadmap
6. **Slack-native approval workflow** - conversational approvals, not GitHub PRs
7. **Emergency hotfix bypass** - direct develop → fix → develop for emergencies

---

## Previous Milestones

### v2.1 - Migration Experience Improvements ✅
Fixed migration logic, added Slack channel automation, introduced approval workflow.

### v2.0 - Social Development Operating System ✅
Complete WGSD foundation with 47 requirements — migration, channels, Canvas, workflows.

---

## Architecture Overview

### Current (v2.0-2.1)
```
Concept → Single Focus Group → Approval → Implementation
                     └── One owner, one priority
```

### Target (v2.2)
```
Concept Directory → Multiple Focus Groups → Matrix Approval → Roadmap Branch
      │                     │                      │               │
      │                     │                      │               └── Implementations branch here
      │                     │                      └── Each FG prioritizes independently
      │                     └── Cross-cutting impact declarations
      └── Multiple artifacts: CONCEPT.md, API-SPEC.md, wireframes/, etc.
```

---

## Key Concepts Introduced

### Concept Directories
Not just `concepts/oauth-integration.md` but full directories:
```
concepts/oauth-integration/
├── CONCEPT.md              # Core concept description
├── API-SPEC.md            # API specification  
├── wireframes/            # Design artifacts
├── acceptance-criteria.md # Detailed AC
└── impact-matrix.md       # Which FGs, what priority per FG
```

### Cross-Cutting Impact Matrix
A single concept can impact multiple focus groups:
```
oauth-integration impacts:
  - Core (P1): Authentication foundation
  - API (P0): Breaking changes to auth endpoints
  - Mobile (P2): SDK integration required
  - Documentation (P1): New auth guide needed
```

### Approval Matrix
Each impacted focus group must approve independently:
```
┌─────────────────────────────────────────────────────────┐
│ oauth-integration Approval Matrix                       │
├──────────┬──────────┬──────────────┬──────────────────┤
│ Focus    │ Priority │ Status       │ Approver         │
├──────────┼──────────┼──────────────┼──────────────────┤
│ Core     │ P1       │ ✅ Approved  │ @alice (2/23)    │
│ API      │ P0       │ ⏳ Pending   │ Needs @bob       │
│ Mobile   │ P2       │ ✅ Approved  │ @charlie (2/22)  │
│ Docs     │ P1       │ 🚫 Blocked  │ Waiting on API   │
└──────────┴──────────┴──────────────┴──────────────────┘
```

### Roadmap Branch
```
main
├── develop (active development, implementations)
├── roadmap (approved concepts ready for implementation)
│   └── Concepts merge here when fully approved
└── concepts/oauth-integration (concept development)
    └── Merged to roadmap when approval matrix complete
```

### Implementation Flow (v2.2)
```
Concept Branch → Approval Matrix → Roadmap Branch → Implementation Branch → Develop → Main
```

### Emergency Hotfix Bypass
```
Emergency: develop → hotfix/fix-name → develop → main
           (skip concept, skip roadmap, skip approval matrix)
```

---

## Constraints

- **Slack-Native**: All approval workflows happen in Slack, not GitHub PRs
- **AI-Managed Canvas**: Humans don't edit Canvas directly; AI keeps it in sync
- **Backward Compatibility**: Existing simple concepts (single FG) still work
- **Git-Based Truth**: Branch/file structure is authoritative, Canvas reflects it

---

## Success Criteria

1. Concept creates as directory with multiple artifacts
2. Concept declares impact across multiple focus groups
3. Each focus group can set independent priority
4. Approval matrix tracks per-focus-group approval status
5. Fully approved concepts merge to roadmap branch automatically
6. Implementations branch from roadmap, not develop
7. Canvas shows developer paths (branch, worktree)
8. Emergency hotfixes bypass full approval matrix
9. Conversational approvals work in Slack channels

---

## Out of Scope

- GitHub PR-based approvals (Slack-native only)
- Multi-repository concept spanning
- Direct Canvas editing by humans
- Non-Slack platforms

---

*Last updated: 2026-02-23 — Milestone v2.2 initialized*
