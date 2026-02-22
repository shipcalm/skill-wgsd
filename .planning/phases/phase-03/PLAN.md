# Phase 3: Channel Infrastructure - Execution Plan

**Created:** 2026-02-22
**Phase:** 3 - Channel Infrastructure
**Requirements:** 7 (CHANNEL-03, 04, 05, 06, 07, 08, INTEGRATE-04)

---

## Execution Strategy

3 waves, building infrastructure → types → lifecycle.

---

## Wave 1: Core Infrastructure

### Deliverable 1: Slack API Library
**File:** `workflows/lib/slack-api.md`
**Lines:** ~350
**Requirements:** INTEGRATE-04

Functions:
- `slack_get_token()` - Get bot token from OpenClaw config
- `slack_create_channel(name, is_private, topic)` - Create channel
- `slack_archive_channel(channel_id)` - Archive channel
- `slack_unarchive_channel(channel_id)` - Restore channel
- `slack_invite_users(channel_id, user_ids)` - Invite users
- `slack_get_channel(channel_id)` - Get channel info
- `slack_list_channels(types)` - List all channels
- `slack_set_topic(channel_id, topic)` - Set topic
- `slack_set_purpose(channel_id, purpose)` - Set purpose

### Deliverable 2: Channel Registry Library
**File:** `workflows/lib/channel-registry.md`
**Lines:** ~200
**Requirements:** CHANNEL-08 (foundation)

Functions:
- `registry_add_channel(stub, type, name, id)` - Add to registry
- `registry_remove_channel(id)` - Remove from registry
- `registry_get_channel(type, name)` - Get channel by type/name
- `registry_list_channels(stub, type)` - List channels
- `registry_update_status(id, status)` - Update status
- `registry_sync_from_slack()` - Sync with actual Slack state

---

## Wave 2: Channel Type Handlers

### Deliverable 3: Enhanced create-channel.md
**File:** `workflows/create-channel.md`
**Lines:** ~400 (enhance existing)
**Requirements:** CHANNEL-03, 04, 05, 06, 07

Enhancements:
- Add `is_private` parameter based on channel type
- Auto-detect type from channel name
- Integrate with registry library
- Add OpenClaw config registration
- Standard topic generation per type

### Deliverable 4: Core Channel Setup
**File:** `workflows/setup-core-channels.md`
**Lines:** ~200
**Requirements:** CHANNEL-03, 04

Creates both:
- `{stub}-dev` (private) - Core development
- `{stub}-community` (public) - Community feedback

---

## Wave 3: Lifecycle Management

### Deliverable 5: Archive Channel Workflow
**File:** `workflows/archive-channel.md`
**Lines:** ~250
**Requirements:** CHANNEL-08

Operations:
- Archive channel with state preservation
- Update registry status
- Option to delete vs archive
- Archive reason tracking

### Deliverable 6: Restore Channel Workflow
**File:** `workflows/restore-channel.md`
**Lines:** ~150
**Requirements:** CHANNEL-08

Operations:
- Unarchive Slack channel
- Update registry status
- Re-register with OpenClaw

---

## Dependency Graph

```
Wave 1: slack-api.md + channel-registry.md
           │
           ▼
Wave 2: create-channel.md (enhanced) + setup-core-channels.md
           │
           ▼
Wave 3: archive-channel.md + restore-channel.md
```

---

## Estimated Effort

| Wave | Files | Est. Lines | Duration |
|------|-------|------------|----------|
| 1 | 2 | ~550 | Core infrastructure |
| 2 | 2 | ~600 | Channel creation |
| 3 | 2 | ~400 | Lifecycle mgmt |
| **Total** | **6** | **~1550** | |

---

*Plan ready for execution*
