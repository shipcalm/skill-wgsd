# WGSD v2.2 Requirements

**Milestone:** Matrix-Based Social Development Architecture  
**Created:** 2026-02-23  
**Source:** Architectural evolution for enterprise-scale social development

---

## Overview

| Category | Count | Priority |
|----------|-------|----------|
| Concept Directory Architecture | 5 | P0 - Critical |
| Cross-Cutting Impact System | 4 | P0 - Critical |
| Matrix-Based Approval | 6 | P0 - Critical |
| Roadmap Branch Architecture | 5 | P1 - High |
| Slack-Native Approval Workflow | 5 | P1 - High |
| Canvas Developer Info | 3 | P2 - Medium |
| Emergency Hotfix Bypass | 3 | P2 - Medium |
| **Total** | **31** | — |

---

## Phase 10: Concept Directory Architecture (P0 - Critical)

Transform concepts from single markdown files to full directories with multiple artifacts.

### CONCEPT-DIR-01: Create Concept as Directory

**As a** user creating a new concept  
**I want** a directory created instead of a single .md file  
**So that** I can store multiple related artifacts

**Acceptance Criteria:**
- [ ] `create-concept` workflow creates `concepts/{name}/` directory
- [ ] Directory structure includes CONCEPT.md as primary file
- [ ] Existing single-file concepts continue to work (backward compat)

**Files:** `workflows/create-concept.md` (new or modified)

---

### CONCEPT-DIR-02: Concept Directory Structure Template

**As a** developer working on a concept  
**I want** a standard directory structure  
**So that** artifacts are organized consistently

**Acceptance Criteria:**
- [ ] Template includes: CONCEPT.md, impact-matrix.md
- [ ] Optional artifacts supported: API-SPEC.md, wireframes/, acceptance-criteria.md
- [ ] Template is customizable via config

**Files:** `templates/concept-directory/` (new)

---

### CONCEPT-DIR-03: Independent Concept Branches

**As a** concept author  
**I want** concepts developed on independent branches  
**So that** work can happen in parallel without conflicts

**Acceptance Criteria:**
- [ ] New concepts create `concepts/{name}` branch
- [ ] Branch contains concept directory at `concepts/{name}/`
- [ ] Multiple concept branches can exist simultaneously

**Files:** `workflows/lib/branch-ops.md`, `workflows/create-concept.md`

---

### CONCEPT-DIR-04: Concept Worktree Support

**As a** developer  
**I want** optional git worktree setup for concept branches  
**So that** I can work on multiple concepts simultaneously

**Acceptance Criteria:**
- [ ] Optional worktree creation at `worktrees/{concept-name}/`
- [ ] Worktree path surfaced in Canvas
- [ ] Cleanup handles worktree removal on concept completion

**Files:** `workflows/lib/git-ops.md`, `workflows/create-concept.md`

---

### CONCEPT-DIR-05: Multi-Artifact Concept Sync

**As a** Canvas viewer  
**I want** to see all concept artifacts  
**So that** I have full visibility into concept scope

**Acceptance Criteria:**
- [ ] Canvas lists all files in concept directory
- [ ] Primary artifacts (CONCEPT.md, impact-matrix.md) displayed inline
- [ ] Secondary artifacts linked

**Files:** `workflows/canvas-sync.md`, `workflows/lib/canvas-templates.md`

---

## Phase 11: Cross-Cutting Impact System (P0 - Critical)

Enable concepts to declare and track impact across multiple focus groups.

### IMPACT-01: Impact Matrix File Format

**As a** concept author  
**I want** a structured way to declare focus group impacts  
**So that** stakeholders know what's affected

**Acceptance Criteria:**
- [ ] `impact-matrix.md` format defined with YAML frontmatter
- [ ] Supports multiple focus groups with per-group priority
- [ ] Includes impact description per focus group

**Example Format:**
```yaml
---
concept: oauth-integration
impacts:
  - focus_group: core
    priority: P1
    impact: "Authentication foundation changes"
  - focus_group: api
    priority: P0
    impact: "Breaking changes to auth endpoints"
---
```

**Files:** `templates/concept-directory/impact-matrix.md`

---

### IMPACT-02: Impact Declaration Workflow

**As a** concept author  
**I want** an interactive way to declare impacts  
**So that** I don't miss affected stakeholders

