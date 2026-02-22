# WGSD v2.0 Requirements

## v1 Requirements

### Migration System
- [ ] **MIGRATE-01**: Detect GSD project state (current branch, phase, uncommitted work, clean migration requirements) → **Phase 2**
- [ ] **MIGRATE-02**: Analyze existing GSD roadmap phases to suggest intelligent focus group mappings → **Phase 2**
- [ ] **MIGRATE-03**: Analyze codebase structure to suggest additional focus groups (e.g., /api → API focus group) → **Phase 2**
- [ ] **MIGRATE-04**: Preserve work-in-progress by converting current phase to active implementation → **Phase 2**
- [ ] **MIGRATE-05**: Auto-generate implementation names based on current phase/branch (e.g., mvn-impl-auth-v2) → **Phase 2**
- [ ] **MIGRATE-06**: Create wgsd/{project} workspace with clean develop branch checkout → **Phase 2**
- [ ] **MIGRATE-07**: Migrate GSD planning artifacts (.planning/ROADMAP.md → focus group structure) → **Phase 2**
- [ ] **MIGRATE-08**: Draft team communication announcement for GSD → WGSD transition → **Phase 2**

### Channel Infrastructure
- [ ] **CHANNEL-01**: Ask for repo slack stub during setup (e.g., "mvn" for marvin project) → **Phase 1**
- [ ] **CHANNEL-02**: Create standardized channel naming convention implementation → **Phase 1**
- [ ] **CHANNEL-03**: Auto-create core dev channel ({stub}-dev) as private channel → **Phase 3**
- [ ] **CHANNEL-04**: Auto-create public community channel ({stub}-community) for customer feedback → **Phase 3**
- [ ] **CHANNEL-05**: Auto-create focus group channels ({stub}-fg-{name}) as private channels → **Phase 3**
- [ ] **CHANNEL-06**: Auto-create concept channels ({stub}-cpt-{name}) as private channels → **Phase 3**
- [ ] **CHANNEL-07**: Auto-create implementation channels ({stub}-impl-{name}) as private channels → **Phase 3**
- [ ] **CHANNEL-08**: Implement channel lifecycle management (creation, archival, cleanup) → **Phase 3**

### Canvas Management System
- [ ] **CANVAS-01**: Create master dashboard Canvas in core dev channel with WGSD methodology guide → **Phase 4**
- [ ] **CANVAS-02**: Implement live roadmap visualization showing all focus groups, concepts, implementations → **Phase 4**
- [ ] **CANVAS-03**: Auto-update master Canvas when focus groups, concepts, or implementations change state → **Phase 4**
- [ ] **CANVAS-04**: Create focus group Canvas populated from migrated GSD planning docs → **Phase 4**
- [ ] **CANVAS-05**: Implement AI-managed Canvas updates (no direct human editing allowed) → **Phase 4**
- [ ] **CANVAS-06**: Sync Canvas content with .planning/ git files bidirectionally → **Phase 4**
- [ ] **CANVAS-07**: Create community roadmap Canvas showing public development progress → **Phase 4**
- [ ] **CANVAS-08**: Show implementation dashboard with queue, active work, and completed items → **Phase 4**

### Workflow System
- [ ] **WORKFLOW-01**: Implement two-track development (collaborative planning track + trunk-based implementation track) → **Phase 5**
- [ ] **WORKFLOW-02**: Create focus group approval process for concept maturation → **Phase 5**
- [ ] **WORKFLOW-03**: Implement concept → implementation promotion with owner assignment → **Phase 5**
- [ ] **WORKFLOW-04**: Create implementation prioritization system visible in master dashboard → **Phase 5**
- [ ] **WORKFLOW-05**: Enforce ~1 implementation per focus group natural scaling → **Phase 5**
- [ ] **WORKFLOW-06**: Implement concept development workflow (planning PRs to focus groups) → **Phase 5**
- [ ] **WORKFLOW-07**: Implement implementation workflow (code PRs directly to develop) → **Phase 5**
- [ ] **WORKFLOW-08**: Create implementation status tracking and progress visualization → **Phase 5**

### Community Integration
- [ ] **COMMUNITY-01**: Create public community feedback collection system → **Phase 6**
- [ ] **COMMUNITY-02**: Implement moderation triage workflow (move community ideas to focus groups) → **Phase 6**
- [ ] **COMMUNITY-03**: Auto-invite original community contributors to relevant private channels → **Phase 6**
- [ ] **COMMUNITY-04**: Create community access request system for interested users → **Phase 6**
- [ ] **COMMUNITY-05**: Show community contributors in focus group Canvas with attribution → **Phase 6**
- [ ] **COMMUNITY-06**: Create moderation commands (move-to-focus-group, invite-contributor, etc.) → **Phase 6**
- [ ] **COMMUNITY-07**: Implement transparent development visibility for community members → **Phase 6**

