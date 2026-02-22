# WGSD Migration Announcement Template

This template generates team communication for GSD → WGSD migrations.

---

## Usage

The migration wizard automatically generates this announcement at:
`.planning/MIGRATION-ANNOUNCEMENT.md`

You can also generate manually using this template.

---

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{project_name}` | Project display name | Marvin |
| `{stub}` | Slack channel stub | mvn |
| `{focus_groups}` | List of focus groups | core, security, api |
| `{dev_channel}` | Main dev channel | #mvn-dev |
| `{migration_date}` | Date of migration | 2026-02-22 |
| `{active_impl}` | Active implementation (if any) | mvn-impl-auth |

---

## Full Template

```markdown
# 🚀 {project_name} has migrated to WGSD!

Hey team! We've upgraded our development workflow from GSD to **WGSD (We Get Shit Done)** v2.0.

## What's WGSD?

WGSD is a social development methodology that transforms how we collaborate:

- **Focus Groups** - Long-lived topic discussions (like working groups)
- **Concepts** - Feature ideas developed socially before any code
- **Implementations** - Short-lived coding sprints (1-3 days max)

## Why the Change?

- 🗣️ **More collaboration** - Ideas develop through discussion, not isolation
- 🎯 **Focused work** - One implementation at a time per focus group
- 🔄 **Faster iteration** - Small, quick implementations instead of long phases
- 👥 **Community input** - Public channel for user feedback

## New Channels

| Channel | Purpose |
|---------|---------|
| #{stub}-dev | Main development channel - daily standups, announcements |
| #{stub}-community | Public feedback from users (read-only for most) |
{focus_group_channels}
{implementation_channels}

## How It Works

```
🌐 Community Feedback    💬 Focus Group Discussion
         ↓                         ↓
    #{stub}-community  →    #{stub}-fg-{name}
                                   ↓
                              💡 Concept
                                   ↓
                         👍 Focus Group Approval
                                   ↓
                           🔧 Implementation
                                   ↓
                              📦 Merged!
```

## How to Contribute

### Have an idea?
1. Share it in the relevant focus group channel (#{stub}-fg-*)
2. We'll discuss it as a team
3. Good ideas become concepts

### Want to implement something?
1. Pick up an approved concept
2. Claim it by creating an implementation
3. Build it in 1-3 days, merge to develop

### Found a bug or have feedback?
Post in #{stub}-community (if you have access) or #{stub}-dev

## What Happened to Phases?

GSD phases have been converted to WGSD focus groups:
{phase_to_fg_mapping}

Any work-in-progress has been preserved as an implementation:
{wip_preservation}

## Questions?

- Check the Canvas in #{stub}-dev for methodology guide
- Ask in #{stub}-dev - we're all learning together
- Read .planning/WGSD-CONFIG.md for configuration details

---

🙌 Let's get shit done!

*Migrated on {migration_date}*
```

---

## Short Version (Slack)

For posting directly to Slack:

```
🚀 *{project_name} has migrated to WGSD!*

We've upgraded from GSD to WGSD (We Get Shit Done) v2.0.

*What's new:*
• Focus Groups for ongoing discussions
• Concepts for feature planning
• Implementations for quick coding sprints

*New channels:*
• #{stub}-dev - Main development
• #{stub}-community - Public feedback
{focus_group_list}

Check the pinned message in #{stub}-dev for the full guide!
```

---

## Email Version

For stakeholder communication:

```
Subject: {project_name} Development Workflow Update

Hi team,

We've upgraded {project_name}'s development workflow to WGSD 
(We Get Shit Done) v2.0.

KEY CHANGES:
- Phases are now Focus Groups (ongoing discussions)
- Features start as Concepts (social planning)
- Coding happens in Implementations (1-3 day sprints)

BENEFITS:
- More collaborative planning process
- Faster, more focused implementations
- Better visibility into development progress
- Community feedback integration

ACTION REQUIRED:
- Join the new #{stub}-dev Slack channel
- Review your focus group assignments
- Check the Canvas for the methodology guide

Questions? Reply to this email or ask in #{stub}-dev.

Best,
{sender_name}
```

---

## Generator Function

```bash
# Usage: generate_announcement <project_name> <stub> <focus_groups> [active_impl]
generate_announcement() {
  local project_name="$1"
  local stub="$2"
  local focus_groups="$3"
  local active_impl="${4:-}"
  local migration_date=$(date -I)
  
  cat << EOF
# 🚀 $project_name has migrated to WGSD!

Hey team! We've upgraded our development workflow from GSD to **WGSD (We Get Shit Done)** v2.0.

## What Changed

- **Focus Groups** replace phases for ongoing discussions
- **Concepts** are ideas we develop socially before coding
- **Implementations** are short-lived coding sprints (1-3 days)

## New Channels

| Channel | Purpose |
|---------|---------|
| #${stub}-dev | Main development channel |
$(for fg in $focus_groups; do
  echo "| #${stub}-fg-${fg} | ${fg^} focus group |"
done)
$([ -n "$active_impl" ] && echo "| #${active_impl} | Active implementation |")

## How to Contribute

1. **Have an idea?** Share it in the relevant focus group channel
2. **Want to build something?** Create a concept for discussion
3. **Ready to code?** Promote your concept to an implementation

## Questions?

Drop a message in #${stub}-dev and we'll help you get started!

---

*Migrated on $migration_date*
EOF
}
```

---

*Template created for WGSD Phase 2 - MIGRATE-08*
