# Gemforce Configuration (`gemforce.config.ts`)

The `gemforce.config.ts` file serves as the central configuration hub for the entire Gemforce platform, encompassing settings for smart contract deployments, cloud function behaviors, API endpoints, and external service integrations. Properly understanding and managing this configuration is crucial for deploying, operating, and customizing the Gemforce ecosystem.

## 1. Overview and Purpose

The `gemforce.config.ts` file (or its equivalent in other environments/projects) defines all environment-specific and platform-wide parameters. Its primary purposes include:

*   **Centralized Configuration**: All key parameters are managed from a single source, simplifying deployment and ensuring consistency.
*   **Environment Agnostic Deployment**: Allows the same codebase to be deployed across different networks (e.g., local, testnet, mainnet) by simply swapping configuration files or environment variables.
*   **External Service Integration**: Configures connections to various external services such as DFNS, blockchain nodes, email providers (SendGrid), and payment processors (Plaid).
*   **Smart Contract Addresses**: Stores the deployed addresses of Diamond contracts, facets, and other critical on-chain components.
*   **Feature Flags**: Can be used to enable or disable certain platform features.

## 2. Structure and Key Sections

The `gemforce.config.ts` typically follows a consistent structure, often exporting a configuration object. Below are common sections and their parameters:

```typescript
// Example gemforce.config.ts structure
export const GemforceConfig = {
  // 1. Core Platform Settings
  platform: {
    name: "Gemforce Platform",
    version: "1.0.0",
    defaultNetwork: "sepolia", // or "localhost", "optimismSepolia", etc.
  },

  // 2. Blockchain Settings
  blockchain: {
    // Current deployed Diamond address for each network
    diamondAddress: {
      localhost: "0x...",
      sepolia: "0x...",
      optimismSepolia: "0x...",
      // ...other networks
    },
    // RPC URLs for each network
    rpcUrls: {
      localhost: "http://localhost:8545",
      sepolia: "https://sepolia.infura.io/v3/YOUR_INFURA_PROJECT_ID",
      optimismSepolia: "https://opt-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_API_KEY",
      // ...
    },
    // Block explorer URLs
    blockExplorers: {
      sepolia: "https://sepolia.etherscan.io",
      optimismSepolia: "https://sepolia-optimism.etherscan.io",
      // ...
    },
    // Private key for deployer/admin account (for deployment or admin functions)
    privateKey: process.env.DEPLOYER_PRIVATE_KEY || "", // Loaded from environment variable
  },

  // 3. Parse Server & Cloud Functions Settings
  parseServer: {
    appId: "GEMFORCE_APP_ID",
    masterKey: process.env.PARSE_MASTER_KEY || "", // Securely loaded
    serverURL: "http://localhost:1337/parse", // Or deployed cloud URL
    cloudCodeMain: "./cloud/main.js", // Path to Cloud Code entry file
    databaseURI: process.env.DATABASE_URI || "mongodb://localhost:27017/gemforcedb",
    readOnlyMasterKey: process.env.PARSE_RO_MASTER_KEY || "", // For read-only access
  },

  // 4. External Service Integrations
  externalServices: {
    dfns: {
      apiKey: process.env.DFNS_API_KEY || "",
      privateKey: process.env.DFNS_PRIVATE_KEY || "",
      apiUrl: "https://api.dfns.io",
    },
    sendGrid: {
      apiKey: process.env.SENDGRID_API_KEY || "",
      senderEmail: "noreply@gemforce.xyz",
    },
    ipfs: {
      gatewayUrl: "https://ipfs.io/ipfs/",
      pinataApiKey: process.env.PINATA_API_KEY || "", // Example IPFS provider
      pinataSecretApiKey: process.env.PINATA_SECRET_API_KEY || "",
    },
    // ... other services like Plaid, Infura, Alchemy
  },

  // 5. Application Specific Settings
  appSettings: {
    // For front-end or specific cloud function logic that varies by environment
    marketplace: {
      enableTrading: true,
      featuredItems: ["token123", "token456"],
    },
    fees: {
      platformFeePercentage: 0.025, // 2.5%
    },
    // ...
  },

  // 6. Security & Audit Related Settings
  security: {
    allowUnsafeTransactions: false, // For development/testing only
    auditLogRetentionDays: 90,
  },
};
```

## 3. Configuration Management Best Practices

*   **Environment Variables**: **Never hardcode sensitive information** (private keys, API keys, database credentials) directly into `gemforce.config.ts`. Always load them from environment variables (e.g., `process.env.YOUR_VARIABLE_NAME`).
*   **Version Control**: Commit `gemforce.config.ts` (without sensitive values) to version control. Use `.env.example` files to document required environment variables.
*   **Environment-Specific Files**: For complex deployments, consider having multiple configuration files (e.g., `config.development.ts`, `config.production.ts`) and dynamically loading the appropriate one based on an `NODE_ENV` or similar environment variable.
*   **Validation**: Implement runtime validation for configuration parameters to ensure all necessary values are present and correctly formatted before application startup.
*   **Change Control**: Any changes to crucial configuration parameters should follow a strict change control process, including review, testing, and approval.
*   **Security for Configuration Management**: Ensure access to the systems and methods used to set environment variables (e.g., CI/CD pipelines, server configurations) is highly restricted and audited.