### Integration & Architecture
- [ ] **INTEGRATE-01**: Implement workspace management for wgsd/{project} structure → **Phase 1**
- [ ] **INTEGRATE-02**: Create branch strategy enforcement (develop base, clean checkouts) → **Phase 1**
- [ ] **INTEGRATE-03**: Implement .planning/ structure migration from GSD to WGSD format → **Phase 1**
- [ ] **INTEGRATE-04**: Create Slack API integration for private/public channel management → **Phase 3**
- [ ] **INTEGRATE-05**: Implement conversations.canvases.create API integration for Canvas management → **Phase 4**
- [ ] **INTEGRATE-06**: Create git operations integration for workspace and branch management → **Phase 1**
- [ ] **INTEGRATE-07**: Implement GitHub PR workflow integration for focus group → develop flow → **Phase 5**
- [ ] **INTEGRATE-08**: Create proper error handling and rollback for failed migrations → **Phase 2**

---

## Traceability Matrix

| Phase | Requirements | Count |
|-------|--------------|-------|
| **Phase 1: Foundation** | INTEGRATE-01, INTEGRATE-02, INTEGRATE-03, INTEGRATE-06, CHANNEL-01, CHANNEL-02 | 6 |
| **Phase 2: Migration** | MIGRATE-01, MIGRATE-02, MIGRATE-03, MIGRATE-04, MIGRATE-05, MIGRATE-06, MIGRATE-07, MIGRATE-08, INTEGRATE-08 | 9 |
| **Phase 3: Channels** | CHANNEL-03, CHANNEL-04, CHANNEL-05, CHANNEL-06, CHANNEL-07, CHANNEL-08, INTEGRATE-04 | 7 |
| **Phase 4: Canvas** | CANVAS-01, CANVAS-02, CANVAS-03, CANVAS-04, CANVAS-05, CANVAS-06, CANVAS-07, CANVAS-08, INTEGRATE-05 | 9 |
| **Phase 5: Workflow** | WORKFLOW-01, WORKFLOW-02, WORKFLOW-03, WORKFLOW-04, WORKFLOW-05, WORKFLOW-06, WORKFLOW-07, WORKFLOW-08, INTEGRATE-07 | 9 |
| **Phase 6: Community** | COMMUNITY-01, COMMUNITY-02, COMMUNITY-03, COMMUNITY-04, COMMUNITY-05, COMMUNITY-06, COMMUNITY-07 | 7 |
| **Total** | | **47** |

---

## v2 Requirements (Future)

### Advanced Features
- [ ] **ADVANCED-01**: Multi-repository WGSD project support
- [ ] **ADVANCED-02**: Automated dependency detection between focus groups
- [ ] **ADVANCED-03**: Implementation resource allocation and scheduling
- [ ] **ADVANCED-04**: Advanced analytics and development velocity tracking
- [ ] **ADVANCED-05**: Integration with other collaboration platforms beyond Slack

### Scale & Performance  
- [ ] **SCALE-01**: Support for large teams (20+ developers across multiple focus groups)
- [ ] **SCALE-02**: Advanced conflict detection and resolution between implementations
- [ ] **SCALE-03**: Automated load balancing of implementation assignments
- [ ] **SCALE-04**: Performance optimization for Canvas updates at scale

---

## Out of Scope

- **Direct Canvas editing by humans** — All Canvas updates must go through AI conversation to maintain consistency
- **Public development channels** — Security requirement keeps all development discussions private
- **Automated implementation assignment** — Implementation ownership requires human decision and commitment
- **Non-Slack platforms** — Focus on Slack-native experience for initial version
- **Backwards compatibility with old WGSD format** — Clean break with v2.0 enhanced structure
- **Real-time collaboration editing** — Canvas updates are AI-managed, not real-time collaborative
- **Multi-tenant WGSD hosting** — Each organization runs their own WGSD instance

---

## Coverage Verification

**All 47 v1 requirements mapped to exactly one phase:**

### By Category
| Category | Req Count | Phases Used |
|----------|-----------|-------------|
| Migration System | 8 | Phase 2 |
| Channel Infrastructure | 8 | Phase 1, Phase 3 |
| Canvas Management | 8 | Phase 4 |
| Workflow System | 8 | Phase 5 |
| Community Integration | 7 | Phase 6 |
| Integration & Architecture | 8 | Phase 1, Phase 2, Phase 3, Phase 4, Phase 5 |

### Integration Requirements Distribution
| INTEGRATE-* | Phase | Rationale |
|-------------|-------|-----------|
| INTEGRATE-01 | 1 | Workspace is foundational |
| INTEGRATE-02 | 1 | Branch strategy is foundational |
| INTEGRATE-03 | 1 | Planning structure needed before migration |
| INTEGRATE-04 | 3 | Slack API for channel automation |
| INTEGRATE-05 | 4 | Canvas API for canvas management |
| INTEGRATE-06 | 1 | Git ops are foundational |
| INTEGRATE-07 | 5 | PR workflow is part of workflow engine |
| INTEGRATE-08 | 2 | Rollback needed for migration safety |

---

*Requirements defined: 2026-02-22 during comprehensive GSD planning session*
*Traceability added: 2026-02-22 by roadmapper*
