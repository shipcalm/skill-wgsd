# Marvin OpenClaw Codebase Conventions

## Project Architecture

### High-Level Structure
- **Multi-tenant AI assistant platform** with sandbox isolation
- **TypeScript-first** codebase with strict type checking
- **Modular architecture** with clear separation of concerns:
  - `src/proxy/` - LLM Authentication Proxy with provider adapters
  - `src/router/` - Main routing and orchestration logic
  - `src/sandbox/` - Isolated sandbox environment management
  - `src/billing/` - Usage tracking and billing logic
  - `src/shared/` - Common utilities and database models

### Technology Stack
- **Runtime**: Node.js 20+ with ES2022 target
- **Language**: TypeScript with strict mode enabled
- **Module System**: ESM with NodeNext module resolution
- **Build**: TypeScript compiler (tsc) with source maps
- **Testing**: Vitest for unit and integration testing
- **Linting**: ESLint with TypeScript support
- **Formatting**: Prettier (configured via package.json script)
- **Database**: PostgreSQL with direct pg client usage
- **Cloud**: Google Cloud Platform (Cloud Run, GCS, Secret Manager)

## Code Style & Formatting

### TypeScript Configuration
- **Target**: ES2022 with NodeNext modules
- **Strict mode**: Enabled with full type checking
- **Source maps**: Generated for debugging
- **Declaration files**: Generated with source maps
- **File extensions**: `.ts` for source, `.js` for compiled output

### Import/Export Patterns
- **ESM imports**: Always use `.js` extension in imports for compiled output
- **Type imports**: Separate `type` imports when importing only types
- **Barrel exports**: Used in module index files (e.g., `src/proxy/index.ts`)
- **Named exports**: Preferred over default exports

```typescript
// Good - type-only import with .js extension
import type { ProxyLLMRequest, ProxyLLMResponse } from './types.js';

// Good - barrel export pattern
export {
  createSandboxToken,
  verifyToken,
  TokenRevocationStore,
} from './auth.js';
```

### Documentation Standards
- **File-level JSDoc**: Every module starts with comprehensive documentation
- **Architecture diagrams**: ASCII art diagrams for complex systems
- **API documentation**: Detailed descriptions of public interfaces
- **Security notes**: Explicit security considerations documented

### Error Handling
- **Custom error types**: `ProxyError` class with specific error codes
- **Graceful degradation**: Comprehensive error handling with fallbacks
- **Logging**: Structured logging with appropriate levels

## File Organization

### Directory Structure
```
src/
├── proxy/               # LLM proxy service
│   ├── providers/       # Provider-specific adapters
│   ├── examples/        # Usage examples
│   └── types.ts         # Type definitions
├── router/              # Main routing logic
├── sandbox/             # Sandbox management
├── billing/             # Usage tracking
└── shared/              # Common utilities
    └── db/              # Database related code
        └── migrations/  # Database migrations
```

### Naming Conventions
- **Files**: kebab-case (e.g., `llm-proxy.ts`)
- **Directories**: kebab-case (e.g., `shared/db`)
- **Classes**: PascalCase (e.g., `LLMProxyServer`)
- **Functions**: camelCase (e.g., `createSandboxToken`)
- **Constants**: SCREAMING_SNAKE_CASE (e.g., `MODEL_PRICING`)
- **Types**: PascalCase (e.g., `ProxyLLMRequest`)
- **Interfaces**: PascalCase (e.g., `ProviderAdapter`)

### Code Organization Patterns
- **Barrel exports**: Module-level index files re-export public APIs
- **Type-first**: Type definitions often at module top or in separate files
- **Provider pattern**: Consistent adapter interfaces for external services
- **Configuration objects**: Structured config with environment variable fallbacks

## API Design Patterns

### Request/Response Types
- **Consistent naming**: `ProxyLLMRequest`, `ProxyLLMResponse`
- **Generic interfaces**: Reusable across different providers
- **Type safety**: Full TypeScript coverage for API boundaries

### Authentication & Security
- **JWT tokens**: For sandbox authentication
- **API key injection**: Proxy pattern to hide keys from sandboxes
- **Usage tracking**: Every request metered for billing
- **Signed requests**: Request signature verification

### Provider Abstraction
- **Unified interface**: `ProviderAdapter` for all LLM providers
- **Model mapping**: Consistent model name handling across providers
- **Error normalization**: Consistent error types across providers
- **Token usage**: Standardized usage reporting

## Database Patterns

### Connection Management
- **Connection pooling**: Using `pg.Pool` for PostgreSQL connections
- **Environment configuration**: Database URL from environment variables
- **Graceful cleanup**: Proper connection cleanup in tests and shutdown

### Schema Organization
- **Migration-based**: Database migrations in `shared/db/migrations/`
- **Type safety**: Database types generated or manually maintained
- **User-centric**: Core tables include `users` with proper relationships

## Testing Patterns

### Test Structure
- **Integration-focused**: Comprehensive integration tests over unit tests
- **Real dependencies**: Tests against actual database and Cloud Run APIs
- **Environment configuration**: Flexible test configuration via environment variables
- **Mock services**: Mock servers for external dependencies when needed

### Test Organization
```
tests/
├── router/              # Router-specific tests
│   ├── integration.test.ts
│   └── user.test.ts
└── sandbox/             # Sandbox-specific tests
    └── integration.test.ts
```

### Test Configuration
- **Environment variables**: Extensive configuration via env vars
- **Conditional tests**: Tests skip gracefully when dependencies unavailable
- **Cleanup procedures**: Automatic cleanup of test resources
- **Debug support**: Debug modes for troubleshooting

## Security Considerations

### Sandbox Isolation
- **No direct API access**: Sandboxes access LLMs only through proxy
- **Token-based auth**: JWT tokens for sandbox authentication
- **Key isolation**: API keys never exposed to sandbox environments
- **Usage limits**: Balance checking prevents runaway costs

### Data Protection
- **Secure token storage**: Proper token lifecycle management
- **Request signing**: Cryptographic request verification
- **Environment isolation**: Clear separation between environments
- **Minimal permissions**: Service accounts with least-privilege access

## Performance Patterns

### Efficient Operations
- **BigInt for precision**: Using BigInt for micro-cent pricing calculations
- **Connection pooling**: Proper database connection management
- **Batch operations**: Batched event recording for efficiency
- **Async/await**: Modern async patterns throughout

### Resource Management
- **Proper cleanup**: Resource cleanup in tests and production
- **Memory efficiency**: Streaming responses where appropriate
- **Error boundaries**: Graceful error handling without resource leaks

## Development Workflow

### Scripts & Tooling
- **Development**: `tsx watch` for hot reloading
- **Building**: `tsc` for compilation
- **Testing**: `vitest` for test execution
- **Linting**: `eslint` for code quality
- **Formatting**: `prettier` for code formatting

### Environment Management
- **Environment variables**: Comprehensive env var usage
- **Configuration objects**: Centralized configuration management
- **Development vs Production**: Clear environment separation
- **Service account management**: Proper GCP service account handling