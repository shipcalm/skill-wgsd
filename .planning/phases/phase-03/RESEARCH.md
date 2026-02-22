# Phase 3: Channel Infrastructure - Research

**Created:** 2026-02-22
**Phase:** 3 - Channel Infrastructure
**Requirements:** CHANNEL-03, 04, 05, 06, 07, 08, INTEGRATE-04

---

## Domain Research

### Slack API Channel Operations

**Key APIs Required:**

| API Endpoint | Purpose | Scope Required |
|--------------|---------|----------------|
| `conversations.create` | Create channels | `channels:manage`, `groups:write` |
| `conversations.archive` | Archive channel | `channels:manage`, `groups:write` |
| `conversations.unarchive` | Restore channel | `channels:manage`, `groups:write` |
| `conversations.join` | Bot joins channel | `channels:join` |
| `conversations.invite` | Invite users | `channels:manage`, `groups:write` |
| `conversations.setTopic` | Set channel topic | `channels:manage`, `groups:write` |
| `conversations.setPurpose` | Set channel purpose | `channels:manage`, `groups:write` |
| `conversations.list` | List channels | `channels:read`, `groups:read` |
| `conversations.info` | Get channel info | `channels:read`, `groups:read` |
| `conversations.canvases.create` | Create channel canvas | `channels:write.topic` |

### Private vs Public Channel Creation

```bash
# Public channel
curl -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "mvn-community", "is_private": false}'

# Private channel  
curl -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "mvn-dev", "is_private": true}'
```

### Channel Types and Privacy

| Type | Pattern | Private | Use Case |
|------|---------|---------|----------|
| dev | `{stub}-dev` | **Yes** | Core development team discussions |
| community | `{stub}-community` | **No** | Public customer/user feedback |
| fg (focus group) | `{stub}-fg-{name}` | **Yes** | Topic-specific development |
| cpt (concept) | `{stub}-cpt-{name}` | **Yes** | Feature ideation |
| impl (implementation) | `{stub}-impl-{name}` | **Yes** | Short-lived execution |

### Channel Lifecycle States

```
┌─────────┐     ┌────────┐     ┌──────────┐
│ Created │ ──► │ Active │ ──► │ Archived │
└─────────┘     └────────┘     └──────────┘
                    │               │
                    │               ▼
                    │          ┌──────────┐
                    └─────────►│ Restored │
                               └──────────┘
```

### OpenClaw Channel Registration

When a channel is created, it must be registered in OpenClaw config:
- `allow: true` - Enable monitoring
- `requireMention: false` - Auto-respond without @mention

---

## Existing Assets Analysis

### Current `create-channel.md`
- ✅ Basic channel creation via Slack API
- ✅ Channel name validation via naming.md library
- ✅ Topic setting
- ⚠️ Missing: is_private parameter (defaults to public)
- ⚠️ Missing: Channel type-specific workflows
- ⚠️ Missing: Lifecycle management

### Current `naming.md` Library
- ✅ Full naming validation and generation
- ✅ Channel type detection (dev, community, fg, cpt, impl)
- ✅ Auto-correction suggestions
- ✅ Stub validation (2-5 lowercase letters)

---

## Architecture Decisions

### 1. Channel Registry Location

**Decision:** Store channel registry in `.planning/WGSD-CONFIG.md` per project

```yaml
channels:
  mvn-dev:
    id: C0123456789
    type: dev
    created: 2026-02-22
    status: active
  mvn-fg-security:
    id: C0234567890
    type: fg
    created: 2026-02-22
    status: active
  mvn-impl-auth-v2:
    id: C0345678901
    type: impl
    created: 2026-02-22
    status: archived
    archived_at: 2026-02-25
```

### 2. Slack API Library Structure

**Decision:** Create `workflows/lib/slack-api.md` with reusable functions:
- `slack_create_channel()` - Create with private/public support
- `slack_archive_channel()` - Archive channel
- `slack_unarchive_channel()` - Restore archived channel
- `slack_invite_bot()` - Ensure bot is in channel
- `slack_get_channel_info()` - Get channel details
- `slack_list_channels()` - List all channels

### 3. Channel Type Handlers

**Decision:** Create type-specific workflows that call core library:
- `workflows/create-dev-channel.md` - Core dev channel
- `workflows/create-community-channel.md` - Public community
- `workflows/create-focus-group.md` - Focus group (enhanced)
- `workflows/create-concept.md` - Concept channel (enhanced)
- `workflows/create-implementation.md` - Implementation (enhanced)
- `workflows/archive-channel.md` - Lifecycle management

---

## Wave Execution Plan

### Wave 1: Core Infrastructure
- `workflows/lib/slack-api.md` - Slack API wrapper library
- `workflows/lib/channel-registry.md` - Channel tracking

### Wave 2: Channel Types
- Enhanced `workflows/create-channel.md` - Full private/public support
- Specific type handlers integrated into existing workflows

### Wave 3: Lifecycle Management
- `workflows/archive-channel.md` - Archive/restore/cleanup
- Registry synchronization with Slack state

---

## Success Criteria Mapping

| Requirement | Success Criteria | Verification Method |
|-------------|------------------|---------------------|
| CHANNEL-03 | `wgsd init` creates #mvn-dev as private | Check channel is_private=true |
| CHANNEL-04 | `wgsd init` creates #mvn-community as public | Check channel is_private=false |
| CHANNEL-05 | Focus group creation works | `wgsd create-focus-group security` creates private channel |
| CHANNEL-06 | Concept creation works | `wgsd create-concept byof` creates private channel |
| CHANNEL-07 | Implementation creation works | `wgsd create-implementation auth-v2` creates private channel |
| CHANNEL-08 | Lifecycle management works | Archive/restore commands function correctly |
| INTEGRATE-04 | Slack API integration reliable | All API calls succeed with proper error handling |

---

*Research complete - ready for execution planning*