**Acceptance Criteria:**
- [ ] Workflow prompts for affected focus groups
- [ ] Suggests priorities based on impact type
- [ ] Generates impact-matrix.md from inputs

**Files:** `workflows/declare-impact.md` (new)

---

### IMPACT-03: Automatic Focus Group Notifications

**As a** focus group lead  
**I want** notification when a concept impacts my group  
**So that** I can track and prioritize

**Acceptance Criteria:**
- [ ] Slack message sent to {stub}-fg-{name} channel
- [ ] Message includes concept name, declared priority, impact description
- [ ] Message includes approval action prompt

**Files:** `workflows/lib/slack-api.md`, `workflows/declare-impact.md`

---

### IMPACT-04: Impact Change Tracking

**As a** stakeholder  
**I want** to be notified when concept impacts change  
**So that** I can update my prioritization

**Acceptance Criteria:**
- [ ] Impact changes (add/remove FG, priority change) trigger notifications
- [ ] Change history tracked in concept directory
- [ ] Canvas updated with impact changes

**Files:** `workflows/update-impact.md` (new)

---

## Phase 12: Matrix-Based Approval (P0 - Critical)

Transform approval from single owner to multi-stakeholder matrix.

### MATRIX-01: Approval Matrix Data Structure

**As a** system  
**I want** a structured approval matrix per concept  
**So that** multi-stakeholder approval is trackable

**Acceptance Criteria:**
- [ ] Approval matrix stored in concept directory
- [ ] Tracks: focus_group, priority, status, approver, timestamp
- [ ] Status enum: pending, approved, rejected, blocked

**Files:** `concepts/{name}/approval-matrix.json`

---

### MATRIX-02: Per-Focus-Group Approval

**As a** focus group representative  
**I want** to approve my group's involvement independently  
**So that** I don't block other groups

**Acceptance Criteria:**
- [ ] Approval scoped to single focus group
- [ ] Other groups' approvals unaffected
- [ ] Partial approval state visible in Canvas

**Files:** `workflows/lib/approval-system.md`

---

### MATRIX-03: Approval Matrix Canvas Widget

**As a** viewer  
**I want** to see approval matrix status at a glance  
**So that** I know what's blocking progress

**Acceptance Criteria:**
- [ ] Canvas shows approval matrix table
- [ ] Color-coded status (green=approved, yellow=pending, red=rejected)
- [ ] Shows approver and timestamp for completed approvals

**Files:** `workflows/lib/canvas-templates.md`

---

### MATRIX-04: Approval Completion Detection

**As a** system  
**I want** to detect when all required approvals are complete  
**So that** automated advancement can trigger

**Acceptance Criteria:**
- [ ] "Fully approved" state detected when all FGs approved
- [ ] Triggers roadmap merge workflow
- [ ] Handles optional vs required approval distinctions

**Files:** `workflows/lib/approval-system.md`

---

### MATRIX-05: Blocked Status Handling

**As a** focus group  
**I want** to mark my approval as blocked  
**So that** dependencies are visible

**Acceptance Criteria:**
- [ ] "Blocked" status with blocking reason
- [ ] Can specify blocking focus group
- [ ] Unblock triggers when blocker approves

**Files:** `workflows/lib/approval-system.md`

---

### MATRIX-06: Approval Override (Admin)

**As an** admin  
**I want** to override stuck approvals  
**So that** progress isn't blocked indefinitely

**Acceptance Criteria:**
- [ ] Admin can force-approve with reason
- [ ] Override logged and visible in audit trail
- [ ] Original approver notified of override

**Files:** `workflows/approval-override.md` (new)

---

## Phase 13: Roadmap Branch Architecture (P1 - High)

Establish roadmap branch as the source of truth for approved concepts.

### ROADMAP-01: Roadmap Branch Creation

**As a** WGSD project  
**I want** a roadmap branch created  
**So that** approved concepts have a landing place

**Acceptance Criteria:**
- [ ] `roadmap` branch created from main
- [ ] Branch protected from direct pushes
- [ ] Only approved concepts merged

**Files:** `workflows/init.md`, `workflows/lib/branch-ops.md`

---

### ROADMAP-02: Automatic Concept → Roadmap Merge

**As a** concept  
**When** fully approved  
**I want** automatic merge to roadmap branch  
**So that** advancement is seamless

