# Integrator's Guide: Authentication

Authentication is a critical component for any application integrating with the Gemforce platform. This guide outlines the various authentication mechanisms available, focusing on how your application can securely interact with Gemforce's public APIs, cloud functions, and smart contracts.

## Overview of Gemforce Authentication

Gemforce provides a multi-layered authentication strategy to accommodate different integration scenarios:

1.  **User Authentication (Parse Server Backend)**: For client-side applications (web, mobile) that need to manage user accounts, sessions, and interact with cloud functions on behalf of a specific user. This often involves username/password, email/password, or social logins.
2.  **API Key Authentication**: For server-to-server integrations or trusted backend services that need programmatic access to Gemforce's cloud functions and data, typically without a specific end-user context.
3.  **Smart Contract Interaction (Wallet-based)**: Direct interaction with on-chain smart contracts requires users to sign transactions using their blockchain wallets (e.g., MetaMask, WalletConnect). This is not a direct "authentication" to Gemforce's backend but rather a cryptographic signature validating an on-chain action.
4.  **DFNS Integration**: For enhanced security and enterprise-grade key management, Gemforce supports integration with DFNS for secure transaction signing.

## 1. User Authentication (Parse Server Backend)

Gemforce's cloud functions utilize a Parse Server backend. Client applications typically authenticate users directly with the Parse Server instance.

### Authentication Methods

-   **Username/Email & Password**: The most common method for user login.
-   **Social Logins**: Support for third-party authentication providers (e.g., Google, Facebook) can be configured.
-   **Session Tokens**: After successful login, the Parse Server returns a session token that should be stored securely on the client-side and included in subsequent API requests.

### Flow for Client Applications

1.  **Sign Up**:
    -   `POST /users` (Parse Server REST API)
    -   Example: Register a new user with username and password.
2.  **Log In**:
    -   `GET /login` (Parse Server REST API)
    -   Send username/password in `_ApplicationId`, `_ClientKey`, `_Username`, `_Password` headers.
    -   Receive `sessionToken` in response.
3.  **Authenticated Requests**:
    -   Include the `X-Parse-Session-Token` header in all subsequent requests to cloud functions or protected data.

### Example (JavaScript/TypeScript)

```javascript
// Assuming you have Parse SDK initialized or using direct REST calls
// Example using Fetch API for direct REST call
const PARSE_SERVER_URL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";
const APP_ID = "YOUR_PARSE_APP_ID";
const CLIENT_KEY = "YOUR_PARSE_CLIENT_KEY"; // Or Master Key for backend

// User Sign Up
async function signUp(username, password, email) {
    const response = await fetch(`${PARSE_SERVER_URL}/users`, {
        method: 'POST',
        headers: {
            'X-Parse-Application-Id': APP_ID,
            'X-Parse-REST-API-Key': CLIENT_KEY,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            username: username,
            password: password,
            email: email
        })
    });
    const data = await response.json();
    if (!response.ok) {
        throw new Error(data.error || 'Sign up failed');
    }
    console.log("Sign up successful:", data);
    return data; // Contains sessionToken
}

// User Log In
async function login(username, password) {
    const response = await fetch(`${PARSE_SERVER_URL}/login?username=${username}&password=${password}`, {
        method: 'GET',
        headers: {
            'X-Parse-Application-Id': APP_ID,
            'X-Parse-REST-API-Key': CLIENT_KEY // Use Client Key for client-side API
        }
    });
    const data = await response.json();
    if (!response.ok) {
        throw new Error(data.error || 'Login failed');
    }
    console.log("Login successful, session token:", data.sessionToken);
    return data.sessionToken;
}

// Authenticated Request to a Cloud Function
async function callCloudFunction(functionName, params, sessionToken) {
    const response = await fetch(`${PARSE_SERVER_URL}/functions/${functionName}`, {
        method: 'POST',
        headers: {
            'X-Parse-Application-Id': APP_ID,
            'X-Parse-REST-API-Key': CLIENT_KEY,
            'X-Parse-Session-Token': sessionToken, // Include session token
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(params)
    });
    const data = await response.json();
    if (!response.ok) {
        throw new Error(data.error || `Cloud function ${functionName} failed`);
    }
    console.log(`Cloud function ${functionName} response:`, data.result);
    return data.result;
}

// Example usage:
(async () => {
    try {
        // await signUp("testuser1", "password123", "test1@example.com");
        const token = await login("testuser1", "password123");
        const result = await callCloudFunction("helloWorld", { message: "from integrator" }, token);
    } catch (e) {
        console.error("Authentication error:", e.message);
    }
})();
```

