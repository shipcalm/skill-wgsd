# WGSD v2.0 Roadmap

**Generated:** 2026-02-22
**Project:** WGSD v2.0 - Social Development Operating System
**Requirements:** 47 across 6 categories → 6 phases derived from natural dependencies

---

## Phase Overview

| Phase | Name | Requirements | Goal |
|-------|------|--------------|------|
| **1** | Foundation & Core Infrastructure | 6 | Establish workspace, git ops, and naming conventions |
| **2** | Migration Wizard | 9 | Intelligent GSD → WGSD migration with state detection |
| **3** | Channel Infrastructure | 7 | Automated Slack channel creation and lifecycle |
| **4** | Canvas Management System | 9 | AI-managed Canvas with bidirectional sync |
| **5** | Workflow Engine | 9 | Two-track development with approval gates |
| **6** | Community Integration | 7 | Public → private development pipeline |

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the foundational infrastructure for WGSD workspaces, git operations, and standardized naming conventions that all other phases depend on.

**Duration:** ~2-3 days
**Dependencies:** None (foundation layer)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| INTEGRATE-01 | Workspace management for wgsd/{project} structure | Medium |
| INTEGRATE-02 | Branch strategy enforcement (develop base, clean checkouts) | Medium |
| INTEGRATE-06 | Git operations integration for workspace and branch management | High |
| CHANNEL-01 | Ask for repo slack stub during setup (e.g., "mvn" for marvin) | Low |
| CHANNEL-02 | Create standardized channel naming convention implementation | Low |
| INTEGRATE-03 | .planning/ structure migration from GSD to WGSD format | Medium |

### Success Criteria

1. **User can initialize WGSD workspace**: Running `wgsd init mvn` creates `wgsd/mvn/` with clean develop checkout
2. **Naming conventions are enforced**: System rejects non-conforming channel names and suggests corrections
3. **Git operations work reliably**: User can see workspace status showing branch, uncommitted changes, and sync state
4. **Planning structure exists**: The `.planning/` directory contains WGSD-compatible structure (focus-groups/, etc.)

### Key Deliverables

- `workflows/init.md` - WGSD initialization workflow
- `workflows/workspace-status.md` - Workspace health check
- Git operations library (branch, checkout, worktree management)
- Channel naming convention validator

---

## Phase 2: Migration Wizard

**Goal:** Enable intelligent migration from existing GSD projects to WGSD, preserving work-in-progress and generating smart focus group suggestions from existing roadmap analysis.

**Duration:** ~3-4 days
**Dependencies:** Phase 1 (workspace, git ops, naming conventions)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| MIGRATE-01 | Detect GSD project state (current branch, phase, uncommitted work) | High |
| MIGRATE-02 | Analyze existing GSD roadmap phases to suggest focus group mappings | High |
| MIGRATE-03 | Analyze codebase structure to suggest additional focus groups (e.g., /api → API) | Medium |
| MIGRATE-04 | Preserve work-in-progress by converting current phase to active implementation | High |
| MIGRATE-05 | Auto-generate implementation names based on current phase/branch | Low |
| MIGRATE-06 | Create wgsd/{project} workspace with clean develop branch checkout | Medium |
| MIGRATE-07 | Migrate GSD planning artifacts (.planning/ROADMAP.md → focus group structure) | High |
| MIGRATE-08 | Draft team communication announcement for GSD → WGSD transition | Low |
| INTEGRATE-08 | Error handling and rollback for failed migrations | Medium |

### Success Criteria

1. **Migration wizard detects GSD state**: User sees accurate report of current phase, branch, uncommitted changes, and migration readiness
2. **Focus groups are intelligently suggested**: System proposes 2-5 focus groups based on roadmap phases AND codebase structure
3. **Work-in-progress is preserved**: Current phase converts to active implementation without losing any commits or changes
4. **Team communication is generated**: Draft announcement message is ready for Slack posting
5. **Rollback works on failure**: Failed migration leaves original GSD project untouched

### Key Deliverables

- `workflows/migrate.md` - Full migration wizard workflow
- `agents/migration-analyzer.md` - Roadmap and codebase analysis agent
- Migration state machine with rollback capability
- Team announcement template generator

---

## Phase 3: Channel Infrastructure

