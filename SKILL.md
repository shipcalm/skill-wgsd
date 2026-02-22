---
name: wgsd
description: We Get Shit Done - Social collaborative development workflow. Extends GSD with focus groups, concepts, and implementation tracking across Slack channels with Canvas integration.
user-invocable: true
---

<objective>
WGSD (We Get Shit Done) enables social software development through:

- **Focus Groups**: Long-lived topic channels (Security, Onboarding, Billing)
- **Concepts**: Feature ideas developed socially within focus groups
- **Implementations**: Short-lived execution of mature concepts (1-3 days)
- **Canvas Integration**: Live roadmaps that sync bidirectionally with git
- **Shared Planning**: All channels reference unified .planning/ structure
- **Parallel Development**: Multiple implementations without merge conflicts

**Three-Tier Architecture:**
1. **Focus Groups** (dozens) - Social ideation, roadmap building
2. **Concepts** (hundreds) - Focused feature discussions  
3. **Implementations** (2-4 max) - Actual code execution

This is collaborative GSD - "We Get Shit Done" together.
</objective>

<intake>
What would you like to do?

**Focus Group Commands:**
- **create-focus-group [name]** - Create new focus group with channel, canvas, and planning structure
- **list-focus-groups** - Show all active focus groups and their status
- **archive-focus-group [name]** - Archive completed focus group

**Concept Commands:**
- **create-concept [name]** - Create concept within current focus group channel
- **promote-concept [name]** - Move concept to implementation queue
- **list-concepts [focus-group]** - Show concepts for focus group

**Implementation Commands:**
- **create-implementation [concept]** - Start implementation from mature concept
- **list-implementations** - Show active implementations (max 2-4)
- **complete-implementation [name]** - Merge and clean up implementation

**Canvas & Sync Commands:**
- **sync-canvas** - Pull canvas changes to git planning files
- **update-canvas** - Push git changes to canvas
- **roadmap** - Show cross-focus-group roadmap in main channel
- **create-canvas** - Create new canvas using Slack API (requires `exec` tool with OAuth token)

**Migration Commands:**
- **migrate [repo-path]** - Full GSD to WGSD migration wizard
- **analyze [repo-path]** - Analyze GSD project for migration readiness

**Repository Commands:**
- **status** - Show WGSD status across all focus groups and implementations
- **setup-repo [repo-path]** - Initialize WGSD structure in repository
- **create-channel [name] [topic]** - Create Slack channel via exec tool and API

**Usage Examples:**
- `/wgsd create-focus-group security` - Create #mvn-security channel and structure
- `/wgsd create-concept byof-filesystem` - Create concept within focus group
- `/wgsd promote-concept byof-filesystem` - Move to implementation queue  
- `/wgsd roadmap` - Show master roadmap in main development channel

**Workflow:**
1. **Focus Group Creation** - Long-lived topic channels for ideation
2. **Social Concept Development** - Ideas mature through team discussion
3. **Implementation Promotion** - Ready concepts become short-lived execution
4. **Canvas Sync** - Live roadmaps stay current with git planning
5. **Cross-Channel Collaboration** - Shared planning enables references
</intake>

<routing>
Based on user input, route to appropriate workflow:

| Intent | Workflow |
|--------|----------|
| "create focus group", "new focus group" | workflows/create-focus-group.md |
| "create-focus-group", "focus-group" | workflows/create-focus-group.md |
| "list focus groups", "show focus groups" | workflows/list-focus-groups.md |
| "create concept", "new concept" | workflows/create-concept.md |
| "create-concept", "concept" | workflows/create-concept.md |
| "promote concept", "implement concept" | workflows/promote-concept.md |
| "create implementation", "start implementation" | workflows/create-implementation.md |
| "list implementations", "show implementations" | workflows/list-implementations.md |
| "sync canvas", "pull canvas" | workflows/sync-canvas.md |
| "update canvas", "push canvas" | workflows/update-canvas.md |
| "roadmap", "master roadmap", "status" | workflows/roadmap.md |
| "setup repo", "initialize", "setup" | workflows/setup-repo.md |
| "create channel", "new channel" | workflows/create-channel.md |
| "migrate", "migration", "migrate from gsd" | workflows/migrate.md |
| "analyze", "analyze project", "migration readiness" | workflows/migrate.md (analysis mode) |
| "init", "initialize workspace" | workflows/init.md |
| "workspace status", "wgsd status" | workflows/workspace-status.md |

</routing>

<architecture>
## Repository Structure

WGSD creates shared planning structure:

```
{repo}/.planning/
├── PROJECT.md                      # Overall project context
├── focus-groups/                   # Long-lived topic areas
│   ├── security/
│   │   ├── ROADMAP.md             # Focus group roadmap (↔ Canvas)
│   │   ├── STATE.md               # Current status
│   │   ├── concepts/              # Feature concepts
│   │   │   ├── byof-filesystem.md
│   │   │   ├── auth-v2.md
│   │   │   └── sso-integration.md
│   │   └── CHANNELS.md            # Slack channel mappings
│   ├── onboarding/
│   │   ├── ROADMAP.md
│   │   ├── STATE.md
│   │   ├── concepts/
│   │   │   ├── recommended-prompts.md
│   │   │   └── agent-setup.md
│   │   └── CHANNELS.md
│   └── billing/
│       ├── ROADMAP.md
│       ├── STATE.md
│       ├── concepts/
│       └── CHANNELS.md
├── active-implementations/         # Current execution (2-4 max)
│   ├── auth-v2-impl/
│   │   ├── REQUIREMENTS.md        # Traditional GSD structure
│   │   ├── ROADMAP.md
│   │   ├── STATE.md
│   │   └── concept-source.md      # Links to originating concept
│   └── billing-api-impl/
│       ├── REQUIREMENTS.md
│       ├── ROADMAP.md
│       ├── STATE.md
│       └── concept-source.md
├── MASTER-ROADMAP.md              # Cross-focus-group aggregation
└── WGSD-CONFIG.md                 # Channel mappings and settings
```

## Git Integration

**Branches:**
- `focus-groups/security` - Planning and concept development
- `focus-groups/onboarding` - Planning and concept development  
- `implementations/auth-v2` - Code implementation (off develop)
- `implementations/billing-api` - Code implementation (off develop)

**Worktrees:**
```
{repo}/
├── .git/                          # Main git directory
├── concepts/                      # Focus group worktrees
│   ├── security/                  # → focus-groups/security branch
│   ├── onboarding/                # → focus-groups/onboarding branch
│   └── billing/                   # → focus-groups/billing branch
└── implementations/               # Implementation worktrees  
    ├── auth-v2/                   # → implementations/auth-v2 branch
    └── billing-api/               # → implementations/billing-api branch
```

## Slack Integration

**Channel Naming Convention:**
- `#{repo}-dev` - Main development community channel
- `#{repo}-{focus-group}` - Focus group channels (e.g., #mvn-security)
- `#{repo}-impl-{name}` - Implementation channels (e.g., #mvn-impl-auth-v2)

**Canvas Integration:**
- Focus group ROADMAP.md ↔ Focus group channel canvas
- Implementation STATE.md ↔ Implementation channel canvas
- Master roadmap ↔ Main dev channel canvas

**Canvas Creation Process:**
Canvas creation requires the `exec` tool with Slack API calls, not the `message` tool.

**For Channel Canvases (recommended):**
Use `conversations.canvases.create` - creates canvas that appears as icon in channel automatically:
```bash
curl -X POST https://slack.com/api/conversations.canvases.create \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_id": "CHANNEL_ID",
    "title": "Canvas Title",
    "document_content": {"type": "markdown", "markdown": "# Content"}
  }'
```

**For Standalone Canvases:**
Use `canvases.create` + `canvases.access.set` (requires manual sharing):
```bash
# 1. Create standalone canvas
curl -X POST https://slack.com/api/canvases.create \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "Canvas Title", "document_content": {"type": "markdown", "markdown": "# Content"}}'

# 2. Share to channel
curl -X POST https://slack.com/api/canvases.access.set \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"canvas_id": "CANVAS_ID", "access_level": "read", "channel_ids": ["CHANNEL_ID"]}'
```

The Slack bot token is available in OpenClaw config: `channels.slack.botToken`

## Versioning Strategy

**Focus Groups & Concepts:** No versions, just states
- [Draft] → [Exploring] → [Mature] → [Ready for Implementation]

**Implementations:** Semantic versioning based on merge reality
- Independent version planning: v2.1.0-auth-rework, v1.8.0-onboarding
- Actual version determined at merge time based on develop state
- Parallel implementations don't conflict because versions are contextual

## Workflow Files

Located in `workflows/`:
- **create-focus-group.md** - Create focus group structure and channels
- **create-concept.md** - Add concept to focus group  
- **promote-concept.md** - Move concept to implementation queue
- **create-implementation.md** - Start implementation from concept
- **sync-canvas.md** - Pull Slack canvas changes to git
- **update-canvas.md** - Push git changes to Slack canvas
- **roadmap.md** - Generate cross-focus-group status
- **setup-repo.md** - Initialize WGSD in repository

## Success Criteria

- [ ] Focus groups can be created with channels and planning structure
- [ ] Concepts mature through social discussion in dedicated channels  
- [ ] Ready concepts promote to controlled implementation (2-4 max)
- [ ] Canvas integration syncs bidirectionally with git planning
- [ ] Cross-channel references work through shared .planning/
- [ ] Parallel implementations don't conflict (semantic versioning)
- [ ] Team collaboration enabled through clear workflow explanation

</architecture>

<success_criteria>
- Repository initialized with WGSD planning structure
- Focus groups created with channels, canvases, and git branches
- Concepts tracked and promoted through maturity states
- Implementation channels limited to 2-4 concurrent maximum
- Canvas sync enables live roadmap collaboration in Slack
- Master roadmap aggregates status across all focus groups
- Clear developer onboarding and workflow documentation
</success_criteria>