## 2. API Key Authentication (Server-to-Server)

For backend services or scripts that need direct access to cloud functions without an explicit user session, Gemforce uses API keys. These keys are configured on the Parse Server and grant specific permissions.

### Types of API Keys

-   **REST API Key (`X-Parse-REST-API-Key`)**: Used for access from trusted client applications (if `masterKeyAsUser` is `false`) or for read-only access.
-   **Master Key (`X-Parse-Master-Key`)**: Grants full administrative privileges and bypasses all Class-Level Permissions (CLPs) and Access Control Lists (ACLs). **Use with extreme caution and only in secure server environments.**

### Example (Node.js)

```javascript
const express = require('express');
const ParseServer = require('parse-server').ParseServer;
const Parse = require('parse/node'); // Import Parse SDK for Node.js environments

// --- Initialize Parse SDK (usually done globally once) ---
Parse.initialize("YOUR_PARSE_APP_ID", "YOUR_PARSE_JAVASCRIPT_KEY", "YOUR_PARSE_MASTER_KEY");
Parse.serverURL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";

async function callCloudFunctionWithMasterKey(functionName, params) {
    try {
        // Use Parse.Cloud.run directly in Node.js, it will use the initialized Master Key
        const result = await Parse.Cloud.run(functionName, params, {
            useMasterKey: true // Use Master Key for this request
        });
        console.log(`Cloud function ${functionName} response:`, result);
        return result;
    } catch (e) {
        console.error(`Cloud function ${functionName} failed:`, e.message);
        throw e;
    }
}

// Example usage:
(async () => {
    try {
        const adminResult = await callCloudFunctionWithMasterKey("processAdminTask", { userId: "someUserId" });
    } catch (e) {
        console.error("Admin task error:", e.message);
    }
})();
```

## 3. Smart Contract Interaction (Wallet-based)

Direct interaction with Gemforce's smart contracts (Diamond, Facets, ERC-20/721/1155) occurs directly on the blockchain. This involves:

-   **Connecting a Web3 wallet**: Users connect their browser-based or mobile wallets (e.g., MetaMask, WalletConnect) to your dApp.
-   **Signing Transactions**: Users sign transactions using their private keys to perform on-chain operations (e.g., mint NFT, transfer tokens, interact with a trade deal).
-   **No Centralized Authentication**: There is no centralized authentication for smart contract calls; the blockchain validates the cryptographic signature against the sender's address.

### Key Tools & Libraries

-   **Ethers.js / Web3.js**: JavaScript libraries for interacting with the Ethereum blockchain.
-   **Wagmi / Web3Modal**: Libraries for simplifying wallet connection in frontend applications.
-   **Hardhat / Foundry**: Development environments for smart contract testing and deployment.

### Example (Ethers.js for signing a transaction)

```typescript
import { ethers } from 'ethers';
// Assuming you have the ABI for the smart contract you want to interact with
import MyContractABI from './MyContract.json'; 

const CONTRACT_ADDRESS = "0x..."; // Address of the deployed Gemforce smart contract
const PROVIDER_URL = "https://sepolia.base.org"; // Or use window.ethereum for browser dApps

async function interactWithSmartContract() {
    try {
        // Connect to the user's wallet
        // For browser environments, use window.ethereum
        // const provider = new ethers.providers.Web3Provider(window.ethereum);
        // await provider.send("eth_requestAccounts", []); // Request account access
        // const signer = provider.getSigner();

        // For Node.js/backend automation (using a private key)
        const provider = new ethers.JsonRpcProvider(PROVIDER_URL);
        const privateKey = "YOUR_PRIVATE_KEY"; // Keep this secure! For demo only.
        const signer = new ethers.Wallet(privateKey, provider);

        // Create a contract instance
        const myContract = new ethers.Contract(CONTRACT_ADDRESS, MyContractABI, signer);

        // Example: Call a function that changes state (requires gas)
        const tx = await myContract.someStateChangingFunction("some_parameter", { value: ethers.parseEther("0.1") });
        console.log("Transaction sent:", tx.hash);

        // Wait for the transaction to be mined
        const receipt = await tx.wait();
        console.log("Transaction confirmed in block:", receipt.blockNumber);

        // Example: Call a read-only function (does not require gas, no transaction)
        const result = await myContract.someViewFunction();
        console.log("View function result:", result);

    } catch (error) {
        console.error("Smart contract interaction error:", error);
    }
}

// interactWithSmartContract();
```

