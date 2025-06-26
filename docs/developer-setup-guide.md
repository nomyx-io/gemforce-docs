# Developer Setup Guide

## Overview

This guide provides step-by-step instructions for setting up a complete Gemforce development environment. Follow these instructions to get the platform running locally for development, testing, and integration work.

## Prerequisites

### System Requirements
- **Node.js**: Version 18.x or higher
- **npm**: Version 8.x or higher (comes with Node.js)
- **Git**: Latest version
- **MongoDB**: Version 5.0 or higher (local or cloud)
- **Operating System**: macOS, Linux, or Windows with WSL2

### Required Accounts
- **Infura Account**: For Ethereum network access
- **MongoDB Atlas Account**: For cloud database (optional)
- **DFNS Account**: For wallet services (optional for basic setup)
- **Bridge API Account**: For financial services (optional)

## Installation Steps

### 1. Clone the Repository

```bash
# Clone the main repository
git clone https://github.com/your-org/gem-base.git
cd gem-base

# Install dependencies
npm install
```

### 2. Environment Configuration

Create environment configuration files:

```bash
# Copy environment template
cp .env.example .env

# Edit environment variables
nano .env
```

#### Required Environment Variables

```bash
# Parse Server Configuration
APP_ID=your_unique_app_id
MASTER_KEY=your_secure_master_key
JAVASCRIPT_KEY=your_javascript_key
SERVER_URL=https://localhost:8337/parse
VERCEL_URL=https://localhost:8337/parse

# Database Configuration
MONGODB_URL=mongodb://localhost:27017/gemforce_dev
MONGODB_EVENTSOURCE_URL=mongodb://localhost:27017/gemforce_indexer

# Network Configuration
NETWORK_HOST=127.0.0.1
NETWORK_PORT=8337
HTTPS_ENABLED=true
HTTPS_PORT=443

# SSL Certificates (for HTTPS)
KEY_FILE=./privatekey.pem
CERT_FILE=./certificate.pem

# Blockchain Network URLs
ETH_NODE_URI_SEPOLIA=https://sepolia.infura.io/v3/YOUR_INFURA_KEY
ETH_NODE_URI_BASESEP=https://base-sepolia.infura.io/v3/YOUR_INFURA_KEY
ETH_NODE_URI_OPTSEP=https://optimism-sepolia.infura.io/v3/YOUR_INFURA_KEY

# Dashboard Configuration
DASHBOARD_PASSWORD=your_secure_password
DASHBOARD_ALLOW_INSECURE_HTTP=true

# External Services (Optional)
DFNS_API_KEY=your_dfns_api_key
BRIDGE_API_KEY=your_bridge_api_key
SENDGRID_API_KEY=your_sendgrid_api_key
PERSONA_WEBHOOK_SECRET=your_persona_webhook_secret
```

### 3. SSL Certificate Setup

For HTTPS development (recommended):

```bash
# Generate self-signed certificates
openssl req -x509 -newkey rsa:4096 -keyout privatekey.pem -out certificate.pem -days 365 -nodes

# Follow prompts, use 'localhost' as Common Name
```

### 4. Database Setup

#### Option A: Local MongoDB
```bash
# Install MongoDB (macOS with Homebrew)
brew tap mongodb/brew
brew install mongodb-community

# Start MongoDB service
brew services start mongodb-community

# Create databases
mongosh
> use gemforce_dev
> use gemforce_indexer
> exit
```

