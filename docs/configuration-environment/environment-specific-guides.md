# Environment-Specific Configuration Guides

The Gemforce platform supports deployment across various environments, each with its unique configuration requirements. This guide provides detailed instructions and best practices for configuring the platform for different environments, including local development, testnets, and production.

## 1. Local Development Environment

For local development, the focus is on ease of setup, rapid iteration, and isolated testing.

### 1.1. Prerequisites

*   **Node.js and npm/yarn**: For running cloud functions, SDKs, and development tools.
*   **Docker and Docker Compose**: For containerized services like Parse Server and MongoDB.
*   **Hardhat/Foundry**: For local blockchain development and smart contract testing.

### 1.2. Configuration (`.env.local` or similar)

Sensitive configurations are typically managed via `.env` files, which are not committed to version control.

```dotenv
# .env.local example
# Blockchain
DEPLOYER_PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 # Hardhat default account 0
LOCAL_RPC_URL=http://localhost:8545

# Parse Server
PARSE_APP_ID=GEMFORCE_LOCAL_APP_ID
PARSE_MASTER_KEY=localMasterKey
PARSE_SERVER_URL=http://localhost:1337/parse
DATABASE_URI=mongodb://localhost:27017/gemforce_local_db

# External Services (local mocks or test credentials)
DFNS_API_KEY=local_dfns_api_key
DFNS_PRIVATE_KEY=local_dfns_private_key
SENDGRID_API_KEY=local_sendgrid_api_key
```

### 1.3. Setup Steps

1.  **Start Local Blockchain**:
    ```bash
    npx hardhat node
    # or for Foundry
    anvil
    ```
2.  **Start Local Parse Server & MongoDB**:
    ```bash
    docker-compose -f docker-compose.local.yml up -d
    ```
3.  **Deploy Smart Contracts**: Run deployment scripts targeting the local network.
    ```bash
    npx hardhat run scripts/deploy.js --network localhost
    ```
4.  **Run Cloud Functions**:
    ```bash
    npm run start:cloud-functions # or similar command
    ```

## 2. Testnet Environments (e.g., Sepolia, Optimism Sepolia)

Testnets are used for integration testing, staging, and demonstrating features in a public, yet non-production, environment.

### 2.1. Prerequisites

*   **Infura/Alchemy Project ID**: For reliable RPC access to public testnets.
*   **Testnet Funds**: Obtain test ETH or other tokens from faucets.
*   **DFNS Test Credentials**: Separate API keys and private keys for DFNS test environments.

### 2.2. Configuration (`.env.sepolia`, `.env.optimism_sepolia` or environment variables in CI/CD)

Sensitive values are typically managed as environment variables in CI/CD pipelines or secure secret management systems.

```dotenv
# .env.sepolia example
# Blockchain
DEPLOYER_PRIVATE_KEY=YOUR_SEPOLIA_DEPLOYER_PRIVATE_KEY
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_INFURA_PROJECT_ID
DIAMOND_ADDRESS_SEPOLIA=0x... # Deployed Diamond address on Sepolia

# Parse Server
PARSE_APP_ID=GEMFORCE_SEPOLIA_APP_ID
PARSE_MASTER_KEY=YOUR_SEPOLIA_PARSE_MASTER_KEY
PARSE_SERVER_URL=https://sepolia.api.gemforce.xyz/parse # Deployed Parse Server URL
DATABASE_URI=YOUR_SEPOLIA_DATABASE_URI # Managed by cloud provider

# External Services
DFNS_API_KEY=YOUR_SEPOLIA_DFNS_API_KEY
DFNS_PRIVATE_KEY=YOUR_SEPOLIA_DFNS_PRIVATE_KEY
SENDGRID_API_KEY=YOUR_SEPOLIA_SENDGRID_API_KEY
```

### 2.3. Deployment Steps

Deployment to testnets often involves CI/CD pipelines:

1.  **Build and Test**: Run all automated tests.
2.  **Deploy Smart Contracts**: Use deployment scripts configured for the target testnet.
3.  **Deploy Cloud Functions/API**: Deploy Parse Server instance and cloud functions to a cloud provider (e.g., AWS, GCP, Azure).
4.  **Update Configuration**: Update `gemforce.config.ts` or environment variables with newly deployed smart contract addresses and API endpoints.

## 3. Production Environment

The production environment demands the highest level of security, reliability, and performance.

### 3.1. Prerequisites

*   **Dedicated Infrastructure**: Production-grade cloud resources (e.g., managed databases, load balancers, auto-scaling groups).
*   **Mainnet Funds**: Real ETH or other tokens for transaction fees.
*   **Production DFNS Credentials**: Separate, highly secured credentials for production DFNS.
*   **Security Audits**: All components should have undergone thorough security audits.

### 3.2. Configuration (Secure Secret Management)

All sensitive configurations **must** be managed through secure secret management systems (e.g., AWS Secrets Manager, Google Secret Manager, HashiCorp Vault) and injected into the environment at runtime. **No sensitive data should ever be hardcoded or stored directly in `.env` files in production.**

```
# Example of how production environment variables might be set (abstracted)
# These are typically injected by CI/CD or orchestration tools
export DEPLOYER_PRIVATE_KEY=... # Highly restricted access
export MAINNET_RPC_URL=...
export DIAMOND_ADDRESS_MAINNET=...

export PARSE_APP_ID=GEMFORCE_PROD_APP_ID
export PARSE_MASTER_KEY=...
export PARSE_SERVER_URL=https://api.gemforce.xyz/parse
export DATABASE_URI=...

export DFNS_API_KEY=...
export DFNS_PRIVATE_KEY=...
export SENDGRID_API_KEY=...
```

### 3.3. Deployment Steps

Production deployments follow strict, automated, and audited CI/CD pipelines:

1.  **Code Review & Approval**: All code changes undergo rigorous review.
2.  **Automated Testing**: All tests (unit, integration, end-to-end) must pass.
3.  **Security Scans**: Automated security scans (SAST, DAST) are performed.
4.  **Staging Environment Deployment**: Deploy to a production-like staging environment for final validation.
5.  **Manual QA/UAT**: Thorough manual testing and User Acceptance Testing (UAT).
6.  **Phased Rollout/Blue-Green Deployment**: Minimize downtime and risk during deployment.
7.  **Monitoring & Rollback**: Implement comprehensive monitoring and define clear rollback procedures in case of issues.

By following these environment-specific configuration guides, the Gemforce platform can be deployed and operated securely and efficiently across its entire lifecycle.