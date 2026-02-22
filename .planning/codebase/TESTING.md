# Marvin OpenClaw Testing Patterns & Guidelines

## Testing Philosophy

### Integration-First Approach
- **Real-world testing**: Tests verify actual system integration rather than isolated units
- **End-to-end validation**: Full stack testing from database to API endpoints
- **Production-like conditions**: Tests use real services (database, Cloud Run) when available
- **Graceful degradation**: Tests skip gracefully when dependencies are unavailable

### Quality Focus
- **Comprehensive coverage**: Tests cover database, cloud services, networking, and business logic
- **Error scenario testing**: Explicit testing of failure modes and error handling
- **Security validation**: Authentication, authorization, and data protection testing
- **Performance awareness**: Tests designed to run quickly (< 30 seconds total)

## Testing Framework & Tools

### Core Testing Stack
- **Test Runner**: Vitest (fast, TypeScript-native)
- **Assertion Library**: Vitest's built-in expectations
- **Mock Services**: Custom Express-based mock servers
- **Database**: Real PostgreSQL connections with proper cleanup
- **Cloud Integration**: Google Cloud SDK for Cloud Run API testing

### Test Configuration
```typescript
// Centralized test configuration pattern
const TEST_CONFIG = {
  projectId: process.env.GCP_PROJECT_ID || 'marvin-openclaw',
  region: process.env.GCP_REGION || 'us-west1',
  environment: process.env.ENVIRONMENT || 'dev',
  testUserId: 'TEST_USER_123',
  dbUrl: process.env.DATABASE_URL,
  // ... other configuration
};
```

## Test Organization Structure

### Directory Layout
```
tests/
├── router/              # Router integration tests
│   ├── integration.test.ts
│   └── user.test.ts
├── sandbox/             # Sandbox-specific tests  
│   └── integration.test.ts
└── README.md           # Comprehensive testing documentation
```

### Test File Structure
- **File-level documentation**: Each test file starts with comprehensive JSDoc
- **Configuration section**: Centralized test configuration objects
- **Setup/teardown**: Proper beforeAll/afterAll for resource management
- **Grouped test suites**: Related tests grouped in describe blocks

## Testing Patterns

### Database Testing
```typescript
describe('Database Schema Integration', () => {
  it('should connect to database and verify basic schema', async () => {
    if (!dbPool) {
      console.warn('Database not configured, skipping database tests');
      return;
    }
    
    const client = await dbPool.connect();
    try {
      // Test basic connectivity
      const result = await client.query('SELECT NOW()');
      expect(result.rows).toHaveLength(1);
      
      // Verify schema
      const tables = await client.query(tableQuery);
      expect(tableNames).toContain('users');
    } finally {
      client.release();
    }
  });
});
```

### Mock Service Testing
```typescript
// Custom mock server creation
const mockApp = express();
mockApp.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: Date.now() });
});

// Integration with real HTTP calls
const response = await fetch(`http://localhost:${mockPort}/health`);
const data = await response.json();
expect(data.status).toBe('healthy');
```

### Cloud Integration Testing
```typescript
describe('Google Cloud Integration', () => {
  it('should validate Google Cloud authentication', async () => {
    try {
      const authClient = await auth.getClient();
      expect(authClient).toBeDefined();
    } catch (error) {
      console.warn('Google Auth not available in test environment');
      // Graceful handling of auth failures
    }
  });
});
```

## Environment-Based Testing

### Configuration Management
- **Environment variables**: Extensive use of env vars for test configuration
- **Conditional execution**: Tests skip when required services unavailable
- **Default values**: Sensible defaults for development environment
- **Security**: Separate credentials and configuration for test environments

### Required Environment Variables
```bash
# Database testing
DATABASE_URL="postgresql://user:password@localhost/marvin_dev"

# Cloud testing  
GCP_PROJECT_ID="marvin-openclaw"
GCP_REGION="us-west1"
GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# Optional advanced testing
SANDBOX_IMAGE="us-west1-docker.pkg.dev/marvin-openclaw/marvin/sandbox:latest"
RUN_CLOUD_TESTS="true"
CLEANUP_CLOUD_RUN="true"
```

### Graceful Degradation Pattern
```typescript
it('should test feature when service available', async () => {
  if (!serviceAvailable) {
    console.warn('Service not configured, skipping test');
    return;
  }
  // Actual test logic here
});
```

## Resource Management

### Setup and Cleanup
```typescript
let dbPool: Pool;
let auth: GoogleAuth;

beforeAll(async () => {
  // Initialize resources
  if (TEST_CONFIG.dbUrl) {
    dbPool = new Pool({ connectionString: TEST_CONFIG.dbUrl });
  }
  auth = new GoogleAuth({
    scopes: ['https://www.googleapis.com/auth/cloud-platform']
  });
});

afterAll(async () => {
  // Clean up resources
  if (dbPool) {
    await dbPool.end();
  }
});
```

### Connection Management
- **Connection pooling**: Proper database connection lifecycle
- **Resource cleanup**: Explicit cleanup in afterAll hooks
- **Error handling**: Cleanup even when tests fail
- **Timeout handling**: Reasonable timeouts for external service calls

## Mock Service Patterns

### Express-Based Mock Servers
```typescript
const mockApp = express();

// Health endpoint
mockApp.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: Date.now() });
});

