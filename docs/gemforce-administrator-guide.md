# Gemforce Administrator Guide

## Table of Contents

- [System Overview](#system-overview)
- [Installation and Configuration](#installation-and-configuration)
- [User Management](#user-management)
- [Monitoring and Alerts](#monitoring-and-alerts)
- [Backup and Recovery](#backup-and-recovery)
- [Security Management](#security-management)
- [Troubleshooting](#troubleshooting)
- [Maintenance Procedures](#maintenance-procedures)
- [Performance Optimization](#performance-optimization)

## System Overview

### Architecture Components

The Gemforce platform consists of the following major components:

1. **Blockchain Smart Contracts**
   - Diamond contracts (EIP-2535 implementation)
   - Identity management contracts
   - Token management contracts
   - Carbon credit contracts

2. **Cloud Services**
   - Parse Server backend
   - Database (MongoDB)
   - File storage
   - Cloud functions

3. **External Service Integrations**
   - DFNS wallet-as-a-service
   - Bridge API
   - SendGrid email service
   - Blockchain RPC providers

4. **Client Applications**
   - Web applications
   - Mobile applications
   - API integrations

### Component Relationships

The Gemforce system follows a layered architecture:

```
Client Applications
        ↓↑
Cloud Services (Parse Server)
        ↓↑
External Services ⟷ Blockchain Contracts
```

- Client applications interact with the Parse Server via REST API
- Parse Server executes cloud functions that interact with blockchain contracts and external services
- External services provide specialized functionality (wallet management, financial operations, etc.)
- Blockchain contracts store and manage on-chain data and logic

### Infrastructure Requirements

- **Blockchain Nodes**: Access to Ethereum-compatible blockchain nodes
- **Server Hardware**: 
  - Minimum: 4 CPU cores, 8GB RAM, 100GB SSD
  - Recommended: 8 CPU cores, 16GB RAM, 250GB SSD
- **Database**: MongoDB 4.4+
- **Network**: Reliable internet connection with low latency to blockchain nodes
- **SSL Certificate**: Valid SSL certificate for secure API access

### Security Model Overview

Gemforce implements a multi-layered security approach:

1. **Authentication and Authorization**
   - User authentication via Parse Server
   - Role-based access control
   - API key authentication for B2B integrations
   - DFNS WebAuthn for wallet operations

2. **Smart Contract Security**
   - Role-based access control
   - Function-level permissions
   - Upgradeability via Diamond pattern

3. **Data Protection**
   - Encrypted data at rest
   - TLS for data in transit
   - Private key management via DFNS

4. **Monitoring and Auditing**
   - Comprehensive logging
   - Activity tracking
   - Alert systems

## Installation and Configuration

### Prerequisites

Before installing Gemforce, ensure you have:

- Node.js 16.x or higher
- MongoDB 4.4 or higher
- Access to blockchain nodes (RPC endpoints)
- API keys for external services
- Domain name with SSL certificate
- Git access to the Gemforce repositories

### Environment Variables

The Gemforce system requires several environment variables to be set. Create a `.env` file with the following variables:

```
# Parse Server Configuration
APP_ID=your_app_id
MASTER_KEY=your_master_key
DATABASE_URI=mongodb://username:password@host:port/database
SERVER_URL=https://your-server-url.com/parse
PROJECT_WIZARD_URL=https://your-server-url.com

# Blockchain Configuration
ETH_NODE_URI_MAINNET=https://mainnet.infura.io/v3/your-key
ETH_NODE_URI_BASESEP=https://sepolia.base.org
CHAIN_ID=base-sepolia
METADATA_BASE_URI=https://your-metadata-url.com/

# DFNS Configuration
DFNS_APP_ID=your_dfns_app_id
DFNS_API_URL=https://api.dfns.io
DFNS_CRED_ID=your_dfns_credential_id
DFNS_AUTH_TOKEN=your_dfns_auth_token

# Bridge API Configuration
BASE_BRIDGE_URL=https://api.bridge-api.com
BRIDGE_API_KEY=your_bridge_api_key

# Email Configuration
SENDGRID_API_KEY=your_sendgrid_key
FROM_EMAIL=noreply@your-domain.com

# Security Configuration
AUTH_SECRET_KEY=your_auth_secret_key
```

### Network Settings

1. **Firewall Configuration**:
   - Allow inbound connections on ports 80 (HTTP), 443 (HTTPS), and your Parse Server port
   - Allow outbound connections to MongoDB, blockchain nodes, and external APIs

2. **Load Balancer Configuration** (if applicable):
   - Configure health checks to the Parse Server health endpoint
   - Set appropriate timeouts (at least 30 seconds for blockchain operations)
   - Enable SSL termination

3. **DNS Configuration**:
   - Set up A records for your domain
   - Configure CNAME records for subdomains if needed

### Database Configuration

1. **MongoDB Setup**:

   ```bash
   # Create a MongoDB user for the Gemforce database
   mongo admin -u admin -p admin
   use gemforce
   db.createUser({
     user: "gemforce_user",
     pwd: "secure_password",
     roles: [{ role: "readWrite", db: "gemforce" }]
   })
   ```

2. **Indexes**:
   
   Ensure the following indexes are created for optimal performance:

   ```javascript
   // User collection indexes
   db.User.createIndex({ email: 1 }, { unique: true })
   db.User.createIndex({ username: 1 }, { unique: true })
   db.User.createIndex({ walletAddress: 1 })
   
   // Identity collection indexes
   db.Identity.createIndex({ walletAddress: 1 }, { unique: true })
   
   // Transaction collection indexes
   db.Transaction.createIndex({ hash: 1 }, { unique: true })
   db.Transaction.createIndex({ user: 1 })
   db.Transaction.createIndex({ createdAt: 1 })
   ```

### External Service Connections

1. **DFNS Setup**:
   - Create a DFNS account at https://dashboard.dfns.io
   - Create an application and credential
   - Copy the App ID and Credential ID to your environment variables
   - Store the private key in `dfns_private.key`

2. **Bridge API Setup**:
   - Obtain API credentials from Bridge API
   - Configure webhooks for notifications (if needed)
   - Set rate limiting based on expected traffic

3. **SendGrid Setup**:
   - Create a SendGrid account
   - Set up sender authentication for your domain
   - Create email templates for verification, password reset, etc.
   - Generate API key and add to environment variables

### Security Settings

1. **API Key Management**:
   - Rotate API keys periodically (recommended every 90 days)
   - Store API keys securely using environment variables
   - Never expose API keys in client-side code

2. **Cross-Origin Resource Sharing (CORS)**:
   
   Configure CORS settings in the Parse Server configuration:

   ```javascript
   const corsConfig = {
     allowOrigin: ['https://your-domain.com', 'https://app.your-domain.com'],
     allowHeaders: ['X-Parse-Application-Id', 'X-Parse-REST-API-Key', 'Content-Type'],
     allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
   };
   ```

3. **Rate Limiting**:

   Configure rate limiting to prevent abuse:

   ```javascript
   const rateLimitConfig = {
     rateLimit: 1000,  // requests per minute
     burstLimit: 50,   // concurrent requests
     expiration: 60    // seconds until reset
   };
   ```

## User Management

### User Roles and Permissions

Gemforce implements role-based access control with the following default roles:

1. **Admin**:
   - Full access to all functions
   - Can manage users and roles
   - Can deploy and update contracts

2. **CentralAuthority**:
   - Can manage trusted issuers
   - Can add claim topics
   - Can manage identities

3. **TrustedIssuer**:
   - Can issue claims to identities
   - Can verify identities
   - Limited access to identity management

4. **User**:
   - Can manage own wallet
   - Can view own transactions
   - Can participate in marketplace

### Adding and Removing Users

#### Adding Users

1. **Via Admin Dashboard**:
   - Navigate to User Management
   - Click "Add User"
   - Enter user details (email, name, role)
   - System will send invitation email

2. **Via API**:

   ```javascript
   // Example using Parse JavaScript SDK
   const user = new Parse.User();
   user.set("username", "user@example.com");
   user.set("password", "securePassword");
   user.set("email", "user@example.com");
   user.set("firstName", "John");
   user.set("lastName", "Doe");
   
   await user.signUp();
   ```

3. **Via Cloud Function**:

   ```javascript
   // Using the registerUser cloud function
   Parse.Cloud.run("registerUser", {
     username: "user@example.com",
     password: "securePassword",
     email: "user@example.com",
     firstName: "John",
     lastName: "Doe",
     company: "Example Inc"
   });
   ```

#### Removing Users

1. **Via Admin Dashboard**:
   - Navigate to User Management
   - Select user(s) to remove
   - Click "Delete" or "Deactivate"

2. **Via API**:

   ```javascript
   // Example using Parse JavaScript SDK
   const query = new Parse.Query(Parse.User);
   query.equalTo("email", "user@example.com");
   const user = await query.first({ useMasterKey: true });
   await user.destroy({ useMasterKey: true });
   ```

### Role Assignment

1. **Assigning Roles to Users**:

   ```javascript
   // Example using Parse JavaScript SDK
   const userQuery = new Parse.Query(Parse.User);
   userQuery.equalTo("email", "user@example.com");
   const user = await userQuery.first({ useMasterKey: true });
   
   const roleQuery = new Parse.Query(Parse.Role);
   roleQuery.equalTo("name", "TrustedIssuer");
   const role = await roleQuery.first({ useMasterKey: true });
   
   role.getUsers().add(user);
   await role.save(null, { useMasterKey: true });
   ```

2. **Creating New Roles**:

   ```javascript
   // Example using Parse JavaScript SDK
   const acl = new Parse.ACL();
   acl.setPublicReadAccess(true);
   
   const role = new Parse.Role("CustomRole", acl);
   await role.save(null, { useMasterKey: true });
   ```

### Identity Verification Processes

1. **KYC Verification**:
   - Initiate via `generateKycLink` cloud function
   - User completes KYC via Bridge API
   - Webhook notification sent back to Gemforce
   - Identity status updated accordingly

2. **Trusted Issuer Verification**:
   - Trusted Issuer reviews identity information
   - Issues verification claim to user identity
   - Claim is written to blockchain
   - Identity state is updated to "verified"

### Managing Trusted Issuers

1. **Adding a Trusted Issuer**:

   ```javascript
   // Using the dfnsAddTrustedIssuerInit/Complete cloud functions
   const { challenge, requestBody } = await Parse.Cloud.run("dfnsAddTrustedIssuerInit", {
     trustedIssuer: "0x1234...",  // Issuer wallet address
     claimTopics: [1, 2, 3],      // Claim topics this issuer can verify
     walletId: "wallet_id",
     dfns_token: "dfns_token"
   });
   
   // Sign the challenge client-side
   const signedChallenge = await signChallenge(challenge);
   
   // Complete the transaction
   await Parse.Cloud.run("dfnsAddTrustedIssuerComplete", {
     walletId: "wallet_id",
     dfns_token: "dfns_token",
     signedChallenge: signedChallenge,
     requestBody: requestBody
   });
   ```

2. **Removing a Trusted Issuer**:

   Use the `dfnsRemoveTrustedIssuerInit/Complete` cloud functions following the same pattern as above.

3. **Updating Trusted Issuer Claim Topics**:

   Use the `dfnsUpdateIssuerClaimTopicsInit/Complete` cloud functions.

## Monitoring and Alerts

### System Health Checks

1. **Parse Server Health Check**:
   - Endpoint: `/parse/health`
   - Expected response: `{"status":"ok"}`
   - Monitor response time (should be < 200ms)

2. **Database Health Check**:
   - Query execution time
   - Connection pool status
   - Replica set status (if applicable)

3. **Blockchain Connectivity**:
   - RPC endpoint response time
   - Block height synchronization
   - Transaction submission success rate

### Performance Metrics

Key metrics to monitor:

1. **API Performance**:
   - Request latency (avg, p95, p99)
   - Request throughput
   - Error rate

2. **Database Performance**:
   - Query execution time
   - Index usage
   - Connection count

3. **Blockchain Performance**:
   - Gas costs per transaction type
   - Transaction confirmation time
   - Failed transaction rate

### Log Management

1. **Log Aggregation**:
   - Implement centralized logging (e.g., ELK Stack, Splunk)
   - Include correlation IDs across service boundaries
   - Implement structured logging for easier querying

2. **Log Levels**:
   - ERROR: Issues requiring immediate attention
   - WARN: Potential issues to investigate
   - INFO: Normal operations
   - DEBUG: Detailed information for troubleshooting

3. **Key Events to Log**:
   - Authentication events
   - User management operations
   - Blockchain transactions
   - External API calls

### Alert Configuration

Configure alerts for the following scenarios:

1. **Critical Alerts** (immediate action required):
   - Parse Server unavailability
   - Database connection failures
   - High error rates (>5%)
   - Failed blockchain transactions

2. **Warning Alerts** (investigation needed):
   - Elevated API latency (>500ms)
   - Increased error rates (>1%)
   - Low disk space (<20%)
   - Delayed blockchain confirmations

3. **Notification Channels**:
   - Email
   - SMS/Text
   - Slack/Teams
   - PagerDuty or similar service

### Common Warning Signs

Watch for these indicators of potential issues:

1. **Increasing API Latency**: May indicate database issues or resource constraints
2. **Growing Database Size**: May require indexing or cleanup
3. **Increasing Error Rates**: May indicate bugs or external service issues
4. **Blockchain Transaction Failures**: May indicate gas price issues or contract bugs
5. **Declining User Activity**: May indicate UX issues or service degradation

### Dashboards Setup

Implement monitoring dashboards that show:

1. **System Overview**:
   - Overall health status
   - Current alert status
   - Key metrics summary

2. **API Performance**:
   - Request volume
   - Response time by endpoint
   - Error rate by endpoint

3. **User Activity**:
   - Active users
   - Registration rate
   - Transaction volume

4. **Blockchain Activity**:
   - Transaction success rate
   - Gas costs
   - Contract interactions

## Backup and Recovery

### Backup Procedures

1. **Database Backups**:

   ```bash
   # MongoDB backup command
   mongodump --uri="mongodb://username:password@host:port/database" --out=/backup/path/$(date +%Y-%m-%d)
   
   # Compress the backup
   tar -zcvf /backup/path/$(date +%Y-%m-%d).tar.gz /backup/path/$(date +%Y-%m-%d)
   ```

   **Schedule**:
   - Full backup: Daily
   - Incremental backup: Hourly

2. **Configuration Backups**:

   ```bash
   # Back up environment variables and config files
   cp .env /backup/config/$(date +%Y-%m-%d)-env
   cp gemforce.config.ts /backup/config/$(date +%Y-%m-%d)-gemforce-config.ts
   ```

   **Schedule**: After any configuration change

3. **Contract Deployment Records**:

   ```bash
   # Back up deployment records
   cp ./deployments /backup/deployments/$(date +%Y-%m-%d) -r
   cp deployed.json /backup/deployments/$(date +%Y-%m-%d)-deployed.json
   ```

   **Schedule**: After any contract deployment

### Recovery Procedures

1. **Database Recovery**:

   ```bash
   # Restore MongoDB database
   mongorestore --uri="mongodb://username:password@host:port/database" --drop /backup/path/YYYY-MM-DD
   ```

2. **Configuration Recovery**:

   ```bash
   # Restore configuration files
   cp /backup/config/YYYY-MM-DD-env .env
   cp /backup/config/YYYY-MM-DD-gemforce-config.ts gemforce.config.ts
   ```

3. **Contract Redeployment**:
   - Restore deployment records
   - Use the DiamondFactory to recreate diamonds if needed
   - Verify contract states

### Disaster Recovery Planning

1. **Disaster Recovery Scenarios**:
   - Database corruption
   - Server hardware failure
   - Cloud provider outage
   - Security breach

2. **Recovery Time Objectives (RTO)**:
   - Critical systems: 4 hours
   - Non-critical systems: 24 hours

3. **Recovery Point Objectives (RPO)**:
   - Database: 1 hour
   - Configuration: 24 hours

4. **Disaster Recovery Runbook**:
   - Maintain up-to-date documentation
   - Conduct periodic recovery tests
   - Automate recovery procedures where possible

## Security Management

### Access Control

1. **API Key Management**:
   - Generate strong API keys (min 32 characters)
   - Store securely (environment variables, secret management service)
   - Implement key rotation (90-day cycle)
   - Revoke compromised keys immediately

2. **User Authentication Controls**:
   - Enforce strong password policies
   - Implement account lockout after failed attempts
   - Consider implementing MFA for admin accounts
   - Session timeout (default: 24 hours)

3. **Role-Based Access Control**:
   - Limit permissions to minimum required
   - Regularly audit role assignments
   - Implement principle of least privilege

### API Key Rotation

Implement a process for rotating API keys:

```javascript
// Example: Rotate Bridge API key
async function rotateBridgeAPIKey() {
  // Generate a new API key (provider-specific)
  const newKey = await generateNewBridgeAPIKey();
  
  // Update environment variable
  process.env.BRIDGE_API_KEY = newKey;
  
  // Update configuration in database
  const config = await Config.get("bridgeAPIKey");
  config.set("value", newKey);
  await config.save(null, { useMasterKey: true });
  
  // Log the rotation
  console.log(`Bridge API key rotated at ${new Date().toISOString()}`);
}
```

### Audit Logging

1. **Security Events to Log**:
   - Authentication attempts (successful and failed)
   - Authorization changes
   - User creation/deletion
   - Role assignment
   - API key usage
   - Admin actions

2. **Log Format**:

   ```json
   {
     "timestamp": "2025-02-25T13:51:49.123Z",
     "event": "user.login",
     "success": true,
     "userId": "user123",
     "ipAddress": "192.168.1.1",
     "userAgent": "Mozilla/5.0...",
     "additionalDetails": {}
   }
   ```

3. **Log Retention**:
   - Security logs: 12 months minimum
   - Normal operation logs: 3 months

### Security Incident Response

1. **Incident Classification**:
   - P1: Critical (data breach, service unavailable)
   - P2: High (limited breach, partial service degradation)
   - P3: Medium (minor security issue, limited impact)
   - P4: Low (potential vulnerability, no active exploitation)

2. **Response Procedure**:
   - Identify and classify the incident
   - Contain the incident
   - Eradicate the cause
   - Recover systems
   - Conduct post-incident analysis

3. **Contact List**:
   - Security team
   - IT operations
   - Legal department
   - Executive leadership
   - External security consultants (if applicable)

### Compliance Considerations

1. **Data Privacy**:
   - Identify personal data stored in the system
   - Implement data minimization
   - Configure data retention policies
   - Provide data export/deletion capabilities

2. **Regulatory Compliance**:
   - KYC/AML requirements
   - Financial regulations
   - Industry-specific regulations
   - Cross-border data transfer requirements

## Troubleshooting

### Common Issues and Solutions

1. **API Request Failures**:
   - **Symptom**: HTTP 400/500 errors
   - **Check**: API logs, request parameters
   - **Solution**: Verify request format, check server logs for details

2. **Blockchain Transaction Failures**:
   - **Symptom**: Transaction hash returned but transaction fails
   - **Check**: Gas price, contract state, transaction parameters
   - **Solution**: Adjust gas price, verify contract accepts the transaction

3. **Database Connection Issues**:
   - **Symptom**: Cannot connect to database error
   - **Check**: MongoDB status, network connectivity
   - **Solution**: Restart MongoDB, check firewall rules

4. **DFNS Integration Issues**:
   - **Symptom**: "Cannot sign transaction" errors
   - **Check**: DFNS credentials, WebAuthn support
   - **Solution**: Verify DFNS credentials, ensure browser supports WebAuthn

### Diagnostic Tools

1. **Log Analysis**:
   ```bash
   # Search for errors in logs
   grep "ERROR" /var/log/gemforce/app.log
   
   # Find recent activity for a specific user
   grep "userId\":\"user123" /var/log/gemforce/app.log
   ```

2. **Database Queries**:
   ```javascript
   // Check user status
   db.User.findOne({ email: "user@example.com" })
   
   // Look for recent errors
   db.ErrorLog.find().sort({ createdAt: -1 }).limit(10)
   ```

3. **Blockchain Explorers**:
   - Use block explorers to verify transaction status
   - Check contract events for expected emissions
   - Verify contract state after transactions

### Error Codes Explanation

1. **HTTP Status Codes**:
   - 400: Bad Request (invalid parameters)
   - 401: Unauthorized (missing/invalid authentication)
   - 403: Forbidden (insufficient permissions)
   - 404: Not Found (resource doesn't exist)
   - 429: Too Many Requests (rate limit exceeded)
   - 500: Internal Server Error (server-side issue)

2. **Parse Error Codes**:
   - 101: Object not found
   - 141: Missing required field
   - 209: Invalid session token

3. **Blockchain Error Codes**:
   - "gas required exceeds allowance": Insufficient gas
   - "execution reverted": Contract condition not met
   - "nonce too low": Transaction nonce issue

### Support Escalation Procedures

1. **Tier 1 Support**:
   - Initial triage
   - Common issue resolution
   - Escalation timeframe: 30 minutes

2. **Tier 2 Support**:
   - Technical investigation
   - Complex issue resolution
   - Escalation timeframe: 2 hours

3. **Tier 3 Support**:
   - Engineering team involvement
   - Critical issue resolution
   - Escalation timeframe: 4 hours

4. **Escalation Contact Information**:
   - Tier 1: support@gemforce.com
   - Tier 2: tech-support@gemforce.com
   - Tier 3: engineering@gemforce.com
   - Emergency: +1-555-123-4567

### Service Dependencies

Map of service dependencies to check during outages:

1. **Parse Server depends on**:
   - MongoDB
   - File storage
   - DFNS API
   - Bridge API
   - Blockchain RPC nodes

2. **Blockchain operations depend on**:
   - RPC node availability
   - Gas price oracle
   - Contract state

3. **User authentication depends on**:
   - Parse Server
   - Email service
   - DFNS (for wallet operations)

## Maintenance Procedures

### Routine Maintenance Tasks

1. **Daily Tasks**:
   - Review error logs
   - Check backup status
   - Monitor system performance

2. **Weekly Tasks**:
   - Analyze API usage patterns
   - Review security logs
   - Check disk space usage

3. **Monthly Tasks**:
   - User access review
   - API key rotation
   - Performance optimization
   - Database maintenance

### Update Procedures

1. **Parse Server Updates**:

   ```bash
   # Update Parse Server
   npm update parse-server
   
   # Restart Parse Server
   pm2 restart parse-server
   ```

2. **Node.js Updates**:

   ```bash
   # Update Node.js using NVM
   nvm install 16.x
   nvm use 16.x
   
   # Verify version
   node -v
   ```

3. **Cloud Function Updates**:

   ```bash
   # Pull latest changes
   git pull origin main
   
   # Install dependencies
   npm install
   
   # Build the project
   npm run build
   
   # Restart the server
   pm2 restart gemforce
   ```

### Database Optimization

1. **Index Optimization**:

   ```javascript
   // Analyze query performance
   db.User.find({ walletAddress: { $exists: true } }).explain("executionStats")
   
   // Add missing indexes
   db.User.createIndex({ walletAddress: 1 })
   ```

2. **Data Archiving**:

   ```javascript
   // Move old logs to archive collection
   db.SystemLog.aggregate([
     { $match: { createdAt: { $lt: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) } } },
     { $out: "SystemLogArchive" }
   ])
   
   // Remove archived logs
   db.SystemLog.deleteMany({ createdAt: { $lt: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) } })
   ```

3. **Database Maintenance**:

   ```javascript
   // Repair database
   db.repairDatabase()
   
   // Compact collections
   db.runCommand({ compact: "User" })
   ```

### Cache Management

1. **Redis Cache Configuration** (if applicable):

   ```javascript
   // Example cache configuration
   const redisCache = {
     host: "localhost",
     port: 6379,
     ttl: 600 // 10 minutes
   };
   ```

2. **Cache Invalidation**:

   ```javascript
   // Invalidate specific keys
   redisClient.del("user_123_profile")
   
   // Invalidate pattern
   redisClient.keys("user_*_profile", (err, keys) => {
     if (keys.length > 0) redisClient.del(keys);
   })
   ```

3. **Cache Monitoring**:

   ```bash
   # Check Redis info
   redis-cli info
   
   # Monitor cache hit rate
   redis-cli info stats | grep hit_rate
   ```

### System Scaling Procedures

1. **Horizontal Scaling**:
   - Add more Parse Server instances
   - Configure load balancer
   - Update DNS if needed

2. **Vertical Scaling**:
   - Upgrade server resources
   - Schedule downtime for migration
   - Verify performance after upgrade

3. **Database Scaling**:
   - Implement MongoDB replica set
   - Consider sharding for large deployments
   - Optimize query patterns

## Performance Optimization

### Database Tuning

1. **Query Optimization**:
   - Use explain plan to analyze queries
   - Ensure proper indexes are in place
   - Limit returned fields using projection
   - Use aggregation pipeline for complex queries

2. **Connection Pooling**:

   ```javascript
   // Example MongoDB connection pool configuration
   const mongoConfig = {
     uri: process.env.DATABASE_URI,
     options: {
       maxPoolSize: 50,
       minPoolSize: 10,
       socketTimeoutMS: 30000,
       connectTimeoutMS: 30000
     }
   };
   ```

3. **Index Analysis**:

   ```javascript
   // Check index usage
   db.User.aggregate([
     { $indexStats: {} }
   ])
   
   // Remove unused indexes
   db.User.dropIndex("unusedIndex")
   ```

### Cache Configuration

1. **Cacheable Data Types**:
   - User profiles
   - Contract metadata
   - Configuration settings
   - Static content

2. **Cache Strategy**:
   - Cache-aside: Application checks cache before database
   - TTL-based expiration
   - Event-based invalidation

3. **Example Redis Configuration**:

   ```javascript
   // Redis cache configuration
   const redisOptions = {
     host: process.env.REDIS_HOST || "localhost",
     port: process.env.REDIS_PORT || 6379,
     password: process.env.REDIS_PASSWORD,
     db: 0,
     ttl: 3600 // 1 hour
   };
   ```

### Rate Limiting Configuration

1. **API Rate Limits**:

   ```javascript
   // Example rate limiting configuration
   const rateLimits = {
     global: {
       windowMs: 60 * 1000, // 1 minute
       max: 1000 // limit each IP to 1000 requests per minute
     },
     login: {
       windowMs: 60 * 1000, // 1 minute
       max: 10 // limit each IP to 10 login attempts per minute
     },
     createUser: {
       windowMs: 60 * 60 * 1000, // 1 hour
       max: 50 // limit each IP to 50 user creations per hour
     }
   };
   ```

2. **Rate Limit Response**:

   ```javascript
   // Example rate limit exceeded response
   {
     "status": "error",
     "code": 429,
     "message": "Rate limit exceeded. Try again in X seconds.",
     "retryAfter": 30
   }
   ```

3. **Rate Limit Monitoring**:
   - Track rate limit hits
   - Alert on sustained high rejection rates
   - Analyze traffic patterns to adjust limits

### Resource Allocation Guidelines

1. **Server Resources**:
   
   Guidelines for allocating resources based on load:

   | Load Level | Users | API Requests/min | CPU Cores | RAM | Disk |
   |------------|-------|------------------|-----------|-----|------|
   | Small      | <1k   | <100             | 2         | 4GB | 20GB |
   | Medium     | <10k  | <1k              | 4         | 8GB | 50GB |
   | Large      | <100k | <10k             | 8         | 16GB| 100GB|
   | X-Large    | >100k | >10k             | 16+       | 32GB+| 200GB+|

2. **Database Resources**:
   
   Guidelines for MongoDB resources:

   | Load Level | Documents | Indexes | RAM  | Disk    |
   |------------|-----------|---------|------|---------|
   | Small      | <1M       | <20     | 2GB  | 10GB    |
   | Medium     | <10M      | <50     | 4GB  | 50GB    |
   | Large      | <100M     | <100    | 16GB | 200GB   |
   | X-Large    | >100M     | >100    | 32GB+| 500GB+  |

3. **Blockchain Node Resources**:
   
   Consider using managed node providers for production environments.
   
   If running your own nodes:

   | Network        | Disk    | RAM  | Notes                           |
   |----------------|---------|------|----------------------------------|
   | Ethereum       | >2TB    | 16GB | Full node, growing rapidly      |
   | BaseSepolia    | >100GB  | 8GB  | Testnet, moderate growth        |

4. **Resource Scaling Triggers**:
   - CPU usage consistently >70%
   - RAM usage consistently >80%
   - Disk usage >85%
   - Response time increasing trend
   - Error rate increasing trend