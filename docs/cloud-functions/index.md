# Cloud Functions API Documentation

## Overview

The Gemforce platform provides a comprehensive set of cloud functions built on Parse Server that handle blockchain interactions, user authentication, external service integrations, and business logic operations. These cloud functions serve as the bridge between the frontend applications and the underlying blockchain infrastructure.

## Architecture

The cloud functions are organized into several modules:

- **Authentication Functions** - User authentication and session management
- **Blockchain Functions** - Blockchain network interactions and provider management
- **Bridge API Functions** - External financial service integrations
- **Contract Functions** - Smart contract deployment and interaction
- **DFNS Functions** - Wallet-as-a-Service integration
- **Project Functions** - Project lifecycle management
- **Trade Deal Functions** - Trade deal operations and management

## Function Categories

### Core Infrastructure
- [Blockchain Functions](blockchain.md) - Network provider management and blockchain interactions
- [Authentication Functions](auth-functions.md) - User authentication and authorization
- [Contract Functions](contracts.md) - Smart contract deployment and management

### External Integrations
- [DFNS Functions](dfns.md) - Wallet-as-a-Service integration
- [Bridge API Functions](bridge.md) - Financial services and KYC integration
- [Project Functions](project.md) - Project management and configuration

### Business Logic
- [Trade Deal Functions](trade-deal.md) - Trade deal lifecycle management
- [Deploy Functions](deploy.md) - Automated deployment operations

## Authentication & Security

### API Authentication
All cloud functions require proper authentication:

```javascript
// Client-side authentication
Parse.User.logIn(username, password).then((user) => {
  // User is authenticated, can call cloud functions
  return Parse.Cloud.run("functionName", parameters);
});
```

### Session Management
- Session tokens are automatically managed by Parse SDK
- Sessions expire based on configuration settings
- Master key access for administrative functions

### Access Control
- Role-based access control through Parse roles
- Function-level permissions
- Parameter validation and sanitization

## Common Usage Patterns

### Calling Cloud Functions

#### From JavaScript/TypeScript
```javascript
// Basic cloud function call
const result = await Parse.Cloud.run("functionName", {
  parameter1: "value1",
  parameter2: "value2"
});

// With error handling
try {
  const result = await Parse.Cloud.run("loadBlockchainDataForNetwork", {
    networkId: 1337
  });
  console.log("Blockchain data:", result);
} catch (error) {
  console.error("Error:", error.message);
}
```

#### From REST API
```bash
# POST request to cloud function
curl -X POST \
  -H "X-Parse-Application-Id: YOUR_APP_ID" \
  -H "X-Parse-Session-Token: USER_SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"parameter1":"value1","parameter2":"value2"}' \
  https://your-server.com/parse/functions/functionName
```

### Error Handling

Cloud functions use consistent error handling:

```javascript
// Function implementation with error handling
Parse.Cloud.define("exampleFunction", async (request) => {
  try {
    // Validate parameters
    if (!request.params.requiredParam) {
      throw new Error("Missing required parameter: requiredParam");
    }
    
    // Perform operation
    const result = await someOperation(request.params);
    return result;
    
  } catch (error) {
    // Log error for debugging
    console.error("Error in exampleFunction:", error);
    throw new Error(`Operation failed: ${error.message}`);
  }
});
```

## Response Formats

### Success Response
```json
{
  "result": {
    "data": "function-specific-data",
    "status": "success"
  }
}
```

### Error Response
```json
{
  "code": 141,
  "error": "Error message describing what went wrong"
}
```

## Rate Limiting & Performance

### Rate Limiting
- Functions are subject to Parse Server rate limiting
- Implement client-side throttling for high-frequency calls
- Use batch operations where possible

### Performance Optimization
- Functions are cached where appropriate
- Database queries are optimized with proper indexing
- Async/await patterns for non-blocking operations

## Environment Configuration

### Required Environment Variables
```bash
# Parse Server Configuration
APP_ID=your_app_id
MASTER_KEY=your_master_key
SERVER_URL=https://your-server.com/parse

# Blockchain Configuration
ETH_NODE_URI_SEPOLIA=https://sepolia.infura.io/v3/your-key
ETH_NODE_URI_BASESEP=https://base-sepolia.infura.io/v3/your-key

# External Service Keys
DFNS_API_KEY=your_dfns_key
BRIDGE_API_KEY=your_bridge_key
SENDGRID_API_KEY=your_sendgrid_key
```

### Network Configuration
The system supports multiple blockchain networks:
- **Localhost** (networkId: 1337) - Development
- **Sepolia** (networkId: 11155111) - Ethereum testnet
- **Base Sepolia** (networkId: 84532) - Base testnet
- **OP Sepolia** (networkId: 11155420) - Optimism testnet

## Monitoring & Logging

### Function Logging
```javascript
Parse.Cloud.define("exampleFunction", async (request) => {
  // Log function entry
  console.log(`Function called: exampleFunction`, request.params);
  
  try {
    const result = await operation();
    
    // Log success
    console.log(`Function completed: exampleFunction`, result);
    return result;
    
  } catch (error) {
    // Log error with context
    console.error(`Function error: exampleFunction`, {
      error: error.message,
      params: request.params,
      stack: error.stack
    });
    throw error;
  }
});
```

### Performance Monitoring
- Function execution times are logged
- Database query performance is monitored
- External API call latencies are tracked

## Development Guidelines

### Function Structure
```javascript
Parse.Cloud.define("functionName", async (request) => {
  // 1. Parameter validation
  const { param1, param2 } = request.params;
  if (!param1) {
    throw new Error("Missing required parameter: param1");
  }
  
  // 2. Authentication/authorization checks
  if (!request.user) {
    throw new Error("Authentication required");
  }
  
  // 3. Business logic
  const result = await performOperation(param1, param2);
  
  // 4. Return result
  return result;
});
```

### Best Practices
- Always validate input parameters
- Use try-catch blocks for error handling
- Log important operations and errors
- Return consistent response formats
- Implement proper access control
- Use async/await for asynchronous operations

## Testing

### Unit Testing
```javascript
// Test cloud function
describe("Cloud Function Tests", () => {
  it("should return blockchain data", async () => {
    const result = await Parse.Cloud.run("loadBlockchainDataForNetwork", {
      networkId: 1337
    });
    
    expect(result).toHaveProperty("provider");
    expect(result).toHaveProperty("signer");
    expect(result).toHaveProperty("wallet");
  });
});
```

### Integration Testing
- Test with real blockchain networks
- Validate external service integrations
- Test error conditions and edge cases

## Deployment

### Local Development
```bash
# Start Parse Server with cloud functions
npm start

# Functions are automatically loaded from src/cloud-functions.ts
```

### Production Deployment
- Functions are deployed with Parse Server
- Environment variables must be configured
- Database migrations may be required

## Related Documentation

- [Parse Server Documentation](https://docs.parseplatform.org/)
- [Blockchain Integration Guide](../integration-guides/blockchain.md)
- [DFNS Integration Guide](../integration-guides/dfns.md)
- [Bridge API Integration Guide](../integration-guides/bridge-api.md)

---

*For detailed information about specific cloud functions, navigate to the individual function documentation pages.*