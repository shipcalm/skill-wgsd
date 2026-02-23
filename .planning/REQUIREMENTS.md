# WGSD v2.1 Requirements

**Milestone:** Migration Experience Improvements  
**Created:** 2026-02-23  
**Source:** User feedback from real-world migration attempt

---

## Overview

| Category | Count | Priority |
|----------|-------|----------|
| Migration Logic Fix | 3 | P0 - Critical |
| Slack Channel Automation | 4 | P1 - High |
| Approval Workflow | 4 | P1 - High |
| **Total** | **11** | — |

---

## Migration Logic Fix (P0 - Critical)

The current migration incorrectly maps GSD Phases to WGSD Focus Groups. Phases are specific work items with defined scope and tasks — they should map to Concepts, not Focus Groups.

**Correct Mapping:**
- GSD Phase → WGSD Concept (specific deliverable)
- Focus Groups are thematic containers defined by user (e.g., "Core", "Migration", "Channels")
- Requirements stay as requirements

### MIG-FIX-01: Update migrate.md Workflow

**As a** user migrating from GSD  
**I want** Phases to become Concepts  
**So that** the semantic meaning is preserved correctly

**Acceptance Criteria:**
- [ ] migrate.md maps Phase N → Concept (not Focus Group)
- [ ] Focus Group suggestions are thematic groupings, not 1:1 from phases
- [ ] Existing migration steps updated with correct terminology

**Files:** `workflows/migrate.md`

---

### MIG-FIX-02: Update migration-analyzer.md Agent

**As a** user running migration analysis  
**I want** the analyzer to suggest Concepts from Phases  
**So that** the migration preview shows correct mappings

**Acceptance Criteria:**
- [ ] Agent output includes `suggested_concepts[]` (not focus groups)
- [ ] Agent suggests thematic Focus Groups separately
- [ ] Agent explains Concept derivation from Phase content

**Files:** `agents/migration-analyzer.md`

---

### MIG-FIX-03: Update planning-migrator.md Agent

**As a** user completing migration  
**I want** Phase content transformed to Concept format  
**So that** migrated content matches WGSD structure

**Acceptance Criteria:**
- [ ] Agent transforms Phase description → Concept description
- [ ] Agent maps Phase requirements → Concept acceptance criteria
- [ ] Agent assigns Concepts to appropriate Focus Groups

**Files:** `agents/planning-migrator.md`

---

## Slack Channel Automation (P1 - High)

Currently migration completes but leaves user to manually create all Slack channels. The migration should auto-create the full channel structure.

### SLACK-AUTO-01: Auto-Create Core Dev Channel

**As a** user completing migration  
**I want** {stub}-dev channel created automatically  
**So that** I don't need manual setup

**Acceptance Criteria:**
- [ ] Private channel {stub}-dev created
- [ ] Channel has master dashboard Canvas
- [ ] User is added as channel member

**Files:** `workflows/migrate.md`, `workflows/lib/slack-api.md`

---

### SLACK-AUTO-02: Auto-Create Focus Group Channels

**As a** user completing migration  
**I want** {stub}-fg-{focus} channels created for each Focus Group  
**So that** the full channel structure is ready

**Acceptance Criteria:**
- [ ] Private channel created per Focus Group
- [ ] Each channel has appropriate Canvas
- [ ] User is added as member to all channels

**Files:** `workflows/migrate.md`

---

### SLACK-AUTO-03: Auto-Create Community Channel

**As a** user completing migration  
**I want** {stub}-community channel created  
**So that** community integration is ready

**Acceptance Criteria:**
- [ ] Public channel {stub}-community created
- [ ] Channel has roadmap Canvas
- [ ] Community channels follow security model

**Files:** `workflows/migrate.md`

---

### SLACK-AUTO-04: Integrate Channel Creation into Migration Flow

**As a** migration workflow  
**I want** channel creation to happen at the right step  
**So that** migration is a single cohesive operation

**Acceptance Criteria:**
- [ ] Channel creation happens after approval, before completion
- [ ] Failures are handled gracefully with rollback
- [ ] Success/failure reported in migration summary

**Files:** `workflows/migrate.md`

---

## Approval Workflow (P1 - High)

Currently migration executes immediately without user review. Users need to see what will happen and approve before execution.

### APPROVE-01: Generate Migration Preview

**As a** user starting migration  
**I want** a complete preview of proposed changes  
**So that** I understand what will happen

**Acceptance Criteria:**
- [ ] Preview shows: Focus Groups to create
- [ ] Preview shows: Concepts to create (with source Phase)
- [ ] Preview shows: Channels to create
- [ ] Preview shows: Files to transform

**Files:** `workflows/migrate.md`

---

### APPROVE-02: Display Preview to User

**As a** user viewing migration preview  
**I want** clear, formatted preview output  
**So that** I can easily understand the plan

**Acceptance Criteria:**
- [ ] Preview uses clear section headers
- [ ] Preview shows mapping (Phase X → Concept Y)
- [ ] Preview shows channel names that will be created
- [ ] Preview is conversational, not just data dump

**Files:** `workflows/migrate.md`

---

### APPROVE-03: Require Explicit Approval

**As a** user reviewing migration  
**I want** to explicitly approve before execution  
**So that** I have control over the process

**Acceptance Criteria:**
- [ ] Migration pauses after preview display
- [ ] User must confirm to proceed
- [ ] "No" response aborts migration cleanly
- [ ] Approval is logged in migration output

**Files:** `workflows/migrate.md`

---

### APPROVE-04: Allow Modification Before Approval

**As a** user reviewing migration preview  
**I want** to modify suggestions before approval  
**So that** I can customize the migration

**Acceptance Criteria:**
- [ ] User can rename Focus Groups
- [ ] User can reassign Concepts to different Focus Groups
- [ ] User can exclude certain Phases from migration
- [ ] Changes are reflected in updated preview

**Files:** `workflows/migrate.md`

---

## Traceability Matrix

| Requirement | Phase | Files Affected |
|-------------|-------|----------------|
| MIG-FIX-01 | 7 | workflows/migrate.md |
| MIG-FIX-02 | 7 | agents/migration-analyzer.md |
| MIG-FIX-03 | 7 | agents/planning-migrator.md |
| SLACK-AUTO-01 | 8 | workflows/migrate.md |
| SLACK-AUTO-02 | 8 | workflows/migrate.md |
| SLACK-AUTO-03 | 8 | workflows/migrate.md |
| SLACK-AUTO-04 | 8 | workflows/migrate.md |
| APPROVE-01 | 9 | workflows/migrate.md |
| APPROVE-02 | 9 | workflows/migrate.md |
| APPROVE-03 | 9 | workflows/migrate.md |
| APPROVE-04 | 9 | workflows/migrate.md |

---

## Dependencies

```
Phase 7 (Logic Fix) ─────────┐
                             ├──► Phase 9 (Approval)
Phase 8 (Slack Automation) ──┘
```

**Phase 7** and **Phase 8** can run in parallel.  
**Phase 9** depends on both (approval shows correct mappings + channel names).

---

## Definition of Done

- [ ] All 11 requirements implemented
- [ ] migrate.md correctly maps Phase → Concept
- [ ] All Slack channels auto-created
- [ ] User sees preview and approves before execution
- [ ] End-to-end migration tested successfully

---

*Requirements document for WGSD v2.1 — Migration Experience Improvements*
