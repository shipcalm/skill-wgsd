# WGSD v2.0 Requirements

## v1 Requirements

### Migration System
- [ ] **MIGRATE-01**: Detect GSD project state (current branch, phase, uncommitted work, clean migration requirements)
- [ ] **MIGRATE-02**: Analyze existing GSD roadmap phases to suggest intelligent focus group mappings
- [ ] **MIGRATE-03**: Analyze codebase structure to suggest additional focus groups (e.g., /api → API focus group)
- [ ] **MIGRATE-04**: Preserve work-in-progress by converting current phase to active implementation
- [ ] **MIGRATE-05**: Auto-generate implementation names based on current phase/branch (e.g., mvn-impl-auth-v2)
- [ ] **MIGRATE-06**: Create wgsd/{project} workspace with clean develop branch checkout
- [ ] **MIGRATE-07**: Migrate GSD planning artifacts (.planning/ROADMAP.md → focus group structure)
- [ ] **MIGRATE-08**: Draft team communication announcement for GSD → WGSD transition

### Channel Infrastructure
- [ ] **CHANNEL-01**: Ask for repo slack stub during setup (e.g., "mvn" for marvin project)
- [ ] **CHANNEL-02**: Create standardized channel naming convention implementation
- [ ] **CHANNEL-03**: Auto-create core dev channel ({stub}-dev) as private channel
- [ ] **CHANNEL-04**: Auto-create public community channel ({stub}-community) for customer feedback
- [ ] **CHANNEL-05**: Auto-create focus group channels ({stub}-fg-{name}) as private channels
- [ ] **CHANNEL-06**: Auto-create concept channels ({stub}-cpt-{name}) as private channels
- [ ] **CHANNEL-07**: Auto-create implementation channels ({stub}-impl-{name}) as private channels
- [ ] **CHANNEL-08**: Implement channel lifecycle management (creation, archival, cleanup)

### Canvas Management System
- [ ] **CANVAS-01**: Create master dashboard Canvas in core dev channel with WGSD methodology guide
- [ ] **CANVAS-02**: Implement live roadmap visualization showing all focus groups, concepts, implementations
- [ ] **CANVAS-03**: Auto-update master Canvas when focus groups, concepts, or implementations change state
- [ ] **CANVAS-04**: Create focus group Canvas populated from migrated GSD planning docs
- [ ] **CANVAS-05**: Implement AI-managed Canvas updates (no direct human editing allowed)
- [ ] **CANVAS-06**: Sync Canvas content with .planning/ git files bidirectionally
- [ ] **CANVAS-07**: Create community roadmap Canvas showing public development progress
- [ ] **CANVAS-08**: Show implementation dashboard with queue, active work, and completed items

### Workflow System
- [ ] **WORKFLOW-01**: Implement two-track development (collaborative planning track + trunk-based implementation track)
- [ ] **WORKFLOW-02**: Create focus group approval process for concept maturation
- [ ] **WORKFLOW-03**: Implement concept → implementation promotion with owner assignment
- [ ] **WORKFLOW-04**: Create implementation prioritization system visible in master dashboard
- [ ] **WORKFLOW-05**: Enforce ~1 implementation per focus group natural scaling
- [ ] **WORKFLOW-06**: Implement concept development workflow (planning PRs to focus groups)
- [ ] **WORKFLOW-07**: Implement implementation workflow (code PRs directly to develop)
- [ ] **WORKFLOW-08**: Create implementation status tracking and progress visualization

### Community Integration
- [ ] **COMMUNITY-01**: Create public community feedback collection system
- [ ] **COMMUNITY-02**: Implement moderation triage workflow (move community ideas to focus groups)
- [ ] **COMMUNITY-03**: Auto-invite original community contributors to relevant private channels
- [ ] **COMMUNITY-04**: Create community access request system for interested users
- [ ] **COMMUNITY-05**: Show community contributors in focus group Canvas with attribution
- [ ] **COMMUNITY-06**: Create moderation commands (move-to-focus-group, invite-contributor, etc.)
- [ ] **COMMUNITY-07**: Implement transparent development visibility for community members

### Integration & Architecture
- [ ] **INTEGRATE-01**: Implement workspace management for wgsd/{project} structure
- [ ] **INTEGRATE-02**: Create branch strategy enforcement (develop base, clean checkouts)
- [ ] **INTEGRATE-03**: Implement .planning/ structure migration from GSD to WGSD format
- [ ] **INTEGRATE-04**: Create Slack API integration for private/public channel management
- [ ] **INTEGRATE-05**: Implement conversations.canvases.create API integration for Canvas management
- [ ] **INTEGRATE-06**: Create git operations integration for workspace and branch management
- [ ] **INTEGRATE-07**: Implement GitHub PR workflow integration for focus group → develop flow
- [ ] **INTEGRATE-08**: Create proper error handling and rollback for failed migrations

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

## Out of Scope

- **Direct Canvas editing by humans** — All Canvas updates must go through AI conversation to maintain consistency
- **Public development channels** — Security requirement keeps all development discussions private
- **Automated implementation assignment** — Implementation ownership requires human decision and commitment
- **Non-Slack platforms** — Focus on Slack-native experience for initial version
- **Backwards compatibility with old WGSD format** — Clean break with v2.0 enhanced structure
- **Real-time collaboration editing** — Canvas updates are AI-managed, not real-time collaborative
- **Multi-tenant WGSD hosting** — Each organization runs their own WGSD instance

## Traceability

*This section will be populated by the roadmapper with phase mappings.*

---
*Requirements defined: 2026-02-22 during comprehensive GSD planning session*