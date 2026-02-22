# Technology Stack Analysis

**Generated:** 2026-02-22
**Focus:** Multi-tenant AI assistant platform with billing and LLM proxy

## Runtime & Language

**Primary Runtime:**
- **Node.js:** >=20.0.0 (specified in `marvin-openclaw/package.json`)
- **TypeScript:** ^5.3.0 with strict compilation
- **Module System:** NodeNext with ESM support (`marvin-openclaw/tsconfig.json`)

**Compilation Configuration:**
- **Target:** ES2022 (`marvin-openclaw/tsconfig.json`)
- **Module Resolution:** NodeNext for modern import/export
- **Build Output:** `./dist` directory with source maps and declarations
- **Development:** `tsx watch` for hot reload (`marvin-openclaw/package.json`)

## Web Framework & HTTP

**Core Framework:**
- **Express.js** ^4.18.0 (`marvin-openclaw/src/router/index.ts`)
- **Server Location:** `marvin-openclaw/src/router/index.ts` (main entry point)
- **Port Configuration:** 8080 (from `marvin-openclaw/docker/docker-compose.yml`)

**HTTP Middleware:**
- Request parsing and routing in `marvin-openclaw/src/router/`
- Webhook handling in `marvin-openclaw/src/router/stripe.ts`

## Database & Storage

**Primary Database:**
- **PostgreSQL** 16-alpine (`marvin-openclaw/docker/docker-compose.yml`)
- **Node.js Driver:** `pg` ^8.11.0 (`marvin-openclaw/package.json`)
- **Schema Location:** `marvin-openclaw/src/shared/db/schema.sql`
- **Connection Pattern:** Database client in `marvin-openclaw/src/shared/db/client.ts`

**Schema Highlights:**
- Multi-tenant user system with Slack integration
- Double-entry accounting ledger system
- Session-based conversation management
- Usage events and billing integration

**Database Features:**
- UUID primary keys with `pgcrypto` extension
- Immutable ledger with triggers preventing modifications
- Complex check constraints for data integrity
- Full-text indexing for performance

**Cache & Session Store:**
- **Redis** 7-alpine for caching and pub/sub (`marvin-openclaw/docker/docker-compose.yml`)
- **Configuration:** `redis://redis:6379` in Docker environment

**Object Storage:**
- **MinIO** (S3-compatible) for local development (`marvin-openclaw/docker/docker-compose.yml`)
- **GCS Integration:** Expected for production (environment variable `GCS_BUCKET`)

## External Service Integrations

**AI/LLM Providers:**
- **Anthropic:** Direct integration via `marvin-openclaw/src/proxy/providers/anthropic.ts`
- **OpenAI:** Direct integration via `marvin-openclaw/src/proxy/providers/openai.ts`
- **Google Gemini:** `@google/generative-ai` ^0.24.1 in `marvin-openclaw/src/proxy/providers/gemini.ts`
- **Provider System:** Pluggable adapter pattern in `marvin-openclaw/src/proxy/providers/index.ts`

**Communication Platforms:**
- **Slack:** `@slack/bolt` ^4.0.0 and `@slack/web-api` ^7.0.0
- **Slack Integration:** `marvin-openclaw/src/router/bolt.ts` and `marvin-openclaw/src/router/slack.ts`
- **Markdown Processing:** `slackify-markdown` ^5.0.0 for Slack formatting

**Payment Processing:**
- **Stripe:** `stripe` ^14.0.0 (`marvin-openclaw/src/billing/stripe.ts`)
- **Billing Features:** Credit purchases, webhooks, customer portal
- **API Version:** 2023-10-16 (hardcoded in `marvin-openclaw/src/billing/stripe.ts`)

**Cloud Services:**
- **Google Cloud Secret Manager:** `@google-cloud/secret-manager` ^6.1.1
- **Secret Management:** Cached secret retrieval in `marvin-openclaw/src/shared/secrets.ts`
- **Project Configuration:** Environment variable `GCP_PROJECT_ID`

## Development & Build Tools

**Build System:**
- **TypeScript Compiler:** Standard `tsc` build process
- **Development:** `tsx watch` for hot reloading
- **Build Command:** `tsc` outputting to `dist/`
- **Start Command:** `node dist/router/index.js`

**Code Quality:**
- **ESLint:** ^8.50.0 with TypeScript parser (`@typescript-eslint/eslint-plugin` ^8.54.0)
- **Prettier:** ^3.0.0 for code formatting
- **Configuration:** ESLint targeting TypeScript files in `src/`

**Testing Framework:**
- **Vitest** ^1.0.0 (`marvin-openclaw/package.json`)
- **Test Files:** Located in `marvin-openclaw/tests/` directory
- **Test Types:** Integration tests for router and sandbox components

**Validation:**
- **Zod** ^3.22.0 for runtime type validation
- **Usage:** Request/response validation throughout the codebase

## Container & Deployment

**Containerization:**
- **Docker:** Multi-service setup in `marvin-openclaw/docker/docker-compose.yml`
- **Base Images:** Node.js for application, postgres:16-alpine, redis:7-alpine
- **Network:** Custom bridge network (`marvin-net`)

**Services Architecture:**
- **Router Service:** Main application server
- **Database Service:** PostgreSQL with health checks
- **Cache Service:** Redis with persistence
- **Storage Service:** MinIO for local S3 emulation
- **Admin Tools:** PgAdmin (optional, profile-based)

**Cloud Build:**
- **GCP Cloud Build:** Configuration in `marvin-openclaw/cloudbuild.yaml`
- **Container Registry:** `us-west1-docker.pkg.dev` for artifact storage
- **Build Strategy:** Docker multi-stage with router-specific Dockerfile

**Environment Configuration:**
- **Development:** Docker Compose with local services
- **Production:** GCP-based with Secret Manager integration
- **Health Checks:** Built into all containerized services

## Architecture Patterns

**Multi-Service Design:**
- **Router Service:** Request routing and business logic
- **Proxy Service:** LLM request proxying with provider abstraction
- **Billing Service:** Payment processing and credit management
- **Database Layer:** Shared database client with connection pooling

**Request Flow:**
1. Express router receives requests (`marvin-openclaw/src/router/index.ts`)
2. Authentication and session management
3. LLM proxy for AI requests (`marvin-openclaw/src/proxy/`)
4. Usage tracking and billing (`marvin-openclaw/src/billing/`)
5. Response formatting for platform-specific output

**Double-Entry Accounting:**
- Immutable ledger system for credit transactions
- System accounts for revenue/expense tracking
- Optimistic locking for concurrent balance updates
- Budget enforcement with usage tracking

## Performance & Monitoring

**Database Performance:**
- Comprehensive indexing strategy for all query patterns
- Optimistic locking for high-concurrency operations
- Immutable ledger preventing data corruption
- Running balance calculation for fast lookups

**Caching Strategy:**
- Redis for session storage and temporary data
- In-memory secret caching (5-minute TTL)
- Connection pooling for database efficiency

**Error Handling:**
- Typed error system with `ProxyError` class
- Request ID tracking throughout the system
- Structured logging for debugging and monitoring