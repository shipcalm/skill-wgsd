# Codebase Structure Analysis

*Updated: February 22, 2026*

## Root Directory Overview

The workspace follows a **functional domain organization** with clear separation between platform code, extensibility systems, and operational concerns:

```
workspace/
├── marvin-openclaw/          # Primary commercial platform codebase
├── skills/                   # Modular AI capabilities (18+ skills)
├── projects/                 # Strategic development initiatives (10+ projects)
├── services/                 # Supporting microservices
├── .planning/                # Project planning and documentation hierarchy
├── memory/                   # Session logs and daily activity records
├── github/                   # GitHub integration and repository management
├── [core-configs]*.md        # System configuration files (AGENTS, SOUL, etc.)
├── [automation]*.sh          # Deployment and maintenance scripts
├── [data]*.sql               # Database scripts and migrations
└── [operational-files]       # Development utilities and temporary files
```

## Marvin Platform Structure (`marvin-openclaw/`)

### Application Source Code (`src/`)

#### Core Service Architecture
```
src/
├── router/                   # Main application gateway and API
│   ├── index.ts             # Express server + Slack Events API integration
│   ├── user.ts              # User management, provisioning, database ops
│   ├── bolt.ts              # Slack Bolt app configuration and event handling
│   └── api.ts               # RESTful API endpoints for external integrations
├── proxy/                    # LLM gateway and provider abstraction
│   ├── index.ts             # Main proxy routing and request handling
│   ├── server.ts            # Proxy server implementation
│   ├── client.ts            # Client SDK for proxy integration
│   ├── auth.ts              # Authentication and API key management
│   ├── metering.ts          # Usage tracking and billing integration
│   ├── passthrough.ts       # Direct API proxy mode
│   ├── types.ts             # TypeScript definitions for proxy layer
│   ├── main.ts              # Proxy application entry point
│   ├── providers/           # Multi-provider LLM integrations
│   │   ├── index.ts         # Provider registry and routing
│   │   ├── openai.ts        # OpenAI API integration
│   │   ├── anthropic.ts     # Anthropic Claude API integration
│   │   └── gemini.ts        # Google Gemini API integration
│   └── examples/            # Integration examples and testing utilities
│       └── integration.ts   # End-to-end integration examples
├── billing/                  # Financial and usage management system
│   ├── index.ts             # Billing service main interface
│   └── stripe.ts            # Stripe payment processing integration
├── sandbox/                  # User environment orchestration
│   ├── manager.ts           # Sandbox lifecycle and resource management
│   ├── docker.ts            # Container orchestration and Docker integration
│   └── cleanup.ts           # Resource cleanup and maintenance utilities
└── shared/                   # Common services and utilities
    ├── db/                  # Database abstraction and management
    │   ├── index.ts         # Connection pooling, query abstraction
    │   └── migrations/      # Database schema evolution scripts
    ├── types.ts             # Platform-wide TypeScript type definitions
    ├── config.ts            # Configuration management and environment variables
    └── utils.ts             # Common utility functions and helpers
```

### Web Portal & User Interface (`portal/`)
```
portal/
└── src/
    └── app/                 # Next.js application structure
        ├── api/             # Backend API route handlers
        │   ├── checkout/    # Stripe payment integration endpoints
        │   ├── logout/      # Authentication and session management
        │   ├── usage/       # Usage tracking and analytics endpoints
        │   └── user/        # User profile and account management
        ├── auth/            # Authentication flow pages
        │   └── slack/       # Slack OAuth integration and callbacks
        └── dashboard/       # Main user dashboard interface
```

### Infrastructure & Deployment
```
marvin-openclaw/
├── terraform/               # Infrastructure as Code (Google Cloud)
│   ├── environments/        # Environment-specific configurations
│   ├── main.tf             # Primary GCP infrastructure definition
│   └── variables.tf        # Configuration variables and parameters
├── docker/                 # Container definitions and orchestration
│   ├── Dockerfile          # Multi-stage application container build
│   └── docker-compose.yml  # Local development environment setup
├── .github/                # CI/CD automation and workflows
│   └── workflows/          # GitHub Actions pipeline definitions
│       └── deploy.yml      # Deployment automation pipeline
├── scripts/                # Build, deployment, and maintenance scripts
├── docs/                   # Project documentation and specifications
│   ├── DATABASE.md         # Database schema and design documentation
│   └── ONBOARDING_FLOW.md  # User onboarding process documentation
└── tests/                  # Comprehensive test suites
    ├── router/             # Router service integration tests
    │   ├── integration.test.ts  # End-to-end router testing
    │   └── user.test.ts    # User management functionality tests
    └── sandbox/            # Sandbox service testing
        └── integration.test.ts  # Sandbox integration testing
```