#### Option B: MongoDB Atlas (Cloud)
1. Create account at [MongoDB Atlas](https://www.mongodb.com/atlas)
2. Create a new cluster
3. Get connection string and update `MONGODB_URL` in `.env`

### 5. Blockchain Network Setup

#### Local Hardhat Network
```bash
# Start local blockchain (in separate terminal)
npx hardhat node

# This starts a local network on http://127.0.0.1:8545
# Network ID: 1337
```

#### Testnet Configuration
Update your `.env` file with Infura endpoints:

1. Create account at [Infura](https://infura.io/)
2. Create new project
3. Copy project ID and update environment variables

### 6. Smart Contract Deployment

```bash
# Compile contracts
npx hardhat compile

# Deploy to local network
npx hardhat run scripts/deploy.ts --network localhost

# Deploy to testnet (example: Sepolia)
npx hardhat run scripts/deploy.ts --network sepolia
```

### 7. Start the Development Server

```bash
# Start the main server
npm start

# Or use the direct command
npx hardhat run src/index.ts
```

The server will start on:
- **HTTP**: http://localhost:8337
- **HTTPS**: https://localhost:8337 (if SSL configured)
- **Parse Dashboard**: https://localhost:8337/dashboard

## Development Workflow

### Project Structure
```
gem-base/
├── contracts/              # Smart contracts
│   ├── facets/             # Diamond facets
│   ├── interfaces/         # Contract interfaces
│   ├── libraries/          # Utility libraries
│   └── tokens/             # Token contracts
├── src/                    # TypeScript source code
│   ├── cloud-functions/    # Parse cloud functions
│   ├── lib/               # Utility libraries
│   ├── tasks/             # CLI tasks
│   └── indexer/           # Event indexer
├── docs/                  # Documentation
├── test/                  # Test files
├── deploy/                # Deployment scripts
└── ui/                    # Frontend components
```

### Common Development Tasks

#### Running Tasks
```bash
# List available tasks
npx hardhat run src/tasks/index.ts

# Run specific task
npx hardhat run src/tasks/diamond.ts

# Deploy diamond contract
npx hardhat run src/tasks/deploy-diamond.ts --network localhost
```

#### Testing
```bash
# Run all tests
npm test

# Run specific test file
npx hardhat test test/Diamond.test.js

# Run tests with coverage
npm run test:coverage
```

#### Database Management
```bash
# Clear database (development only)
npx hardhat run src/tasks/clear-database.ts

# Export schemas
npx hardhat run src/tasks/export-all-schemas.ts

# Import test data
npx hardhat run src/tasks/import-test-data.ts
```

## IDE Configuration

### Visual Studio Code

Recommended extensions:
```json
{
  "recommendations": [
    "ms-vscode.vscode-typescript-next",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-eslint",
    "juanblanco.solidity",
    "nomicfoundation.hardhat-solidity"
  ]
}
```

Settings (`.vscode/settings.json`):
```json
{
  "typescript.preferences.importModuleSpecifier": "relative",
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "solidity.defaultCompiler": "remote",
  "solidity.compileUsingRemoteVersion": "v0.8.19"
}
```

### Debugging Configuration

Launch configuration (`.vscode/launch.json`):
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Gemforce Server",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/.bin/hardhat",
      "args": ["run", "src/index.ts"],
      "console": "integratedTerminal",
      "env": {
        "NODE_ENV": "development"
      }
    }
  ]
}
```

## Troubleshooting

### Common Issues

#### Port Already in Use
```bash
# Find process using port 8337
lsof -i :8337

# Kill process
kill -9 <PID>
```

#### SSL Certificate Issues
```bash
# Regenerate certificates
rm privatekey.pem certificate.pem
openssl req -x509 -newkey rsa:4096 -keyout privatekey.pem -out certificate.pem -days 365 -nodes
```

#### MongoDB Connection Issues
```bash
# Check MongoDB status
brew services list | grep mongodb

# Restart MongoDB
brew services restart mongodb-community

# Check connection
mongosh --eval "db.adminCommand('ismaster')"
```

#### Contract Compilation Errors
```bash
# Clean and recompile
npx hardhat clean
npx hardhat compile

# Check Solidity version compatibility
npx hardhat --version
```

### Performance Issues

#### Slow Database Queries
```javascript
// Add indexes to frequently queried fields
db.YourCollection.createIndex({ "fieldName": 1 })
```

#### Memory Issues
```bash
# Increase Node.js memory limit
export NODE_OPTIONS="--max-old-space-size=4096"
npm start
```

## Testing Your Setup

### 1. Verify Server Status
```bash
# Check server health
curl -k https://localhost:8337/parse/health

# Expected response: {"status":"ok"}
```

### 2. Test Cloud Functions
```bash
# Test blockchain function
curl -X POST \
  -H "X-Parse-Application-Id: your_app_id" \
  -H "X-Parse-Master-Key: your_master_key" \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://localhost:8337/parse/functions/loadAllBlockchains
```

### 3. Test Smart Contract Interaction
```javascript
// In Node.js console or test file
const { ethers } = require("ethers");
const provider = new ethers.providers.JsonRpcProvider("http://127.0.0.1:8545");
const blockNumber = await provider.getBlockNumber();
console.log("Current block:", blockNumber);
```

### 4. Access Parse Dashboard
1. Navigate to https://localhost:8337/dashboard
2. Login with credentials from `.env` file
3. Verify database connection and collections

## Next Steps

After completing the setup:

1. **Read the Architecture Documentation**: [System Architecture](system-architecture/gemforce-system-architecture.md)
2. **Explore Smart Contracts**: [Smart Contract Documentation](smart-contracts/index.md)
3. **Review Cloud Functions**: [Cloud Functions API](cloud-functions/index.md)
4. **Check Integration Guides**: [Integrator Guide](gemforce-integrator-guide.md)

## Development Best Practices

### Code Style
- Use TypeScript for all new code
- Follow ESLint configuration
- Use Prettier for code formatting
- Write comprehensive JSDoc comments

### Git Workflow
```bash
# Create feature branch
git checkout -b feature/your-feature-name

# Make changes and commit
git add .
git commit -m "feat: add new feature"

# Push and create pull request
git push origin feature/your-feature-name
```

### Testing Strategy
- Write unit tests for all new functions
- Test smart contracts thoroughly
- Use integration tests for cloud functions
- Test with multiple blockchain networks

## Support

If you encounter issues during setup:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review the [Documentation Gap Analysis](gemforce-documentation-gap-analysis.md)
3. Contact the development team
4. Create an issue in the project repository

---

*This guide covers the essential setup for Gemforce development. For advanced configuration and deployment, refer to the [Deployer Guide](gemforce-deployer-guide.md).*