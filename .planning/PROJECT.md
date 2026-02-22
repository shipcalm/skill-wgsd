# WGSD v2.0 - Social Development Operating System

## What This Is

A comprehensive enhancement to the WGSD (We Get Shit Done) skill that transforms individual GSD projects into collaborative social development platforms. It provides intelligent migration from GSD to WGSD, creates structured channel infrastructure with AI-managed Canvas integration, and enables seamless community feedback → development pipelines through Slack-native collaborative workflows.

## Core Value

Enable effortless transition from individual development (GSD) to social collaborative development (WGSD) while maintaining development velocity and creating transparent community engagement.

## Requirements

### Validated

- ✓ Basic WGSD methodology (Focus Groups → Concepts → Implementations) — existing
- ✓ Workflow system architecture with SKILL.md routing — existing  
- ✓ Slack Canvas integration via conversations.canvases.create API — existing
- ✓ Repository structure with .planning/ directory — existing

### Active

- [ ] **MIGRATE-01**: Intelligent GSD → WGSD migration wizard with current state detection
- [ ] **MIGRATE-02**: Automated focus group suggestions from existing GSD roadmap analysis
- [ ] **MIGRATE-03**: Work-in-progress preservation as active implementations
- [ ] **MIGRATE-04**: Team communication drafting and transition announcement
- [ ] **CHANNEL-01**: Standardized channel naming convention ({stub}-dev, {stub}-fg-{focus}, etc.)
- [ ] **CHANNEL-02**: Automated private channel creation and management
- [ ] **CHANNEL-03**: Public community channel setup with roadmap Canvas
- [ ] **CANVAS-01**: AI-managed Canvas updates (no direct human editing)
- [ ] **CANVAS-02**: Master dashboard Canvas in core dev channel
- [ ] **CANVAS-03**: Cross-channel Canvas synchronization with planning files
- [ ] **WORKFLOW-01**: Two-track development (collaborative planning, trunk-based implementation)
- [ ] **WORKFLOW-02**: Focus group approval gates for concept maturation
- [ ] **WORKFLOW-03**: Implementation ownership assignment and prioritization
- [ ] **COMMUNITY-01**: Public feedback collection and moderation triage
- [ ] **COMMUNITY-02**: Community contributor invitation to private channels
- [ ] **COMMUNITY-03**: Access request system for interested community members
- [ ] **INTEGRATE-01**: Workspace management (wgsd/{project} structure)
- [ ] **INTEGRATE-02**: Branch strategy enforcement (develop base, clean checkouts)
- [ ] **INTEGRATE-03**: Planning artifact migration from GSD to WGSD structure

### Out of Scope

- Direct Canvas editing by humans — AI manages all Canvas updates through conversation
- Support for non-Slack collaboration platforms — focus on Slack-native experience initially
- Multi-repository WGSD projects — one repository per WGSD project for v1
- Automated implementation assignment — requires human decision for ownership
- Public channel support — all development channels remain private for security

## Context

**Current WGSD State:**
- Functional three-tier architecture with basic workflows
- Slack Canvas integration working via exec + API calls
- 6 core workflows implemented (create-focus-group, create-concept, etc.)
- Codebase mapped with 7 analysis documents showing skill-based architecture

**Migration Challenge:**
- The github/marvin project is currently running on GSD mid-phase
- Need seamless migration path that preserves work-in-progress
- Team needs to transition from individual to collaborative development
- Existing GSD planning artifacts contain valuable roadmap and requirements data

**Strategic Vision:**
- Transform WGSD into the definitive collaborative development methodology
- Enable customer-driven development through community integration
- Maintain development velocity while adding social collaboration
- Create reusable migration pattern for other GSD projects

## Constraints

- **Slack API**: Private channels only — security requirement for internal development
- **Git Strategy**: Must support clean develop branch checkout while preserving GSD state
- **Canvas Management**: AI-only updates to prevent inconsistency and chaos
- **Concurrent Implementations**: Natural scaling (~1 per focus group) to prevent conflicts
- **Team Size**: System must work for both small teams (2-3) and larger teams (10+)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| AI-managed Canvas only | Prevents chaos and maintains consistency across focus groups | — Pending |
| Two-track development model | Separates collaborative planning from trunk-based implementation | — Pending |
| Private channels for development | Internal company security requirements | — Pending |
| Standardized {stub}- naming | Instant recognition and easy project discovery | — Pending |
| Community → development pipeline | Customer-driven feature development and transparency | — Pending |
| GSD migration wizard | Enables existing projects to adopt WGSD methodology | — Pending |

---
*Last updated: 2026-02-22 after comprehensive requirements gathering*