### Configuration & Development Support
```
marvin-openclaw/
├── package.json            # Node.js dependencies and build scripts
├── tsconfig.json           # TypeScript compiler configuration
├── vitest.config.ts        # Testing framework configuration
├── .eslintrc.json          # Code linting rules and standards
├── cloudbuild.yaml         # Google Cloud Build deployment configuration
├── cloudbuild-sandbox.yaml # Sandbox-specific build configuration
├── SPEC.md                 # Platform specification and requirements (17KB)
├── INFRASTRUCTURE.md       # Infrastructure design and deployment guide
├── README.md               # Project overview and development setup
└── CONTRIBUTING.md         # Development contribution guidelines
```

## Skills Ecosystem Structure (`skills/`)

### Organizational Pattern
Each skill follows a **standardized plugin architecture**:

```
skills/[skill-name]/
├── SKILL.md                # Skill definition, capabilities, API documentation
├── _meta.json              # ClawHub integration metadata
├── references/             # External API documentation and specifications
├── scripts/                # Automation scripts and utility tools
├── workflows/              # Automated process definitions
├── templates/              # Reusable templates and configurations
├── agents/                 # Specialized sub-agents (for complex skills)
├── .planning/              # Skill-specific planning and documentation
└── [config-files]          # Skill-specific configuration (JSON, mappings, etc.)
```

### Current Skills Inventory (18 Active Skills)

#### Business & Productivity Integration
```
skills/
├── clickup/                # ClickUp project management platform
│   ├── SKILL.md            # API integration guide and capabilities
│   ├── user_mapping.json   # User ID to username mappings (18 team members)
│   └── references/         # ClickUp API documentation and schemas
├── google-workspace/        # Google Workspace suite integration
│   ├── SKILL.md            # Gmail, Drive, Docs integration capabilities
│   └── references/         # Google API documentation and OAuth setup
├── google-calendar/        # Google Calendar scheduling integration
│   ├── SKILL.md            # Calendar management and scheduling automation
│   └── scripts/            # Calendar automation and sync utilities
└── infoplus/               # InfoPlus WMS (Warehouse Management System)
    ├── SKILL.md            # WMS integration and inventory management
    ├── lob_mapping.json    # Line of Business ID mappings (250+ LOBs)
    └── references/         # InfoPlus API documentation and schemas
```

#### AI & Communication Enhancement
```
skills/
├── elevenlabs-agents/      # ElevenLabs AI voice agent platform
│   ├── SKILL.md            # Voice agent creation and management
│   └── references/         # ElevenLabs API and agent configuration docs
├── sag/                    # ElevenLabs TTS (Speech Synthesis) integration
│   ├── SKILL.md            # Text-to-speech conversion and voice selection
│   └── references/         # Voice model documentation and usage guides
├── voice-thinking/         # Voice-based AI interaction enhancement
│   ├── SKILL.md            # Voice processing and thinking augmentation
│   └── scripts/            # Voice processing utilities and filters
├── gemini-image-simple/    # Google Gemini image analysis
│   ├── SKILL.md            # Image processing and analysis capabilities
│   └── references/         # Gemini Vision API documentation
└── jarvis-channel/         # Core Jarvis communication channel management
    ├── SKILL.md            # Channel integration and message routing
    └── _meta.json          # Channel configuration metadata
```