**Goal:** Automate Slack channel creation for all WGSD channel types (dev, focus groups, concepts, implementations) with proper private/public configuration and lifecycle management.

**Duration:** ~2-3 days
**Dependencies:** Phase 1 (naming conventions), Phase 2 (migration provides initial channels)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| CHANNEL-03 | Auto-create core dev channel ({stub}-dev) as private channel | Medium |
| CHANNEL-04 | Auto-create public community channel ({stub}-community) | Medium |
| CHANNEL-05 | Auto-create focus group channels ({stub}-fg-{name}) as private | Medium |
| CHANNEL-06 | Auto-create concept channels ({stub}-cpt-{name}) as private | Medium |
| CHANNEL-07 | Auto-create implementation channels ({stub}-impl-{name}) as private | Medium |
| CHANNEL-08 | Channel lifecycle management (creation, archival, cleanup) | High |
| INTEGRATE-04 | Slack API integration for private/public channel management | High |

### Success Criteria

1. **Channels are created automatically**: Running `wgsd create-focus-group security` creates `#mvn-fg-security` private channel
2. **Channel types are correct**: Dev/FG/concept/impl channels are private, community channel is public
3. **Lifecycle is managed**: Archived implementations are cleaned up, channels can be reactivated
4. **Bot joins automatically**: Jarvis is added to all created channels without manual invitation

### Key Deliverables

- Enhanced `workflows/create-channel.md` with all channel types
- `workflows/archive-channel.md` for lifecycle management
- Slack API wrapper with private/public channel support
- Channel registry (tracks all WGSD channels per project)

---

## Phase 4: Canvas Management System

**Goal:** Implement AI-managed Canvas system with master dashboard, focus group canvases, and bidirectional synchronization with .planning/ git files. Humans cannot edit canvases directly.

**Duration:** ~4-5 days
**Dependencies:** Phase 3 (channels to attach canvases to)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| CANVAS-01 | Create master dashboard Canvas in core dev channel with methodology guide | High |
| CANVAS-02 | Live roadmap visualization showing all focus groups, concepts, implementations | High |
| CANVAS-03 | Auto-update master Canvas when focus groups, concepts, or implementations change state | High |
| CANVAS-04 | Create focus group Canvas populated from migrated GSD planning docs | Medium |
| CANVAS-05 | AI-managed Canvas updates (no direct human editing allowed) | Medium |
| CANVAS-06 | Sync Canvas content with .planning/ git files bidirectionally | High |
| CANVAS-07 | Create community roadmap Canvas showing public development progress | Medium |
| CANVAS-08 | Implementation dashboard with queue, active work, and completed items | Medium |
| INTEGRATE-05 | conversations.canvases.create API integration | Medium |

### Success Criteria

1. **Master dashboard exists**: Core dev channel shows live Canvas with methodology guide and current state
2. **Canvas reflects reality**: Adding a concept in chat immediately appears in focus group Canvas roadmap
3. **Git and Canvas stay synchronized**: Changes to `.planning/` files appear in Canvas within minutes, and vice versa
4. **Community has read-only view**: Public community channel shows roadmap Canvas without internal details
5. **AI is the only editor**: Direct human Canvas edits are detected and reverted with explanation

### Key Deliverables

- `workflows/canvas-create.md` - Create canvases for all channel types
- `workflows/canvas-sync.md` - Bidirectional sync workflow
- Canvas change detection and reversion system
- Master dashboard template with live state injection
- Community roadmap generator (sanitized view)

---

## Phase 5: Workflow Engine

**Goal:** Implement the two-track development model (collaborative planning + trunk-based implementation) with approval gates, concept promotion, and implementation lifecycle management.

**Duration:** ~4-5 days
**Dependencies:** Phase 4 (canvas for visualization and state tracking)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| WORKFLOW-01 | Two-track development (collaborative planning track + trunk-based implementation track) | High |
| WORKFLOW-02 | Focus group approval process for concept maturation | Medium |
| WORKFLOW-03 | Concept → implementation promotion with owner assignment | High |
| WORKFLOW-04 | Implementation prioritization system visible in master dashboard | Medium |
| WORKFLOW-05 | Enforce ~1 implementation per focus group natural scaling | Medium |
| WORKFLOW-06 | Concept development workflow (planning PRs to focus groups) | High |
| WORKFLOW-07 | Implementation workflow (code PRs directly to develop) | High |
| WORKFLOW-08 | Implementation status tracking and progress visualization | Medium |
| INTEGRATE-07 | GitHub PR workflow integration for focus group → develop flow | High |

