# Phase 4: Canvas Management System - Research

**Date:** 2026-02-22
**Phase:** 4 of 6
**Focus:** AI-managed Canvas with bidirectional sync

---

## Canvas API Research

### Available Slack Canvas APIs

| Endpoint | Purpose | Scope Required |
|----------|---------|----------------|
| `conversations.canvases.create` | Create canvas attached to channel | `canvases:write` |
| `canvases.create` | Create standalone canvas | `canvases:write` |
| `canvases.edit` | Edit canvas content | `canvases:write` |
| `canvases.access.set` | Share canvas with channels/users | `canvases:write` |
| `canvases.access.delete` | Remove access to canvas | `canvases:write` |
| `canvases.sections.lookup` | Find sections in canvas | `canvases:read` |

### Canvas Content Format

Slack canvases support markdown-like formatting:
- Headers: `# H1`, `## H2`, `### H3`
- Lists: `- bullet`, `1. numbered`
- Checkboxes: `- [ ] unchecked`, `- [x] checked`
- Bold: `**text**`
- Italic: `*text*`
- Code: `` `inline` ``, ``` ```block``` ```
- Links: `[text](url)`
- Mentions: `<@U123456>`, `<#C123456>`

### Document Content Structure

```json
{
  "document_content": {
    "type": "markdown",
    "markdown": "# Title\n\nContent here..."
  }
}
```

### Edit Operations

Canvas edits use operations array with section targeting:
```json
{
  "canvas_id": "F123ABC",
  "changes": [
    {
      "operation": "replace",
      "section_id": "S123", 
      "document_content": {"type": "markdown", "markdown": "New content"}
    },
    {
      "operation": "insert_after",
      "section_id": "S456",
      "document_content": {"type": "markdown", "markdown": "Inserted content"}
    }
  ]
}
```

---

## Canvas Architecture Design

### Canvas Types

| Canvas | Location | Content | Sync Direction |
|--------|----------|---------|----------------|
| Master Dashboard | `{stub}-dev` channel | Methodology guide + live state | Git → Canvas |
| Focus Group Canvas | `{stub}-fg-{name}` channel | FG roadmap + concepts | Bidirectional |
| Implementation Dashboard | `{stub}-dev` channel | Queue + active + completed | Git → Canvas |
| Community Roadmap | `{stub}-community` channel | Sanitized public view | Git → Canvas |

### Canvas Content Sources

| Canvas Section | Source File | Sync Method |
|----------------|-------------|-------------|
| Methodology Guide | `templates/canvas/methodology.md` | Static template |
| Focus Groups List | `.planning/focus-groups/*/STATE.md` | Aggregate scan |
| Concepts List | `.planning/focus-groups/*/concepts/*.md` | Aggregate scan |
| Implementations | `.planning/active-implementations/*/STATE.md` | Aggregate scan |
| Community View | `.planning/MASTER-ROADMAP.md` | Filter private |

### Bidirectional Sync Strategy

**Git → Canvas (Primary):**
1. Detect git file changes (commit hook or manual trigger)
2. Parse changed `.planning/` files
3. Generate canvas markdown sections
4. Update canvas via `canvases.edit`

**Canvas → Git (Secondary, for concept discussions):**
1. AI monitors canvas edits (via events or polling)
2. Detect human edits → warn and revert
3. AI-initiated edits → sync to git files
4. Commit with attribution

### AI-Only Canvas Enforcement

**Strategy: Detect and Educate**
1. Canvas ID tracked in `.planning/CANVAS-REGISTRY.json`
2. AI is sole editor via API
3. If human edit detected, post message explaining policy
4. Revert to last AI state
5. Guide user to proper workflow (discuss in channel → AI updates)

---

## Canvas Templates

### Master Dashboard Template

```markdown
# {PROJECT} Development Dashboard

## 🎯 WGSD Methodology

**We Get Shit Done** - Social collaborative development:
- **Focus Groups**: Long-lived topic channels for ideation
- **Concepts**: Feature ideas matured through discussion
- **Implementations**: Short-lived execution (1-3 days max)

## 📊 Current State

### Active Focus Groups
{FOCUS_GROUPS_LIST}

### Active Implementations
{IMPLEMENTATIONS_LIST}

### Implementation Queue
{QUEUE_LIST}

## 🔄 Last Sync
{TIMESTAMP}

*This canvas is AI-managed. Discuss changes in channel, not here.*
```