// Workspace initialization API
mockApp.post('/api/workspace/init', (req, res) => {
  const { userId } = req.body;
  res.json({
    success: true,
    workspace: `/workspaces/${userId}/workspace`,
    initialized: true
  });
});

// Error simulation
mockApp.get('/error', (req, res) => {
  res.status(500).json({ error: 'Simulated error' });
});
```

### Mock Server Lifecycle
- **Dynamic port allocation**: Avoid port conflicts
- **Automatic cleanup**: Servers cleaned up after tests
- **Debug mode**: Option to keep servers running for manual testing
- **Health checks**: Verify mock servers are ready before tests

## Test Data Management

### Consistent Test Data
```typescript
const TEST_TEMPLATE_VARIABLES: TemplateVariables = {
  email: 'test.sandbox@example.com',
  name: 'Test Sandbox User',
  timezone: 'UTC',
  created_at: new Date().toISOString()
};
```

### Test User Management
- **Consistent identifiers**: Standard test user IDs across tests
- **Email patterns**: Consistent email patterns for test users
- **Isolation**: Each test uses unique identifiers to avoid conflicts
- **Cleanup**: Test data cleaned up after test runs

## Testing Categories

### 1. Database Integration Tests
**Purpose**: Verify database connectivity and schema validation
- PostgreSQL connection establishment
- Table existence verification
- Basic query functionality
- Error handling for connection issues

### 2. Cloud Service Integration
**Purpose**: Validate Google Cloud Platform integration
- Authentication with service accounts
- Cloud Run API connectivity
- Resource configuration validation
- Permission and access control testing

### 3. HTTP API Testing
**Purpose**: Test API endpoints and communication
- Health check endpoints
- Authentication header validation
- Request/response format verification
- Error response handling

### 4. Template System Testing
**Purpose**: Validate workspace and template functionality
- Template file existence verification
- Variable substitution testing
- Workspace initialization
- File system operations

### 5. Container Health Testing
**Purpose**: Verify container startup and health
- Health endpoint responses
- Readiness check validation
- Startup script environment
- Error simulation and recovery

## Debugging & Troubleshooting

### Debug Mode Support
```bash
# Enable debug logging
DEBUG=marvin:* ./scripts/test-sandbox.sh

# Verbose test output
npm test -- --reporter=verbose

# Keep mock servers running
KEEP_MOCK_SERVER=true ./scripts/test-sandbox.sh
```

### Common Test Failures
1. **Database Connection Issues**
   - Verify DATABASE_URL configuration
   - Check PostgreSQL service status
   - Validate network connectivity

2. **Google Cloud Authentication**
   - Verify GOOGLE_APPLICATION_CREDENTIALS
   - Check service account permissions
   - Validate gcloud CLI installation

3. **Port Conflicts**
   - Check for existing processes on test ports
   - Use dynamic port allocation
   - Clean up zombie processes

### Manual Testing Support
```bash
# Test database connection
node -e "const {Pool} = require('pg'); new Pool({connectionString: process.env.DATABASE_URL}).query('SELECT NOW()').then(r => console.log(r.rows[0]))"

# Test Google Cloud auth
gcloud auth list
gcloud run services list --region=us-west1

# Test mock endpoints
curl http://localhost:PORT/health
curl http://localhost:PORT/ready?user=TEST_USER
```

## CI/CD Integration

### GitHub Actions Pattern
```yaml
- name: Run Integration Tests
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    GOOGLE_APPLICATION_CREDENTIALS: ${{ secrets.GCP_SA_KEY }}
    GCP_PROJECT_ID: marvin-openclaw
  run: |
    chmod +x scripts/test-sandbox.sh
    ./scripts/test-sandbox.sh
```

### Automation Requirements
- **Secrets management**: Secure handling of credentials
- **Service dependencies**: Proper dependency management
- **Cleanup procedures**: Automatic cleanup of test resources
- **Parallel execution**: Safe concurrent test execution

## Performance Considerations

### Test Execution Speed
- **Fast execution**: Target < 30 seconds for full test suite
- **Parallel execution**: Tests designed for concurrent execution
- **Resource efficiency**: Minimal resource usage during tests
- **Connection reuse**: Efficient database connection patterns

### External Service Usage
- **Read-only operations**: Prefer read-only Cloud Run operations
- **Minimal queries**: Database tests use minimal query operations
- **Connection pooling**: Efficient connection management
- **Timeout handling**: Reasonable timeouts for external calls

## Security Testing

### Authentication Testing
- **Token validation**: JWT token verification
- **Permission checks**: Service account permission validation
- **Credential isolation**: No sensitive data in logs or output
- **Test data separation**: Test data isolated from production

### Data Protection
- **No sensitive data**: Tests avoid using real sensitive data
- **Credential management**: Proper test credential lifecycle
- **Environment separation**: Clear test vs production separation
- **Cleanup verification**: Ensure test data is properly removed

## Best Practices Summary

1. **Integration over Unit**: Focus on integration testing for better real-world validation
2. **Environment Flexibility**: Support multiple test environments with graceful degradation
3. **Resource Management**: Proper setup, cleanup, and error handling
4. **Documentation**: Comprehensive test documentation and troubleshooting guides
5. **Performance**: Fast, efficient tests that respect external service limits
6. **Security**: Secure handling of credentials and test data
7. **Debugging**: Rich debugging support and failure diagnosis tools
8. **Maintenance**: Easy-to-maintain tests with clear patterns and conventions