## 4. DFNS Integration

DFNS provides a secure, MPC-based Wallet-as-a-Service solution used within Gemforce. For integrators requiring advanced key management, higher security assurances, or compliance needs, direct integration with DFNS may be an option.

### Key Benefits

-   **Multi-Party Computation (MPC)**: Keys are split and never entirely reside in one place, enhancing security.
-   **Policy Engine**: Define granular policies for transaction signing (e.g., multi-signature approvals).
-   **Audit Trails**: Comprehensive logs of all key operations.

### Integration Approach

Typically, your backend service would communicate with the DFNS API, which then orchestrates the secure signing of blockchain transactions.

-   **API Keys/Authentication**: Authenticate with DFNS using their SDKs and API keys.
-   **Transaction Requests**: Submit transaction payloads to DFNS for signing.
-   **Policy Enforcement**: DFNS applies predefined policies (e.g., requiring multiple approvals) before signing.
-   **Signed Transaction**: DFNS returns a signed transaction that your service can then broadcast to the blockchain.

### Example (Conceptual with DFNS SDK)

```typescript
// This is a conceptual example, actual DFNS SDK usage may vary.
import { DfnsApiClient, AsymmetricKeys, SignatureMechanism, SignTransactionRequest } from '@dfns/sdk';

const DFNS_API_URL = "https://api.dfns.io";
const DFNS_APP_ID = "YOUR_DFNS_APP_ID";
const DFNS_AUTH_TOKEN = "YOUR_DFNS_AUTH_TOKEN"; // Generated from a private key & secret

async function signTransactionWithDFNS(walletId: string, chainId: string, transactionPayload: any) {
    try {
        const dfnsClient = new DfnsApiClient({
            baseUrl: DFNS_API_URL,
            appId: DFNS_APP_ID,
            authToken: DFNS_AUTH_TOKEN, // Or use a signing key provider
            authMethod: new AsymmetricKeys({
                 // This would involve loading private keys securely in a real app
                privateKey: "...", 
                signingKeyId: "...",
            })
        });

        const signRequest: SignTransactionRequest = {
            walletId: walletId,
            chainId: chainId,
            payload: JSON.stringify(transactionPayload),
            signatureMechanism: SignatureMechanism.JsonRpc,
            transactionType: "EvmTransaction", // or other types
        };

        const response = await dfnsClient.wallet.signTransaction(signRequest);
        console.log("DFNS Signed Transaction:", response.signedData);
        return response.signedData; // Contains the raw signed transaction to broadcast
    } catch (error) {
        console.error("DFNS signing error:", error);
        throw error;
    }
}

// Example usage:
// const signedTx = await signTransactionWithDFNS(
//     "some-dfns-wallet-uuid",
//     "eip155:11155111", // Sepolia chain ID
//     {
//         to: "0x...",
//         value: "0",
//         gasLimit: "21000",
//         data: "0x..."
//     }
// );
// // Then broadcast signedTx to the blockchain
```

## Related Documentation

-   [Cloud Functions: Authentication Functions](../cloud-functions/auth-functions.md)
-   [Cloud Functions: DFNS Functions](../cloud-functions/dfns.md)
-   [Parse Server REST API Documentation](https://docs.parseplatform.org/rest/guide/) (External)
-   [DFNS Documentation](https://docs.dfns.co/) (External)