### Focus Group Canvas Template

```markdown
# {FOCUS_GROUP_NAME} Focus Group

## 📋 Roadmap
{ROADMAP_FROM_MD}

## 💡 Concepts

### Ready for Implementation
{READY_CONCEPTS}

### In Development
{ACTIVE_CONCEPTS}

### Proposed
{PROPOSED_CONCEPTS}

## 📈 Recent Activity
{RECENT_CHANGES}

*AI-managed canvas. Discuss ideas in channel to add concepts.*
```

### Implementation Dashboard Template

```markdown
# Implementation Dashboard

## 🚀 Active ({COUNT}/{MAX})
{ACTIVE_IMPLEMENTATIONS}

## 📋 Queue
{QUEUED_IMPLEMENTATIONS}

## ✅ Recently Completed
{COMPLETED_IMPLEMENTATIONS}

## 📊 Velocity
- Avg implementation time: {AVG_TIME}
- Completed this week: {WEEKLY_COUNT}

*AI-managed. Use `/wgsd create-implementation` to start.*
```

### Community Roadmap Template

```markdown
# {PROJECT} Public Roadmap

## 🎯 What We're Building

{PUBLIC_SUMMARY}

## 📅 Current Focus Areas
{PUBLIC_FOCUS_AREAS}

## ✅ Recently Shipped
{PUBLIC_COMPLETIONS}

## 💬 Want to Contribute?
Share your ideas in this channel! Great suggestions may be promoted to our development process.

*Updated automatically by our development system.*
```

---

## Sync Implementation Design

### Canvas Registry File

Location: `.planning/CANVAS-REGISTRY.json`

```json
{
  "version": "1.0",
  "canvases": {
    "master-dashboard": {
      "canvas_id": "F123ABC",
      "channel_id": "C123DEV",
      "type": "master-dashboard",
      "last_sync": "2026-02-22T12:00:00Z",
      "checksum": "abc123"
    },
    "fg-security": {
      "canvas_id": "F456DEF",
      "channel_id": "C456FG",
      "type": "focus-group",
      "focus_group": "security",
      "last_sync": "2026-02-22T12:00:00Z",
      "checksum": "def456"
    }
  }
}
```

### Sync Trigger Points

| Trigger | Action |
|---------|--------|
| `wgsd sync-canvas` | Manual full sync |
| New focus group created | Create FG canvas |
| Concept state change | Update FG canvas |
| Implementation created/completed | Update dashboard |
| Git push to .planning/ | Auto-sync (future) |

---

## Wave Execution Strategy

### Wave 1: Canvas API Foundation
- Canvas API wrapper library extension
- Canvas registry system
- Template engine

### Wave 2: Master Dashboard + Implementation Dashboard
- CANVAS-01: Master dashboard creation
- CANVAS-08: Implementation dashboard
- INTEGRATE-05: API integration

### Wave 3: Focus Group Canvas + Sync
- CANVAS-04: Focus group canvas
- CANVAS-06: Bidirectional sync
- CANVAS-03: Auto-update on state change

### Wave 4: Community + AI Enforcement
- CANVAS-07: Community roadmap
- CANVAS-02: Live roadmap visualization
- CANVAS-05: AI-managed enforcement

---

## Risk Analysis

| Risk | Mitigation |
|------|------------|
| Canvas API rate limits | Batch updates, cache checksums |
| Human edits | Detect and educate, auto-revert |
| Sync conflicts | Git is source of truth |
| Large canvas content | Section-based updates |
| Canvas ID changes | Registry tracks all IDs |

---

## Dependencies

### From Phase 3 (Complete)
- ✅ `workflows/lib/slack-api.md` - Canvas creation function
- ✅ `workflows/lib/channel-registry.md` - Channel tracking
- ✅ Channel infrastructure for canvas attachment

### New for Phase 4
- Canvas template engine
- Canvas registry system
- Bidirectional sync workflow
- State aggregation from .planning/

---

*Research completed: 2026-02-22*
