# Configuration Validation Procedures

Robust configuration validation is essential for the stability, security, and correct operation of the Gemforce platform. This document outlines the procedures and best practices for validating configuration parameters across smart contracts, cloud functions, and API services, ensuring that the platform operates as expected in all environments.

## 1. Importance of Configuration Validation

*   **Preventing Errors**: Catches misconfigurations early, preventing runtime errors and unexpected behavior.
*   **Enhancing Security**: Ensures sensitive parameters are correctly set and not exposed, and that security-critical flags are enabled.
*   **Ensuring Correctness**: Verifies that all necessary parameters are present and adhere to expected formats and value ranges.
*   **Improving Reliability**: Contributes to a more stable and predictable system by eliminating common sources of deployment failures.
*   **Facilitating Debugging**: Clear validation errors provide immediate feedback, simplifying troubleshooting.

## 2. Validation Layers

Configuration validation should be implemented at multiple layers of the application stack.

### 2.1. Build/Compile Time Validation

*   **Purpose**: Catching basic syntax errors or missing required fields before deployment.
*   **Tools**:
    *   **TypeScript**: Leveraging TypeScript's static type checking for `gemforce.config.ts` ensures that configuration objects conform to predefined interfaces.
    *   **Schema Validation (e.g., Zod, Joi)**: Define a schema for your configuration object and validate against it during the build process.
    *   **Linter Rules**: Custom ESLint rules can enforce specific naming conventions or value constraints.

### 2.2. Application Startup Validation

*   **Purpose**: Verify that all environment variables are loaded correctly, external services are reachable, and critical parameters are set before the application begins processing requests.
*   **Implementation**:
    *   **Early Exit**: If critical configurations are missing or invalid, the application should fail fast and exit with a clear error message.
    *   **Dependency Checks**: Verify connectivity to databases, RPC endpoints, and external APIs.
    *   **Value Range Checks**: Ensure numerical values (e.g., percentages, timeouts) are within acceptable ranges.
    *   **Format Checks**: Validate formats of URLs, addresses, and other structured strings.

### 2.3. Runtime/Operational Validation

*   **Purpose**: Monitor configuration integrity during operation and react to changes or inconsistencies.
*   **Implementation**:
    *   **Health Checks**: Include configuration checks in application health endpoints.
    *   **Monitoring and Alerting**: Set up alerts for unexpected configuration changes or failures in external service connections.
    *   **Periodic Checks**: For long-running services, periodically re-validate critical configurations.

## 3. Specific Validation Procedures

### 3.1. Smart Contract Configuration

While smart contracts themselves don't have a `gemforce.config.ts` in the same way, their deployment parameters and initial states are critical configurations.

*   **Deployment Scripts**:
    *   **Parameter Checks**: Ensure that all constructor arguments and initial state variables passed during deployment are valid (e.g., non-zero addresses, valid amounts, correct timestamps).
    *   **Network Verification**: Verify that the contract is being deployed to the intended network (e.g., `chainId` check).
    *   **Post-Deployment Checks**: After deployment, read back critical state variables from the deployed contract to ensure they match the intended configuration.
*   **Hardhat/Foundry Tests**: Write dedicated tests to validate that contract configurations (e.g., owner addresses, fee percentages, whitelisted roles) are correctly set after deployment.

### 3.2. Cloud Functions and API Configuration

*   **Environment Variable Presence**: At application startup, check for the presence of all required environment variables (e.g., `PARSE_MASTER_KEY`, `DFNS_API_KEY`).
*   **URL/Endpoint Reachability**: Attempt to connect to configured external service URLs (e.g., `DFNS_API_URL`, `blockchain.rpcUrls`) to ensure they are accessible.
*   **Credential Validation**: For external services, attempt a basic authenticated call (e.g., a health check endpoint) to validate API keys or credentials.
*   **Database Connection**: Verify successful connection to the configured database (`DATABASE_URI`).
*   **Schema Validation**: For incoming API requests or data processed by cloud functions, validate against a predefined schema (e.g., using JSON Schema, Zod, Joi) to ensure data integrity.

### 3.3. External Service Configuration

*   **API Key/Secret Format**: Validate that API keys and secrets conform to expected formats (e.g., length, character set).
*   **Endpoint Whitelisting**: Ensure that any configured external endpoints are whitelisted in network security configurations (e.g., firewalls, security groups).
*   **Rate Limit Awareness**: Configure clients for external services with awareness of their rate limits to prevent being blocked.

## 4. Example: TypeScript Configuration Validation

```typescript
// configSchema.ts (using Zod for schema validation)
import { z } from 'zod';

const rpcUrlsSchema = z.object({
  localhost: z.string().url(),
  sepolia: z.string().url(),
  optimismSepolia: z.string().url(),
});

const blockchainConfigSchema = z.object({
  diamondAddress: z.record(z.string(), z.string().startsWith("0x").length(42)), // Map of network -> address
  rpcUrls: rpcUrlsSchema,
  privateKey: z.string().min(64, "Private key must be at least 64 characters long (excluding 0x prefix)").startsWith("0x"),
});

const parseServerConfigSchema = z.object({
  appId: z.string().min(1),
  masterKey: z.string().min(1),
  serverURL: z.string().url(),
  databaseURI: z.string().url(),
});

const dfnsConfigSchema = z.object({
  apiKey: z.string().min(1),
  privateKey: z.string().min(1),
  apiUrl: z.string().url(),
});

export const gemforceConfigSchema = z.object({
  platform: z.object({
    name: z.string().min(1),
    version: z.string().regex(/^\d+\.\d+\.\d+$/), // Semantic versioning
    defaultNetwork: z.enum(["localhost", "sepolia", "optimismSepolia"]),
  }),
  blockchain: blockchainConfigSchema,
  parseServer: parseServerConfigSchema,
  externalServices: z.object({
    dfns: dfnsConfigSchema,
    // ... other external services
  }),
  appSettings: z.any(), // Define more specific schema as needed
  security: z.object({
    allowUnsafeTransactions: z.boolean(),
    auditLogRetentionDays: z.number().int().positive(),
  }),
});

// In your application's entry point (e.g., app.ts)
import { GemforceConfig } from './gemforce.config';
import { gemforceConfigSchema } from './configSchema';

try {
  gemforceConfigSchema.parse(GemforceConfig);
  console.log("Gemforce configuration validated successfully.");
} catch (error) {
  console.error("Invalid Gemforce configuration:", error.errors);
  process.exit(1); // Exit if configuration is invalid
}
```
