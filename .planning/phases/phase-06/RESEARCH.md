# Phase 6: Community Integration - Research

**Date:** 2026-02-22
**Phase:** 6 - Community Integration (FINAL PHASE)
**Requirements:** 7 (COMMUNITY-01 through COMMUNITY-07)

---

## Domain Analysis

### Public → Private Development Pipeline

WGSD maintains a security boundary between public community discussions and private development channels. The community integration phase creates controlled bridges between these domains.

**Security Model:**
```
┌─────────────────────────────────────────────────────────────┐
│                    PUBLIC DOMAIN                             │
│  {stub}-community channel                                    │
│  - Customer feedback                                         │
│  - Feature requests                                          │
│  - Bug reports                                               │
│  - General discussion                                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ Moderation Triage
                      │ (COMMUNITY-02)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    PRIVATE DOMAIN                            │
│  - Focus group channels                                      │
│  - Concept channels                                          │
│  - Implementation channels                                   │
│  - Core dev channel                                          │
└─────────────────────────────────────────────────────────────┘
```

### Slack User Types

Understanding Slack user types for invitation management:

| Type | Can Join Public | Can Join Private | Method |
|------|-----------------|------------------|--------|
| Workspace Member | ✅ Yes | ✅ Via invite | conversations.invite |
| Guest (Multi-Channel) | ✅ Specific | ✅ Via invite | conversations.invite |
| Guest (Single-Channel) | ✅ One only | ❌ No | Cannot change |
| External (Connect) | ✅ Shared | ❌ No | Slack Connect |

**Strategy:** Community contributors are workspace members or multi-channel guests.

---

## Community Feedback Collection (COMMUNITY-01)

### Feedback Categories

```yaml
feedback_types:
  feature_request:
    emoji: "💡"
    description: "New feature idea"
    triage_to: "appropriate focus group"
    
  bug_report:
    emoji: "🐛"
    description: "Something isn't working"
    triage_to: "core-dev or relevant focus group"
    
  improvement:
    emoji: "✨"
    description: "Enhancement to existing feature"
    triage_to: "relevant focus group"
    
  question:
    emoji: "❓"
    description: "How does X work?"
    triage_to: "documentation or community response"
    
  praise:
    emoji: "🎉"
    description: "Positive feedback"
    triage_to: "share internally, no action"
```

### Collection Methods

1. **Emoji Reaction Tagging**
   - Team reacts to messages with category emojis
   - Bot detects reactions, adds to triage queue
   
2. **Thread Commands**
   - `/wgsd feedback [type]` in thread
   - Records message as feedback with metadata
   
3. **Message Forwarding**
   - Forward message to intake channel
   - Bot processes and categorizes

### Feedback Record Structure

```markdown
# Feedback: {id}

**Type:** feature_request
**Author:** @community_member
**Channel:** #mvn-community
**Timestamp:** 2026-02-22T10:00:00Z
**Message Link:** https://slack.com/archives/...

## Original Message
> I wish the app could do X because Y

## Triage Status
- [ ] Reviewed
- [ ] Assigned to focus group: _______
- [ ] Contributor invited: [ ] Yes [ ] No

## Attribution
Original proposer: @community_member (community)
```

---

## Moderation Triage (COMMUNITY-02)

### Triage Workflow

```
1. Feedback identified in community channel
2. Moderator reviews and categorizes
3. Move command executed:
   /wgsd move-to-focus-group [message-link] [focus-group]
4. System:
   a. Creates concept from feedback
   b. Records original author
   c. Optionally invites author to focus group
5. Community notified of triage result
```

### Focus Group Mapping

Smart routing based on content analysis:

| Keywords | Focus Group |
|----------|-------------|
| auth, login, sso, password, 2fa | security |
| signup, onboard, getting started, tutorial | onboarding |
| payment, invoice, billing, subscription | billing |
| api, integration, webhook, rest | api |
| performance, speed, slow, fast | performance |

### Triage Decision Record

```markdown
# Triage Decision

**Feedback ID:** FB-2026-0001
**Decision:** Move to focus group
**Focus Group:** security
**Reason:** SSO feature request
**Moderator:** @alice
**Timestamp:** 2026-02-22T10:30:00Z

**Actions Taken:**
- [x] Created concept: sso-integration
- [x] Invited contributor: @community_member
- [x] Posted acknowledgment in community
```

---

## Contributor Invitation (COMMUNITY-03)

### Invitation Criteria

**Auto-invite when:**
- Feedback becomes a concept
- Significant community discussion on topic
- Explicit request to participate

