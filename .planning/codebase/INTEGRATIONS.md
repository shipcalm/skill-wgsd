# External Integrations Analysis

**Generated:** 2026-02-22
**Focus:** Third-party services, APIs, and external dependencies

## AI/LLM Service Integrations

### Anthropic Claude
**Integration Location:** `marvin-openclaw/src/proxy/providers/anthropic.ts`
**Package:** Direct HTTP API calls (no specific npm package)
**Authentication:** API key via GCP Secret Manager (`ANTHROPIC_API_KEY`)
**Usage Pattern:**
- Provider adapter implementation in proxy system
- Streaming and non-streaming request support
- Token usage tracking for billing
- Request/response transformation for normalized interface

### OpenAI
**Integration Location:** `marvin-openclaw/src/proxy/providers/openai.ts`
**Package:** Direct HTTP API calls (no specific npm package)
**Authentication:** API key via GCP Secret Manager (`OPENAI_API_KEY`)
**Usage Pattern:**
- Provider adapter in proxy system
- Chat completions and streaming support
- Token counting and cost calculation
- Model validation and capability checking

### Google Gemini
**Integration Location:** `marvin-openclaw/src/proxy/providers/gemini.ts`
**Package:** `@google/generative-ai` ^0.24.1
**Authentication:** API key via GCP Secret Manager
**Implementation:** `marvin-openclaw/src/router/ai.ts` - GoogleGenerativeAI client
**Usage Pattern:**
- Official Google SDK integration
- Model support for multi-turn conversations
- Safety settings and content filtering
- Structured output support

### LLM Proxy Architecture
**Core System:** `marvin-openclaw/src/proxy/index.ts`
**Provider Registry:** `marvin-openclaw/src/proxy/providers/index.ts`
**Features:**
- Unified interface for multiple LLM providers
- Request metering and cost tracking
- Authentication handling per provider
- Streaming response management
- Error handling and retry logic

## Communication Platform Integrations

### Slack Platform
**Primary Integration:** `@slack/bolt` ^4.0.0
**Secondary Integration:** `@slack/web-api` ^7.0.0
**Implementation Files:**
- `marvin-openclaw/src/router/bolt.ts` - Main Slack app setup
- `marvin-openclaw/src/router/slack.ts` - WebClient for API calls
**Authentication:** Bot tokens and signing secrets via Secret Manager
**Features:**
- Event handling for messages and interactions
- Thread-based conversation tracking
- User and workspace identity management
- Rich message formatting with `slackify-markdown` ^5.0.0

### Message Format Processing
**Package:** `slackify-markdown` ^5.0.0
**Location:** `marvin-openclaw/src/router/format.ts`
**Purpose:** Convert markdown to Slack-specific formatting
**Usage:** Response formatting for platform-appropriate display

## Payment & Billing Integration

### Stripe Payment Processing
**Package:** `stripe` ^14.0.0
**Implementation:** `marvin-openclaw/src/billing/stripe.ts`
**API Version:** 2023-10-16 (locked)
**Authentication:** Secret keys via GCP Secret Manager
- `STRIPE_TEST_SECRET_KEY` - Test environment key
- `STRIPE_WEBHOOK_SECRET` - Webhook signature validation

**Integration Features:**
- Checkout session creation for credit purchases
- Webhook endpoint for payment event handling
- Customer portal for billing management
- Programmatic webhook endpoint configuration
- Credit package management with pricing

**Webhook Events Handled:**
- Payment completion notifications
- Subscription status changes
- Failed payment processing
- Customer updates

**Files:**
- `marvin-openclaw/src/billing/stripe.ts` - Main integration
- `marvin-openclaw/src/router/stripe.ts` - Webhook handling
- `marvin-openclaw/src/router/checkout.ts` - Purchase flow

## Cloud Infrastructure Integration

### Google Cloud Platform
**Secret Manager:** `@google-cloud/secret-manager` ^6.1.1
**Implementation:** `marvin-openclaw/src/shared/secrets.ts`
**Project Configuration:** `GCP_PROJECT_ID` environment variable (default: 'marvin-openclaw')
**Caching:** 5-minute in-memory cache for secret values

**Secrets Managed:**
- `ANTHROPIC_API_KEY` - Anthropic Claude API key
- `OPENAI_API_KEY` - OpenAI API key
- `STRIPE_TEST_SECRET_KEY` - Stripe payment processing
- `STRIPE_WEBHOOK_SECRET` - Stripe webhook validation
- Additional service API keys as needed

**Cloud Build Integration:**
- **Configuration:** `marvin-openclaw/cloudbuild.yaml`
- **Container Registry:** `us-west1-docker.pkg.dev`
- **Build Process:** Docker image creation and deployment
- **Artifact Storage:** Multi-tagged container images

### Google Cloud Storage
**Integration:** Expected for production file storage
**Configuration:** `GCS_BUCKET` environment variable
**Usage:** File and object storage for user content
**Local Development:** MinIO S3-compatible storage emulation

## Database Integration

### PostgreSQL
**Version:** 16-alpine (containerized)
**Driver:** `pg` ^8.11.0
**Connection Management:** `marvin-openclaw/src/shared/db/client.ts`
**Configuration:**
- Host: `DB_HOST` (postgres in Docker)
- Port: `DB_PORT` (5432)
- Database: `DB_NAME` (marvin)
- Credentials: `DB_USER`/`DB_PASSWORD`

**Advanced Features:**
- Connection pooling for performance
- Transaction support for data consistency
- Prepared statements for security
- Health checks for container orchestration

### Redis Cache
**Version:** 7-alpine (containerized)
**Configuration:** `REDIS_URL` (redis://redis:6379)
**Usage Patterns:**
- Session data storage
- Temporary caching for performance
- Pub/sub for real-time features
- Rate limiting and throttling

## Development & DevOps Integration

### Docker Services
**Configuration:** `marvin-openclaw/docker/docker-compose.yml`
**Services Integrated:**
- **PostgreSQL:** Database with initialization scripts
- **Redis:** Cache and session storage
- **MinIO:** S3-compatible object storage
- **PgAdmin:** Database administration (optional)

**Network Configuration:**
- Custom bridge network (`marvin-net`)
- Service discovery by container name
- Health check integration for dependency management

### Development Tools Integration
**TypeScript:** Strict type checking with modern ES modules
**ESLint:** TypeScript-specific linting rules
**Prettier:** Automated code formatting
**Vitest:** Modern testing framework with TypeScript support

## API Integration Patterns

### Authentication Strategy
**Secret Management:** Centralized via GCP Secret Manager
**Caching:** In-memory caching with TTL for performance
**Key Rotation:** Supported through secret versioning
**Environment Separation:** Different secrets for dev/test/prod

### Request/Response Handling
**Normalization:** Common interface across all LLM providers
**Error Handling:** Typed error system with provider-specific mapping
**Retry Logic:** Built into provider adapters
**Rate Limiting:** Request throttling and queue management

### Usage Tracking Integration
**Metering System:** Token and cost tracking per request
**Billing Integration:** Direct connection to credit system
**Analytics:** Detailed usage events for reporting
**Budget Enforcement:** Real-time budget checking and limits

## Security Integration

### Data Protection
**Encryption:** PostgreSQL with encrypted connections
**Secret Storage:** GCP Secret Manager with IAM controls
**API Security:** Webhook signature validation (Stripe)
**Input Validation:** Zod schemas for request/response validation

### Access Control
**Authentication:** Platform-specific (Slack user tokens)
**Authorization:** User-based resource access
**Audit Trail:** Immutable ledger for financial transactions
**Session Management:** Secure session tokens with expiration