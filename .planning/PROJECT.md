# WGSD v2.1 - Migration Experience Improvements

## What This Is

A focused iteration on the WGSD migration system based on real-world user feedback. Addresses migration logic errors, adds Slack channel automation, and introduces interactive approval workflow for safer migrations.

## Core Value

Make GSD → WGSD migration bulletproof with correct mapping logic, full automation, and user confidence through interactive approval before execution.

---

## Current Milestone: v2.1 Migration Experience Improvements

**Goal:** Fix migration logic, automate Slack channels, add approval step

**Target features:**
- Correct Phase → Concept mapping (not Focus Group)
- Auto-create all WGSD Slack channels during migration
- Interactive preview/approval before migration execution

---

## Previous: v2.0 - Social Development Operating System

Comprehensive enhancement that transforms individual GSD projects into collaborative social development platforms. Provides intelligent migration from GSD to WGSD, creates structured channel infrastructure with AI-managed Canvas integration, and enables seamless community feedback → development pipelines through Slack-native collaborative workflows.

## Requirements

### v2.0 (Complete) ✅

All 47 v2.0 requirements delivered — see MILESTONES.md for archive.

### v2.1 Requirements (Active)

#### Migration Logic Fix
- [ ] **MIG-FIX-01**: Update migrate.md to map GSD Phases → WGSD Concepts (not Focus Groups)
- [ ] **MIG-FIX-02**: Update migration-analyzer.md agent to suggest Concepts from Phases
- [ ] **MIG-FIX-03**: Update planning-migrator.md to transform Phase content into Concept format

#### Slack Channel Automation  
- [ ] **SLACK-AUTO-01**: Auto-create {stub}-dev channel during migration
- [ ] **SLACK-AUTO-02**: Auto-create {stub}-fg-{focus} channels for suggested Focus Groups
- [ ] **SLACK-AUTO-03**: Auto-create {stub}-community channel (public) with roadmap Canvas
- [ ] **SLACK-AUTO-04**: Integrate channel creation into migrate.md workflow

#### Approval Workflow
- [ ] **APPROVE-01**: Generate migration preview showing all proposed changes
- [ ] **APPROVE-02**: Display preview to user before any execution
- [ ] **APPROVE-03**: Require explicit approval before migration proceeds
- [ ] **APPROVE-04**: Allow user to modify suggestions before approval

### Out of Scope

- Direct Canvas editing by humans — AI manages all Canvas updates
- Non-Slack platforms — Slack-native focus maintained
- Multi-repository projects — single repo per WGSD project

## Context

**v2.1 User Feedback:**
Real-world migration attempt revealed three issues:
1. **Mapping Error**: migrate.md incorrectly maps Phase → Focus Group (should be Phase → Concept)
2. **Manual Slack Work**: User must manually create channels after migration
3. **No Preview**: Migration executes immediately without user review/approval

**v2.0 Foundation:**
- Complete WGSD v2.0 system with 47 requirements delivered
- Migration wizard exists but has logic error in mapping
- Channel infrastructure fully built but not integrated into migration
- All approval patterns established in workflow engine

## Constraints

- **Quick Iteration**: Focused scope — fix issues, don't add features
- **Backward Compatibility**: Existing workflows must continue working
- **User Experience**: Approval step must be helpful, not annoying

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Phase → Concept mapping | Phases are specific work items, like Concepts (not thematic groups) | v2.1 |
| Focus Groups are thematic | User defines Focus Groups; Concepts are derived from Phases | v2.1 |
| Full Slack automation | Reduce manual work, ensure consistent channel structure | v2.1 |
| Preview before execute | User confidence and ability to adjust suggestions | v2.1 |

---
*Last updated: 2026-02-23 — Milestone v2.1 started*