**Don't auto-invite when:**
- Simple bug report (fix doesn't need input)
- Question already answered
- User previously declined invitation

### Invitation Message Template

```markdown
Hi @{username}! 👋

Your feedback in #{community_channel} caught our attention:
> {original_message_preview}

We've started developing this idea in our **{focus_group}** focus group, and we'd love to have you contribute to the discussion!

You've been invited to **#{focus_group_channel}** where the team is exploring this concept.

**What to expect:**
- Early access to development discussions
- Opportunity to shape the feature
- Attribution in our roadmap

[Join #{focus_group_channel}] or reply "no thanks" if you'd prefer not to participate.

Thanks for making {project} better! 🙏
```

### Invitation Tracking

```yaml
contributor_invitation:
  user_id: "U12345678"
  username: "community_member"
  feedback_id: "FB-2026-0001"
  focus_group: "security"
  channel_id: "C12345678"
  invited_at: "2026-02-22T10:30:00Z"
  status: "pending"  # pending, accepted, declined
  response_at: null
```

---

## Access Request System (COMMUNITY-04)

### Request Flow

```
1. Community member requests access:
   /wgsd request-access [focus-group]
   
2. System creates access request:
   - Records requester info
   - Posts to focus group for review
   
3. Focus group owner reviews:
   /wgsd approve-access [request-id]
   /wgsd deny-access [request-id] [reason]
   
4. System processes decision:
   - If approved: invite to channel
   - If denied: notify with reason
```

### Request Criteria Form

```markdown
# Access Request

**Requester:** @{username}
**Focus Group:** {focus_group}
**Requested:** {timestamp}

## Why are you interested?
{user_provided_reason}

## Relevant Background
{user_provided_background}

## Community Contributions
- Messages in #{community}: {count}
- Feedback submitted: {count}
- Ideas adopted: {count}

## Decision
- [ ] Approve
- [ ] Deny

**Reviewer:** ___________
**Decision Date:** ___________
**Notes:** ___________
```

---

## Attribution System (COMMUNITY-05)

### Attribution Display

In focus group Canvas:

```markdown
## Contributors

### Core Team
- @alice (owner)
- @bob
- @charlie

### Community Contributors
- @community_member - Originally proposed SSO integration (FB-2026-0001)
- @another_user - Contributed billing workflow ideas (FB-2026-0015)
```

### Attribution Records

```yaml
attributions:
  - user_id: "U12345678"
    type: "community"
    contribution: "original_idea"
    concept: "sso-integration"
    feedback_id: "FB-2026-0001"
    attributed_at: "2026-02-22T10:30:00Z"
```

---

## Moderation Commands (COMMUNITY-06)

### Command Reference

| Command | Description | Example |
|---------|-------------|---------|
| move-to-focus-group | Move community message to focus group | `/wgsd move-to-focus-group https://... security` |
| invite-contributor | Invite community member to channel | `/wgsd invite-contributor @user security` |
| approve-access | Approve access request | `/wgsd approve-access REQ-001` |
| deny-access | Deny access request | `/wgsd deny-access REQ-001 "not relevant"` |
| attribute | Add contributor attribution | `/wgsd attribute @user sso-concept "Original idea"` |
| community-status | Show community engagement stats | `/wgsd community-status` |

### Permission Model

| Command | Who Can Execute |
|---------|-----------------|
| move-to-focus-group | Focus group owners, moderators |
| invite-contributor | Focus group owners, moderators |
| approve-access | Focus group owners only |
| deny-access | Focus group owners only |
| attribute | Focus group owners, moderators |
| community-status | Anyone in dev channel |

---

## Transparent Development Visibility (COMMUNITY-07)

### What Community Sees

**In Community Canvas:**
```markdown
# Development Roadmap

## Active Focus Groups
| Focus Group | Active Concepts | Status |
|-------------|-----------------|--------|
| Security | 3 | 🟢 Active |
| Onboarding | 2 | 🟢 Active |
| Billing | 1 | 🟡 Planning |

## In Development
| Feature | Focus Group | Progress | Community Origin |
|---------|-------------|----------|------------------|
| SSO Integration | Security | 60% | ✅ @user1 |
| Onboarding Wizard | Onboarding | 30% | ✅ @user2 |

## Recently Shipped
- **Auth improvements** - Shipped 2026-02-15
- **Billing dashboard** - Shipped 2026-02-10

## How to Contribute
Post your ideas in this channel! Our team reviews all feedback
and the best ideas get developed with your input.
```

### Sanitization Rules

**Shown to community:**
- Feature names and descriptions
- Progress percentages
- Community contributor attributions
- General status indicators

**Hidden from community:**
- Internal discussions
- Technical implementation details
- Security-sensitive information
- Unreleased feature specifics
- Team member comments

---

## Deliverables

| File | Purpose | Requirements |
|------|---------|--------------|
| workflows/moderate-feedback.md | Feedback collection + triage | COMMUNITY-01, 02, 06 |
| workflows/invite-contributor.md | Contributor invitation flow | COMMUNITY-03, 06 |
| workflows/access-request.md | Access request system | COMMUNITY-04 |
| workflows/community-visibility.md | Transparent visibility + attribution | COMMUNITY-05, 07 |

---

*Research completed: 2026-02-22*