**Acceptance Criteria:**
- [ ] Approval completion triggers merge workflow
- [ ] Concept directory merged to roadmap branch
- [ ] Concept branch archived (not deleted)
- [ ] Slack notification sent

**Files:** `workflows/promote-concept.md`

---

### ROADMAP-03: Implementation Branches from Roadmap

**As a** developer starting implementation  
**I want** to branch from roadmap  
**So that** I have all approved concepts available

**Acceptance Criteria:**
- [ ] Implementation workflow branches from `roadmap` not `develop`
- [ ] Develop branch rebased on roadmap periodically
- [ ] Clear documentation on branching model

**Files:** `workflows/create-implementation.md`

---

### ROADMAP-04: Roadmap Canvas View

**As a** stakeholder  
**I want** to see roadmap contents in Canvas  
**So that** I know what's approved and pending implementation

**Acceptance Criteria:**
- [ ] Canvas shows concepts on roadmap branch
- [ ] Implementation status per concept visible
- [ ] Priority sorting within roadmap view

**Files:** `workflows/lib/canvas-templates.md`

---

### ROADMAP-05: Roadmap Sync to Develop

**As a** developer  
**I want** roadmap changes synced to develop  
**So that** I'm working with latest approved concepts

**Acceptance Criteria:**
- [ ] Periodic or on-demand sync from roadmap → develop
- [ ] Conflicts handled with human review
- [ ] Sync status visible in dev Canvas

**Files:** `workflows/roadmap-sync.md` (new)

---

## Phase 14: Slack-Native Approval Workflow (P1 - High)

Conversational approvals in Slack channels without GitHub PRs.

### SLACK-APPROVE-01: Conversational Approval Prompts

**As a** focus group member  
**I want** approval prompts in my channel  
**So that** I can approve without leaving Slack

**Acceptance Criteria:**
- [ ] Approval request posted to focus group channel
- [ ] Includes concept summary, impact, priority
- [ ] "Approve" / "Reject" / "Needs Discussion" reactions or commands

**Files:** `workflows/lib/approval-system.md`, `workflows/lib/slack-api.md`

---

### SLACK-APPROVE-02: Approval via Slash Command

**As a** focus group lead  
**I want** to approve via `/wgsd approve {concept}`  
**So that** I can approve quickly

**Acceptance Criteria:**
- [ ] `/wgsd approve {concept}` command accepted
- [ ] Validates user has approval authority for focus group
- [ ] Updates approval matrix
- [ ] Confirms in channel

**Files:** `workflows/conversational-approve.md` (new)

---

### SLACK-APPROVE-03: Discussion Thread for Approval

**As a** focus group  
**I want** to discuss concept before approving  
**So that** concerns are addressed

**Acceptance Criteria:**
- [ ] Approval prompt creates or links to discussion thread
- [ ] Discussion visible to concept author
- [ ] Author can respond to concerns

**Files:** `workflows/lib/slack-api.md`

---

### SLACK-APPROVE-04: Approval Status Summary

**As a** concept author  
**I want** a summary of approval status  
**So that** I know what's blocking

**Acceptance Criteria:**
- [ ] `/wgsd status {concept}` shows approval matrix
- [ ] Highlights pending and blocked approvals
- [ ] Suggests next actions

**Files:** `workflows/concept-status.md` (new)

---

### SLACK-APPROVE-05: Rejection with Feedback

**As a** focus group  
**I want** to reject with feedback  
**So that** the author knows what to fix

**Acceptance Criteria:**
- [ ] Rejection requires reason
- [ ] Reason posted to dev channel
- [ ] Concept author notified
- [ ] Concept returns to draft status

**Files:** `workflows/lib/approval-system.md`

---

## Phase 15: Canvas Developer Info (P2 - Medium)

Show developers the git information they need.

### CANVAS-DEV-01: Display Current Branch

**As a** developer viewing Canvas  
**I want** to see the git branch  
**So that** I know what I'm looking at

**Acceptance Criteria:**
- [ ] Canvas header includes branch name
- [ ] Updates when viewing different concepts
- [ ] Clear visual distinction

**Files:** `workflows/lib/canvas-templates.md`

---

### CANVAS-DEV-02: Display Worktree Path

**As a** developer with multiple worktrees  
**I want** to see the worktree path  
**So that** I can navigate quickly

