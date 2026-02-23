# Phase 14: Slack-Native Approval Workflow - Execution Plan

**Created:** 2026-02-23  
**Phase:** 14 of 16  
**Priority:** P1 - High  
**Estimated Effort:** 2-3 hours  
**Dependencies:** Phase 12 (Matrix Approval) complete, Phase 13 (Roadmap Branch) complete

---

## Executive Summary

Phase 14 transforms the approval workflow from a technical matrix update system to a **conversational Slack-native experience**. Instead of formal PRs or command-heavy workflows, focus groups approve concepts through natural conversation in their channels.

### Vision

> "Focus groups should be able to discuss and approve concepts the same way they discuss any other topic - in their Slack channel. No GitHub PRs, no complex commands, just conversation."

### Key Outcomes

1. **Approval prompts appear naturally** in focus group channels
2. **Approvals happen via simple commands** or even thread responses
3. **Discussion threads** capture context and concerns
4. **Status summaries** keep everyone informed
5. **Rejection workflow** provides actionable feedback

---

## Architecture Overview

### Integration Points

```
Phase 11 (Impact Declaration)
    │
    ▼ Triggers notification to focus group channels
Phase 12 (Matrix Approval) ─────────────────────────┐
    │                                                │
    ▼                                                │
┌──────────────────────────────────────────────┐    │
│  Phase 14: Slack-Native Approval Workflow    │    │
│                                              │    │
│  ┌─────────────────────────────────────────┐│    │
│  │ SLACK-APPROVE-01: Approval Prompts      ││    │
│  │ - Rich message with context             ││    │
│  │ - Embedded approval actions             ││    │
│  │ - Links to concept details              ││    │
│  └─────────────────────────────────────────┘│    │
│                                              │    │
│  ┌─────────────────────────────────────────┐│    │
│  │ SLACK-APPROVE-02: Slash Commands        │◄────┤
│  │ - /wgsd approve <concept>               ││ Updates
│  │ - /wgsd block <concept>                 ││ Matrix
│  │ - /wgsd reject <concept>                ││    │
│  └─────────────────────────────────────────┘│    │
│                                              │    │
│  ┌─────────────────────────────────────────┐│    │
│  │ SLACK-APPROVE-03: Discussion Threads    ││    │
│  │ - Auto-create on prompt                 ││    │
│  │ - Capture concerns                      ││    │
│  │ - Link to concept author                ││    │
│  └─────────────────────────────────────────┘│    │
│                                              │    │
│  ┌─────────────────────────────────────────┐│    │
│  │ SLACK-APPROVE-04: Status Summary        ││    │
│  │ - Channel-level approval state          ││    │
│  │ - Highlight blockers                    ││    │
│  │ - Next action suggestions               ││    │
│  └─────────────────────────────────────────┘│    │
│                                              │    │
│  ┌─────────────────────────────────────────┐│    │
│  │ SLACK-APPROVE-05: Rejection Workflow    ││    │
│  │ - Require feedback reason               ││    │
│  │ - Notify concept author                 ││    │
│  │ - Re-review workflow                    ││    │
│  └─────────────────────────────────────────┘│    │
└──────────────────────────────────────────────┘    │
    │                                                │
    ▼ Full approval triggers                        │
Phase 13 (Auto-merge to Roadmap) ◄──────────────────┘
```

---

## Requirement Breakdown

### SLACK-APPROVE-01: Conversational Approval Prompts

**Goal:** Post rich, actionable approval requests in focus group channels

#### Tasks

1. **Design approval prompt message format**
   - Concept name and brief description
   - Impact priority and type for this focus group
   - Current approval matrix summary
   - Action buttons/commands
   - Link to full concept details

2. **Create approval prompt template**
   - Location: `workflows/lib/approval-templates.md` (NEW)
   - Rich Slack Block Kit formatting
   - Responsive to focus group context

3. **Integrate with impact notification system**
   - Modify: `workflows/lib/impact-notifications.md`
   - Replace simple notification with rich approval prompt
   - Include approval action hints

4. **Track prompt message IDs**
   - Store message `ts` in impact-matrix.md
   - Enable thread linking and updates

#### Acceptance Criteria
- [ ] Approval request posts to focus group channel automatically
- [ ] Message includes concept summary, impact, priority
- [ ] "Approve" / "Reject" / "Needs Discussion" commands shown
- [ ] Message updates when approval status changes

