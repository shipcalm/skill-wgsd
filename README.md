# 🚀 WGSD - We Get Shit Done

*Social collaborative development methodology for OpenClaw*

## What is WGSD?

WGSD extends traditional GSD methodology with social collaboration across Slack channels. It enables parallel feature development through focus groups, concept maturation, and controlled implementation - all with live Canvas integration and shared git planning structures.

## Architecture

### Three-Tier System
1. **Focus Groups (dozens)** - Long-lived topic channels for social ideation
2. **Concepts (hundreds)** - Feature ideas developed through team discussion  
3. **Implementations (2-4 max)** - Short-lived execution with controlled concurrency

### Key Features
- **Social Ideation**: Ideas develop through team collaboration, not isolation
- **Controlled Execution**: Maximum 2-4 concurrent implementations prevent chaos
- **Live Roadmaps**: Canvas integration keeps planning current across channels
- **Parallel Development**: Multiple implementations without merge conflicts
- **Shared Context**: Cross-channel references through unified planning structure

## Installation

```bash
npm install -g skill-wgsd
```

Or clone and symlink for development:
```bash
git clone https://github.com/shipcalm/skill-wgsd.git
cd skill-wgsd
npm link
```

## Usage

### Core Commands
- `create-focus-group [name]` - Create new focus group with channel + canvas
- `create-concept [name]` - Add concept to current focus group
- `promote-concept [name]` - Move concept to implementation queue
- `create-implementation [concept]` - Start implementation from concept
- `sync-canvas` - Pull canvas changes to git
- `update-canvas` - Push git changes to canvas
- `roadmap` - Show cross-focus-group roadmap

### Getting Started
```bash
setup-repo [repo-path]              # Initialize WGSD structure
create-focus-group [name]           # Create first focus group
```

## Repository Structure

WGSD creates shared planning structure in your repository:

```
{repo}/.planning/
├── PROJECT.md                      # Overall project context
├── focus-groups/                   # Long-lived topic areas
│   ├── security/
│   │   ├── ROADMAP.md             # ↔ Canvas sync
│   │   ├── STATE.md               # Current status
│   │   ├── concepts/              # Feature concepts
│   │   └── CHANNELS.md            # Slack mappings
├── active-implementations/         # Current execution (2-4 max)
├── MASTER-ROADMAP.md              # Cross-focus-group aggregation
└── WGSD-CONFIG.md                 # Channel mappings
```

## Development

This project uses [GSD (Get Shit Done)](https://github.com/openclaw/skill-gsd) methodology for development.

See `CONTRIBUTING.md` for development workflow.

## License

MIT