**Acceptance Criteria:**
- [ ] Worktree path shown when applicable
- [ ] Copy-able path format
- [ ] "No worktree" indicator when not set up

**Files:** `workflows/lib/canvas-templates.md`

---

### CANVAS-DEV-03: Git Status Quick Reference

**As a** developer  
**I want** basic git status in Canvas  
**So that** I have context without terminal

**Acceptance Criteria:**
- [ ] Shows: branch, clean/dirty, ahead/behind
- [ ] Last commit summary
- [ ] Quick-action suggestions

**Files:** `workflows/canvas-sync.md`

---

## Phase 16: Emergency Hotfix Bypass (P2 - Medium)

Allow urgent fixes to bypass approval matrix.

### HOTFIX-01: Emergency Hotfix Workflow

**As a** developer with urgent fix  
**I want** to bypass concept/approval process  
**So that** critical issues are fixed fast

**Acceptance Criteria:**
- [ ] `/wgsd hotfix start {name}` creates hotfix branch from develop
- [ ] No concept, no approval matrix required
- [ ] Clear marking as emergency bypass

**Files:** `workflows/emergency-hotfix.md` (new)

---

### HOTFIX-02: Hotfix Completion and Merge

**As a** developer completing hotfix  
**I want** fast merge to develop and main  
**So that** fix is deployed quickly

**Acceptance Criteria:**
- [ ] Hotfix merges to develop directly
- [ ] Option to fast-track to main
- [ ] Audit trail maintained

**Files:** `workflows/emergency-hotfix.md`

---

### HOTFIX-03: Post-Hotfix Concept Creation

**As a** project maintainer  
**I want** hotfixes tracked retroactively  
**So that** documentation is complete

**Acceptance Criteria:**
- [ ] Optional: create concept from completed hotfix
- [ ] Concept created as "implemented" status
- [ ] Linked to hotfix commits

**Files:** `workflows/emergency-hotfix.md`

---

## Traceability Matrix

| Phase | Requirements | Files Affected |
|-------|--------------|----------------|
| 10 | CONCEPT-DIR-01 to 05 | workflows/create-concept.md, templates/concept-directory/, workflows/lib/branch-ops.md, workflows/lib/git-ops.md, workflows/canvas-sync.md |
| 11 | IMPACT-01 to 04 | templates/concept-directory/impact-matrix.md, workflows/declare-impact.md, workflows/lib/slack-api.md, workflows/update-impact.md |
| 12 | MATRIX-01 to 06 | workflows/lib/approval-system.md, workflows/lib/canvas-templates.md, workflows/approval-override.md |
| 13 | ROADMAP-01 to 05 | workflows/init.md, workflows/lib/branch-ops.md, workflows/promote-concept.md, workflows/create-implementation.md, workflows/roadmap-sync.md |
| 14 | SLACK-APPROVE-01 to 05 | workflows/lib/approval-system.md, workflows/lib/slack-api.md, workflows/conversational-approve.md, workflows/concept-status.md |
| 15 | CANVAS-DEV-01 to 03 | workflows/lib/canvas-templates.md, workflows/canvas-sync.md |
| 16 | HOTFIX-01 to 03 | workflows/emergency-hotfix.md |

---

## Dependencies

```
Phase 10 (Concept Directories) ─────┐
                                    ├──► Phase 12 (Matrix Approval)
Phase 11 (Cross-Cutting Impact) ────┤
                                    │
                                    └──► Phase 14 (Slack Approval)

Phase 12 (Matrix Approval) ─────────┐
                                    ├──► Phase 13 (Roadmap Branch)
Phase 14 (Slack Approval) ──────────┘

Phase 13 (Roadmap Branch) ──────────► Phase 15 (Canvas Dev Info)

Phase 16 (Emergency Hotfix) ────────► Independent (can start after Phase 10)
```

---

## Definition of Done

- [ ] All 31 requirements implemented
- [ ] Concepts are directories with multiple artifacts
- [ ] Cross-cutting impacts declared and tracked
- [ ] Matrix-based approval working per focus group
- [ ] Roadmap branch receives approved concepts
- [ ] Implementations branch from roadmap
- [ ] Slack-native approvals functional
- [ ] Canvas shows developer git info
- [ ] Emergency hotfix bypass operational
- [ ] Full end-to-end test with multi-FG concept

---

*Requirements document for WGSD v2.2 — Matrix-Based Social Development Architecture*
