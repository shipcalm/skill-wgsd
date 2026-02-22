---
name: wgsd:create-focus-group  
description: Create focus group with channel, canvas, git branch, and planning structure
argument-hint: "[focus-group-name]"
allowed-tools:
  - Read
  - Write  
  - Bash
  - Message
  - Canvas
---

<objective>
Create a new focus group for collaborative development with:
- Slack channel for team discussion
- Canvas with live roadmap (syncs with git)
- Git branch for planning and concepts
- Worktree directory for isolated work
- Planning structure integrated with shared .planning/
</objective>

<process>

## Step 1: Validate Setup

```bash
if [ ! -f .planning/WGSD-CONFIG.md ]; then
    echo "❌ WGSD not initialized. Run: /wgsd setup-repo first"
    exit 1
fi

# Get repository name for channel naming
REPO_NAME=$(basename $(pwd))
FOCUS_GROUP="{focus-group-name}"
CHANNEL_NAME="${REPO_NAME}-${FOCUS_GROUP}"

echo "🚀 Creating focus group: ${FOCUS_GROUP}"
echo "📱 Channel: #${CHANNEL_NAME}" 
```

## Step 2: Create Planning Structure

```bash
mkdir -p .planning/focus-groups/${FOCUS_GROUP}/concepts
```

Create focus group files:

**.planning/focus-groups/{focus-group}/ROADMAP.md:**
```markdown
# {Focus Group} Roadmap

**Focus Group:** {Focus Group Name}
**Channel:** #{repo-name}-{focus-group}
**Created:** {date}
**Status:** Active

## Vision

{What this focus group aims to achieve}

## Current Concepts

*No concepts yet - use `/wgsd create-concept` to add ideas*

## Concept Pipeline

### 📝 Draft
*Ideas being captured and initially discussed*

### 🔍 Exploring  
*Concepts being researched and refined*

### ✅ Mature
*Well-defined concepts ready for implementation consideration*

### 🚀 Ready for Implementation
*Concepts approved and queued for implementation*

## Recent Activity

- {date}: Focus group created
- *Activity will be logged automatically*

---

**Canvas Sync:** This file syncs bidirectionally with the #{channel-name} canvas.
Edit either location - changes will be synchronized automatically.
```

**.planning/focus-groups/{focus-group}/STATE.md:**
```markdown
# {Focus Group} State

**Last Updated:** {date}
**Active Concepts:** 0
**Implementation Queue:** 0

## Current Activity

*No activity yet*

## Team Members

*Will be populated as members join channel*

## Next Actions

- Define focus group vision and scope
- Create first concept with `/wgsd create-concept`
- Begin social discussion of ideas

---

*Use `/wgsd status` for cross-focus-group overview*
```

**.planning/focus-groups/{focus-group}/CHANNELS.md:**
```markdown
# {Focus Group} Channel Mappings

**Primary Channel:** #{repo-name}-{focus-group}
**Canvas:** {Focus Group} Roadmap (syncs with ROADMAP.md)

## Channel Purpose

Long-lived social discussion channel for {focus-group} related concepts and ideas.

**Channel Guidelines:**
- Social discussion encouraged - ideas flow freely
- Use `/wgsd create-concept` to capture mature ideas
- Reference other focus groups via shared .planning/ files
- Jarvis doesn't require mentions in this channel

## Related Channels

- **#{repo-name}-dev**: Main development community hub
- **Implementation channels**: Created when concepts move to implementation

---

*This channel is part of the WGSD collaborative development workflow*
```

## Step 3: Create Git Branch and Worktree

```bash
# Create focus group branch off the base branch
git checkout focus-groups/base
git checkout -b focus-groups/${FOCUS_GROUP}

# Add initial planning files to the branch
git add .planning/focus-groups/${FOCUS_GROUP}/
git commit -m "feat: create ${FOCUS_GROUP} focus group structure"
git push -u origin focus-groups/${FOCUS_GROUP}

# Create worktree for isolated focus group work
git worktree add concepts/${FOCUS_GROUP} focus-groups/${FOCUS_GROUP}

echo "✅ Git structure created:"
echo "   Branch: focus-groups/${FOCUS_GROUP}" 
echo "   Worktree: concepts/${FOCUS_GROUP}/"
```

## Step 4: Create Slack Channel