#### Files Affected
- `workflows/lib/approval-templates.md` (NEW)
- `workflows/lib/impact-notifications.md` (MODIFY)
- `templates/concept-directory/impact-matrix.md` (ADD: prompt_message_ts field)

---

### SLACK-APPROVE-02: Approval via Slash Command

**Goal:** Enable quick approvals with minimal friction

#### Tasks

1. **Implement conversational approve command**
   - Location: `workflows/conversational-approve.md` (NEW)
   - Command: `/wgsd approve <concept>` (simplest form)
   - Infer focus group from current channel context
   - Validate user authority

2. **Add quick approval variations**
   - `/wgsd approve <concept>` - Approve from current FG channel
   - `/wgsd approve <concept> --comment "LGTM"` - With comment
   - `/wgsd approve <concept> --fg security` - Explicit FG

3. **Support reaction-based approval**
   - ✅ or 👍 on approval prompt → triggers approval
   - Requires storing prompt message reference

4. **Support thread-based approval**
   - "approved", "lgtm", "+1" in discussion thread
   - Links to matrix-approve.md functions

5. **Provide immediate feedback**
   - Ephemeral confirmation message
   - Update original approval prompt
   - Update concept canvas

#### Acceptance Criteria
- [ ] `/wgsd approve {concept}` command works from FG channel
- [ ] User authority validated (must be FG member)
- [ ] Approval matrix updated
- [ ] Confirmation posted in channel

#### Files Affected
- `workflows/conversational-approve.md` (NEW)
- `workflows/matrix-approve.md` (ADD: channel context inference)
- `SKILL.md` (ADD: routing for approve command)

---

### SLACK-APPROVE-03: Discussion Thread for Approval

**Goal:** Enable structured discussion before approval decisions

#### Tasks

1. **Auto-create discussion thread**
   - Thread starts from approval prompt message
   - Thread header includes concept context
   - Concept author notified of thread creation

2. **Discussion thread tracking**
   - Track thread `ts` in impact-matrix.md
   - Enable `/wgsd discuss <concept>` to link to thread

3. **Cross-link concept author**
   - Notify author when discussion starts
   - Enable author to respond in thread
   - Tag author on questions/concerns

4. **Discussion summary extraction**
   - Extract key concerns from thread
   - Surface unresolved questions
   - Link to approval decision

#### Acceptance Criteria
- [ ] Approval prompt creates or links to discussion thread
- [ ] Discussion visible to concept author
- [ ] Author can respond to concerns
- [ ] Key discussion points captured

#### Files Affected
- `workflows/lib/approval-templates.md` (ADD: thread management)
- `workflows/lib/slack-api.md` (ADD: thread operations)
- `templates/concept-directory/impact-matrix.md` (ADD: discussion_thread_ts)

---

### SLACK-APPROVE-04: Approval Status Summary

**Goal:** Provide at-a-glance approval status in channels

#### Tasks

1. **Create status command**
   - Location: `workflows/concept-status.md` (NEW)
   - Command: `/wgsd status <concept>`
   - Show approval matrix with visual indicators

2. **Design status summary format**
   - Color-coded status (green/yellow/red)
   - Progress bar for completion
   - Highlight pending and blocked approvals
   - Suggest next actions

3. **Enable channel-specific view**
   - When run from FG channel, highlight that FG's status
   - Show relationship to other FG approvals

4. **Add periodic status updates**
   - Optional: Daily summary of pending approvals
   - Stale approval reminders

#### Acceptance Criteria
- [ ] `/wgsd status {concept}` shows approval matrix
- [ ] Highlights pending and blocked approvals
- [ ] Suggests next actions
- [ ] Works from any channel with concept context

#### Files Affected
- `workflows/concept-status.md` (NEW)
- `workflows/lib/approval-templates.md` (ADD: status template)
- `SKILL.md` (ADD: routing for status command)

---

### SLACK-APPROVE-05: Rejection with Feedback

**Goal:** Make rejections constructive with required feedback

#### Tasks

1. **Implement rejection command**
   - Location: `workflows/conversational-reject.md` (NEW)
   - Command: `/wgsd reject <concept> "reason"`
   - Reason is REQUIRED (not optional)