### Success Criteria

1. **Two tracks are distinct**: Concepts evolve through discussion and planning PRs, implementations go directly to develop
2. **Approval gates work**: Concept cannot promote to implementation without focus group approval (via reaction or command)
3. **Implementation has owner**: Promoted concept requires human assignment before becoming active implementation
4. **Natural scaling enforced**: System warns when focus group has >1 active implementation, requires override
5. **Progress is visible**: Master dashboard shows all implementations with status, owner, and estimated completion

### Key Deliverables

- `workflows/promote-concept.md` - Enhanced with approval gate and owner assignment
- `workflows/create-implementation.md` - Enhanced with prioritization and scaling enforcement
- `workflows/implementation-status.md` - Progress tracking and reporting
- GitHub PR integration for both tracks
- Implementation queue management

---

## Phase 6: Community Integration

**Goal:** Create the public feedback → private development pipeline, enabling community contributions to flow into focus groups with proper attribution and access management.

**Duration:** ~3-4 days
**Dependencies:** Phase 5 (workflow engine for moderation flow)

### Requirements

| ID | Requirement | Complexity |
|----|-------------|------------|
| COMMUNITY-01 | Public community feedback collection system | Medium |
| COMMUNITY-02 | Moderation triage workflow (move community ideas to focus groups) | High |
| COMMUNITY-03 | Auto-invite original community contributors to relevant private channels | Medium |
| COMMUNITY-04 | Community access request system for interested users | Medium |
| COMMUNITY-05 | Show community contributors in focus group Canvas with attribution | Low |
| COMMUNITY-06 | Create moderation commands (move-to-focus-group, invite-contributor, etc.) | Medium |
| COMMUNITY-07 | Transparent development visibility for community members | Medium |

### Success Criteria

1. **Feedback flows from public to private**: Community member posts idea in public channel, moderator can move it to focus group with one command
2. **Contributors get invited**: Original idea author receives automatic invitation to the private focus group channel
3. **Attribution is preserved**: Focus group Canvas shows "Originally proposed by @user in #community"
4. **Access requests work**: Community members can request access to specific focus groups, owners can approve/deny
5. **Transparency without security risk**: Community members see sanitized progress updates without internal discussions

### Key Deliverables

- `workflows/moderate-feedback.md` - Triage community input
- `workflows/invite-contributor.md` - Automated invitation flow
- `workflows/access-request.md` - Request and approval workflow
- Community attribution tracking system
- Sanitized progress update generator

---

## Milestone Summary

### v1.0 Scope

All 6 phases comprise v1.0 with full functionality:

| Category | Requirements |
|----------|--------------|
| Migration System | 8 |
| Channel Infrastructure | 8 |
| Canvas Management | 8 |
| Workflow System | 8 |
| Community Integration | 7 |
| Integration & Architecture | 8 |
| **Total** | **47** |

### Critical Path

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6
   │         │         │         │         │         │
   │         │         │         │         │         └── Community pipeline
   │         │         │         │         └── Two-track workflow
   │         │         │         └── AI-managed Canvas
   │         │         └── Channel automation
   │         └── Migration intelligence
   └── Foundation infrastructure
```

### Estimated Timeline

| Phase | Duration | Cumulative |
|-------|----------|------------|
| 1 | 2-3 days | 2-3 days |
| 2 | 3-4 days | 5-7 days |
| 3 | 2-3 days | 7-10 days |
| 4 | 4-5 days | 11-15 days |
| 5 | 4-5 days | 15-20 days |
| 6 | 3-4 days | 18-24 days |

**Total Estimated Duration:** 18-24 days

---

## Future Milestones (v2.0+)

See REQUIREMENTS.md v2 Requirements section for planned enhancements:
- Multi-repository WGSD projects
- Automated dependency detection between focus groups
- Advanced analytics and velocity tracking
- Scale optimization for large teams (20+)

---

*Roadmap derived from natural requirement dependencies, not imposed structure.*
*Last updated: 2026-02-22*