```bash
# Create Slack channel using OpenClaw Slack token
SLACK_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.channels.slack.botToken')
CHANNEL_TOPIC="Collaborative development for ${FOCUS_GROUP} concepts and ideas. Part of WGSD workflow."

# Create channel via Slack API
CHANNEL_RESPONSE=$(curl -s -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"${CHANNEL_NAME}\",\"is_private\":false}")

CHANNEL_ID=$(echo "$CHANNEL_RESPONSE" | jq -r '.channel.id')

if [ "$CHANNEL_ID" != "null" ]; then
    echo "✅ Channel created: #${CHANNEL_NAME} (${CHANNEL_ID})"
    
    # Set channel topic
    curl -s -X POST https://slack.com/api/conversations.setTopic \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"${CHANNEL_ID}\",\"topic\":\"${CHANNEL_TOPIC}\"}" >/dev/null
else
    echo "❌ Channel creation failed"
    exit 1
fi
```

## Step 5: Create Canvas with Roadmap

Create canvas in the new channel with content from ROADMAP.md:

**Canvas Title:** "{Focus Group} Roadmap"
**Canvas Content:** (Pull from .planning/focus-groups/{focus-group}/ROADMAP.md)

Set up bidirectional sync between canvas and ROADMAP.md file.

## Step 6: Update Master Configuration

Update .planning/WGSD-CONFIG.md:
```markdown
### Focus Groups
- **{focus-group}**: #{repo-name}-{focus-group} (created {date})
```

Update .planning/MASTER-ROADMAP.md:
```markdown
## Focus Groups Overview

### {Focus Group}
- **Channel:** #{repo-name}-{focus-group}
- **Status:** Active (0 concepts)
- **Created:** {date}
```

## Step 7: Commit and Announce

```bash
# Commit configuration updates
git checkout develop 2>/dev/null || git checkout main
git add .planning/WGSD-CONFIG.md .planning/MASTER-ROADMAP.md
git commit -m "feat: add ${FOCUS_GROUP} focus group to master configuration"
git push origin HEAD

echo "✅ Focus group ${FOCUS_GROUP} created successfully!"
```

## Step 8: Add Channel to OpenClaw Config

Add the new channel to OpenClaw's allowed channels list so Jarvis can monitor it:

```json
{
  "channels": {
    "slack": {
      "channels": {
        "{channel-id}": {
          "allow": true,
          "requireMention": false
        }
      }
    }
  }
}
```

Use gateway config.patch with the channel ID to automatically configure monitoring.

## Step 9: Welcome Message

Send welcome message to new channel explaining the workflow:

```markdown
🎉 **Welcome to the {Focus Group} Focus Group!**

This is a **WGSD (We Get Shit Done)** collaborative development channel.

**Purpose:** Social discussion and concept development for {focus-group} ideas.

**How it works:**
1. 💬 **Discuss ideas freely** - social collaboration is encouraged
2. 📝 **Capture concepts** with `/wgsd create-concept [name]` when ideas mature
3. 🔄 **Cross-reference** other focus groups via shared planning files
4. 🚀 **Promote to implementation** when concepts are ready

**Canvas Integration:**
- The "{Focus Group} Roadmap" canvas above syncs with git planning files
- Edit the canvas or git files - changes sync automatically
- Live roadmap stays current with your discussions

**Get Started:**
- What should we focus on in {focus-group}?
- What are the big challenges or opportunities?
- Use `/wgsd create-concept [name]` to capture your first idea

*Jarvis is monitoring this channel (no @ mentions needed) and will help with workflow automation.*
```

## Step 10: Return Status

```
---

## ✅ Focus Group Created: {Focus Group}

**Channel:** #{repo-name}-{focus-group}
**Git Branch:** focus-groups/{focus-group}
**Worktree:** concepts/{focus-group}/
**Canvas:** "{Focus Group} Roadmap" (syncs with git)

**Team can now:**
- Join #{repo-name}-{focus-group} for social discussion
- Create concepts with `/wgsd create-concept [name]`
- Reference planning files across focus groups
- View live roadmap via canvas integration

**Next:** Add team members to the channel and start discussing {focus-group} ideas!

---
```

</process>

<success_criteria>
- [ ] Focus group planning structure created in .planning/
- [ ] Git branch and worktree set up for isolated work
- [ ] Slack channel created with appropriate topic and settings
- [ ] Canvas created with roadmap content and bidirectional sync
- [ ] Master configuration updated with new focus group
- [ ] Welcome message sent explaining WGSD workflow to team
- [ ] Clear next steps provided for team collaboration
</success_criteria>