2. **Notify concept author**
   - Post rejection to dev channel
   - Direct message to concept author
   - Include specific feedback

3. **Update concept status**
   - Return concept to draft/needs-revision status
   - Clear any approval progress
   - Preserve rejection history

4. **Enable re-review workflow**
   - `/wgsd request-review <concept> --fg <fg>`
   - Author can address feedback and re-request
   - Re-request notification to rejecting FG

5. **Track rejection history**
   - Store rejections in impact-matrix.md
   - Show rejection history in status

#### Acceptance Criteria
- [ ] Rejection requires feedback reason
- [ ] Reason posted to dev channel
- [ ] Concept author notified
- [ ] Concept returns to draft/needs-revision status
- [ ] Re-review workflow available

#### Files Affected
- `workflows/conversational-reject.md` (NEW)
- `workflows/matrix-approve.md` (MODIFY: rejection handling)
- `templates/concept-directory/impact-matrix.md` (ADD: rejection history)
- `SKILL.md` (ADD: routing for reject command)

---

## New Files Summary

| File | Purpose |
|------|---------|
| `workflows/lib/approval-templates.md` | Rich Slack message templates for approvals |
| `workflows/conversational-approve.md` | Slack-native approval command workflow |
| `workflows/concept-status.md` | Approval status summary command |
| `workflows/conversational-reject.md` | Rejection workflow with feedback |

## Modified Files Summary

| File | Changes |
|------|---------|
| `workflows/lib/impact-notifications.md` | Replace basic notification with rich prompt |
| `workflows/lib/slack-api.md` | Add thread operations |
| `workflows/matrix-approve.md` | Add channel context inference |
| `templates/concept-directory/impact-matrix.md` | Add prompt_ts, thread_ts, rejection history |
| `SKILL.md` | Add routing for new commands |

---

## Execution Order

```
1. SLACK-APPROVE-01 (Approval Prompts)
   ├── Create approval-templates.md
   └── Modify impact-notifications.md
        │
2. SLACK-APPROVE-02 (Slash Commands)
   ├── Create conversational-approve.md  
   └── Modify matrix-approve.md
        │
3. SLACK-APPROVE-03 (Discussion Threads)
   ├── Add thread management to templates
   └── Add thread operations to slack-api.md
        │
4. SLACK-APPROVE-04 (Status Summary)
   └── Create concept-status.md
        │
5. SLACK-APPROVE-05 (Rejection Workflow)
   ├── Create conversational-reject.md
   └── Update impact-matrix template
```

---

## Testing Strategy

### Unit Tests
- Approval prompt message generation
- User authority validation
- Channel context inference
- Rejection reason validation

### Integration Tests
1. **End-to-end approval flow**
   - Declare impact → Prompt appears in FG channel
   - Approve via command → Matrix updates
   - Full approval → Auto-merge triggered

2. **Discussion thread flow**
   - Start discussion → Thread created
   - Author notified → Can respond
   - Approve after discussion → Thread archived

3. **Rejection flow**
   - Reject with reason → Author notified
   - Author addresses → Request re-review
   - Re-approve → Normal flow continues

### Manual Verification
- [ ] Approval prompt renders correctly in Slack
- [ ] Commands work from mobile Slack
- [ ] Thread navigation intuitive
- [ ] Status summary readable at a glance

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Approval prompt visibility | 100% of impacts notify FG channels |
| Approval completion rate | Improve time-to-approval by 50% |
| Discussion thread usage | >30% of approvals have discussion |
| Rejection feedback quality | 100% of rejections include reason |

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Slack API rate limits | Delayed notifications | Batch notifications, use webhooks |
| Channel permission issues | Missing notifications | Validate bot membership on channel creation |
| Thread navigation confusion | User friction | Clear thread naming, link back to main message |
| Approval spam | Channel noise | Consolidate multi-FG approvals, use threads |

---

## Definition of Done

- [x] All 5 requirements implemented
- [x] New workflow files created and documented
- [x] SKILL.md routing updated
- [x] impact-matrix.md template enhanced
- [ ] Integration test with multi-FG concept approval
- [ ] Canvas approval widget reflects Slack approvals
- [x] Full approval triggers Phase 13 roadmap merge

---

*Execution plan for WGSD Phase 14 — Slack-Native Approval Workflow*