#### Productivity & Workflow Automation
```
skills/
├── gsd/                    # Getting Stuff Done framework (comprehensive)
│   ├── SKILL.md            # GSD methodology and workflow management (9KB)
│   ├── agents/             # Specialized GSD agents for different tasks
│   ├── references/         # GSD methodology documentation and best practices
│   ├── templates/          # Project and task templates
│   ├── workflows/          # Automated workflow definitions and triggers
│   └── .planning/          # GSD-specific planning and progress tracking
├── wgsd/                   # Extended GSD capabilities
│   ├── SKILL.md            # Advanced workflow and project management
│   └── references/         # Extended methodology documentation
├── controlplane/           # System control and orchestration
│   └── SKILL.md            # Infrastructure control and monitoring capabilities
└── networking/             # Network intelligence and automation
    ├── SKILL.md            # Network analysis and management tools
    ├── .planning/          # Network project planning and documentation
    ├── attack/             # Security testing and penetration testing tools
    ├── intelligence/       # Network intelligence gathering and analysis
    ├── recon/              # Network reconnaissance and discovery tools
    ├── scripts/            # Network automation and utility scripts
    └── tools/              # Network management and monitoring utilities
```

#### Utility & Development Tools
```
skills/
├── nano-banana/            # Lightweight utility skill for common tasks
│   ├── SKILL.md            # Basic utilities and helper functions
│   └── references/         # Utility documentation and usage examples
├── nano-banana-pro/        # Enhanced utility skill with advanced features
│   ├── SKILL.md            # Advanced utilities and automation tools
│   └── references/         # Extended utility documentation
└── quickbooks/             # QuickBooks accounting integration
    ├── SKILL.md            # Financial data integration and reporting (14KB)
    └── references/         # QuickBooks API documentation and schemas
```

## Strategic Projects Structure (`projects/`)

### Active Development Initiatives (10+ Projects)

#### Platform Evolution & Modernization
```
projects/
├── marvin-openclaw-to-marvin-os/    # Platform modernization initiative
│   └── [migration-documentation]   # Migration strategies and implementation plans
├── marvin-custom-identity/          # Custom identity and branding system
│   ├── .planning/                  # Identity system planning and design
│   ├── docs/                       # Identity implementation documentation
│   ├── src/                        # Custom identity source code
│   └── tests/                      # Identity system testing suite
├── openclaw-multitenant/            # Multi-tenancy enhancement project
│   └── research/                   # Multi-tenancy research and architecture analysis
└── slack-custom-identity/           # Slack-specific identity customization
    ├── .planning/                  # Slack identity planning documentation
    └── src/                        # Slack identity implementation code
```

#### Intelligence & Automation
```
projects/
├── networking-intelligence-v2/      # Advanced network intelligence platform
│   ├── .planning/                  # Network intelligence project planning
│   ├── src/                        # Intelligence gathering and analysis tools
│   ├── docs/                       # Network intelligence documentation
│   └── tests/                      # Intelligence system testing
├── customer-hunter/                 # Customer acquisition and intelligence
│   ├── .planning/                  # Customer research and targeting planning
│   ├── research/                   # Market research and customer analysis
│   ├── automation/                 # Customer outreach automation tools
│   ├── data/                       # Customer data and intelligence gathering
│   └── reports/                    # Customer intelligence reports and findings
├── organic-content-strategy/        # Content marketing automation
│   └── notes/                      # Content strategy planning and execution
└── test-automation/                 # Quality assurance and testing framework
    ├── .planning/                  # Test automation planning and strategy
    └── src/                        # Automated testing tools and frameworks
```

## Supporting Services (`services/`)

### Microservices Supporting Infrastructure
```
services/
├── openclaw-monitoring/            # Platform monitoring and observability
│   └── src/                       # Monitoring service implementation
└── sms-forwarder/                 # SMS forwarding and communication service
    ├── .planning/                 # SMS service planning and configuration
    └── src/                       # SMS forwarding implementation
```

## Planning & Documentation System (`.planning/`)

### Structured Project Management
```
.planning/
├── codebase/                       # Codebase analysis and documentation
│   ├── ARCHITECTURE.md            # System architecture analysis (current)
│   ├── STRUCTURE.md               # File structure analysis (current)  
│   ├── STACK.md                   # Technology stack documentation
│   ├── CONVENTIONS.md             # Coding standards and conventions
│   ├── INTEGRATIONS.md            # External integrations and APIs
│   ├── TESTING.md                 # Testing strategies and frameworks
│   └── CONCERNS.md                # Technical debt and architectural concerns
├── phases/                        # Project phases and milestone tracking
│   ├── phase-01-foundation/       # Initial platform foundation
│   ├── phase-02-multitenancy/     # Multi-tenancy implementation
│   ├── phase-03-scaling/          # Scaling and performance optimization
│   ├── [additional-phases]/       # Future development phases
│   └── current-phase.md           # Active phase tracking and status
├── research/                      # Research documents and analysis
│   └── [research-topics]/         # Specific research areas and findings
└── todos/                         # Task management and TODO tracking
    ├── immediate/                 # High-priority immediate tasks
    ├── backlog/                   # Future development backlog
    └── completed/                 # Completed tasks and retrospectives
```

## System Configuration Files (Root Level)

### Core System Identity & Behavior
```
├── AGENTS.md               # Agent behavior guidelines and workspace rules (7.8KB)
├── SOUL.md                 # Core personality and behavioral definition
├── USER.md                 # User profile, preferences, and context
├── IDENTITY.md             # System identity and branding configuration
├── TOOLS.md                # Tool-specific local configuration and credentials (3.2KB)
├── MEMORY.md               # Long-term memory and contextual knowledge (27.6KB)
└── HEARTBEAT.md            # Proactive task checklist and automation triggers
```

### Operational & Development Support
```
├── ROADMAP.md              # Development roadmap and strategic planning (7.2KB)
├── STATE.md                # Current system state and status tracking (3.9KB)
├── BOOTSTRAP.md            # Initial setup and configuration instructions
└── VERIFICATION.md         # System verification and health check procedures (6.7KB)
```

## Memory & Session Management (`memory/`)

### Session Tracking & Historical Data
```
memory/
├── 2026-02-22.md          # Today's session log and activity record
├── 2026-02-21.md          # Recent daily activity logs
├── [historical-logs]      # Historical daily session records
└── heartbeat-state.json   # Heartbeat system state tracking
```

## File Organization Patterns & Conventions

### Documentation Hierarchy
1. **System Level**: Root configuration files (AGENTS.md, SOUL.md, etc.)
2. **Project Level**: Project-specific documentation in dedicated directories
3. **Service Level**: Service-specific documentation within service directories  
4. **Skill Level**: SKILL.md files with comprehensive capability documentation
5. **Planning Level**: Structured planning documents in `.planning/` hierarchy

### Code Organization Principles
1. **Domain Separation**: Clear boundaries between business domains and technical concerns
2. **Modular Architecture**: Independent, deployable services and skills
3. **Configuration Management**: Environment-specific configurations isolated and secured
4. **Infrastructure as Code**: All infrastructure defined and versioned in source control

### File Size & Complexity Analysis

#### Large Documentation Files (>10KB)
- `MEMORY.md` (27.6KB) - Comprehensive long-term memory and context
- `ARCHITECTURE.md` (current) (15.8KB) - Detailed architectural analysis
- `STRUCTURE.md` (current) (9.8KB) - Comprehensive structure documentation  
- `quickbooks/SKILL.md` (14KB) - Complex financial integration documentation
- `gsd/SKILL.md` (9KB) - Comprehensive workflow management framework

#### High-Complexity Integration Points
- **Multi-provider LLM Proxy**: Complex routing and provider abstraction
- **Multi-tenant Billing**: Credit-based accounting with Stripe integration
- **Slack Integration**: Event-driven message processing and bot management
- **Sandbox Orchestration**: Container-based user environment management
- **Skills Ecosystem**: Plugin-based capability extension and management

### Development Workflow Structure

#### Version Control Strategy
- **Git Repository**: Comprehensive version control with structured branching
- **Monorepo Approach**: Multiple projects and services in single repository
- **Clear History**: Structured commit patterns with descriptive messages
- **Branch Protection**: Main branch protection with automated testing

#### Environment Management
- **Local Development**: Docker Compose for consistent development environments
- **Testing Automation**: Comprehensive test suites with CI/CD integration
- **Production Deployment**: Cloud-based deployment with infrastructure automation
- **Configuration Management**: Environment-specific configurations with secret management

This structure analysis reflects a mature, well-organized codebase with clear separation of concerns, comprehensive documentation, and strong foundations for continued development and scaling.