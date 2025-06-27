# Integrator's Guide: DFNS Integration

DFNS provides a highly secure Wallet-as-a-Service solution that Gemforce utilizes for cryptographic operations, particularly secure key management and transaction signing. For integrators working with sensitive operations that require advanced security, regulatory compliance, or robust policy enforcement, direct integration with the DFNS API through Gemforce's ecosystem is crucial.

## Overview of DFNS in Gemforce

DFNS offers:

-   **Multi-Party Computation (MPC)**: Keys are never held in one place, significantly reducing the risk of single points of failure and theft.
-   **Policy Engine**: Define precise rules for transaction approvals, including multi-signature requirements, whitelisting/blacklisting, and spend limits.
-   **Comprehensive Audit Trails**: All cryptographic actions are logged, providing an immutable record for compliance and security monitoring.
-   **Scalability**: Supports high-volume transaction signing for enterprise applications.

Gemforce integrates with DFNS primarily through its Cloud Functions, abstracting some of the direct DFNS API interactions. However, understanding the underlying DFNS capabilities is beneficial for deeper integrations.

## Core DFNS Concepts

-   **Wallets**: DFNS manages virtual "wallets," which are essentially key pairs (or key shares in MPC).
-   **Signing Keys**: Used for API authentication with DFNS.
-   **Policies**: Rules attached to wallets that dictate when a transaction can be signed and by whom.
-   **Signer Authentication**: DFNS supports various methods to authenticate the "signer" (the human or system authorizing an action), including FIDO, API keys, and more.

## Integration Methods

You can interact with DFNS through Gemforce in two primary ways:

1.  **Via Gemforce Cloud Functions (Recommended for most cases)**:
    Gemforce's Cloud Functions (e.g., `dfns.ts`) encapsulate common DFNS operations, making it simpler to:
    -   Create and manage DFNS wallets.
    -   Initiate transaction signing requests.
    -   Monitor the status of signing operations.
    This method abstracts the direct DFNS API calls, allowing you to use your existing Parse Server authentication.

2.  **Direct DFNS API Integration (For advanced users)**:
    For highly customized workflows or scenarios where fine-grained control over DFNS is required, you can integrate directly with the DFNS API. This involves:
    -   Obtaining your own DFNS API credentials.
    -   Implementing DFNS SDKs or building direct API calls.
    -   Managing DFNS wallet IDs and policies externally.

## Integration via Gemforce Cloud Functions

Gemforce provides several Cloud Functions that wrap DFNS operations. These functions require appropriate authentication (Parse session token or Master Key) to be invoked.

### Example: Signing a Transaction via Gemforce Cloud Function

Let's assume a Gemforce Cloud Function named `dfnsSignTransaction` exists, which takes a transaction payload and returns a signed transaction.

```javascript
// Function: callCloudFunction (from Authentication guide)
// This example assumes 'targetWalletId' and 'transactionDetails' are prepared by your backend service.

async function signTransactionViaGemforce(sessionToken, targetWalletId, transactionDetails) {
    try {
        const result = await callCloudFunction("dfnsSignTransaction", {
            walletId: targetWalletId,
            payload: transactionDetails // e.g., { to: "0x...", value: "0", data: "0x..." }
        }, sessionToken);
        console.log("Signed transaction from Gemforce Cloud Function:", result);
        return result; // The signed, raw transaction string (e.g., "0x...")
    } catch (error) {
        console.error("Error signing transaction via Gemforce Cloud Function:", error);
        throw error;
    }
}

// Example usage (assuming 'sessionToken' is obtained from user login)
// (async () => {
//     const token = "YOUR_SESSION_TOKEN"; // Replace with actual session token
//     const walletId = "YOUR_DFNS_WALLET_UUID";
//     const txPayload = {
//         to: "0xRecipientAddress",
//         value: "10000000000000000", // 0.01 ETH in wei
//         gasLimit: "21000",
//         chainId: 11155111, // Sepolia
//         nonce: 5, // Get current nonce
//         // Add other fields like gasPrice, maxFeePerGas, maxPriorityFeePerGas for EIP-1559
//     };
//     try {
//         const signedTx = await signTransactionViaGemforce(token, walletId, txPayload);
//         // Now broadcast this signedTx to the blockchain using an Ethers.js or Web3.js provider
//         // const provider = new ethers.providers.JsonRpcProvider("YOUR_RPC_URL");
//         // await provider.sendTransaction(signedTx);
//         // console.log("Transaction broadcasted:", signedTx);
//     } catch (e) {
//         console.error("DFNS Integration Example Failed:", e.message);
//     }
// })();
```

## Direct DFNS API Integration (Advanced)

If using Gemforce Cloud Functions doesn't meet your specific needs, you can integrate directly with the DFNS API. This path gives you full control but requires careful management of DFNS credentials and a deeper understanding of their SDK.

### Prerequisites

-   A DFNS Account and Console Access.
-   Your DFNS Application ID and a long-lived API signing key pair.
-   DFNS SDK for your preferred language (e.g., TypeScript/JavaScript).

### Steps Involved

1.  **DFNS Client Initialization**: Authenticate your application with the DFNS API using your signing key.
2.  **Wallet Management (Optional)**: If you need to programmatically create or manage DFNS wallets, use their API.
3.  **Transaction Draft Creation**: Prepare the transaction object (e.g., Ethereum transaction details).
4.  **Signing Request**: Submit the transaction draft to DFNS for signing, potentially specifying policies or required authenticators.
5.  **Signer Authentication**: Depending on your policy, DFNS might require additional authentication from a human signer (e.g., via FIDO device). This part is usually handled out-of-band by the DFNS platform.
6.  **Receive Signed Transaction**: Once policies are met and signers authenticated, DFNS returns the cryptographically signed transaction.
7.  **Broadcast**: Broadcast the signed transaction to the relevant blockchain network.

### Example (Conceptual Direct DFNS SDK Usage - TypeScript)

This example extends the conceptual example from the [Authentication Guide](authentication.md).

```typescript
import { DfnsApiClient, AsymmetricKeys, SignatureMechanism, SignTransactionRequest } from '@dfns/sdk';
import { EvmTransaction } from '@dfns/sdk/codegen/datamodel/EvmTransaction'; // Import specific type

const DFNS_API_URL = "https://api.dfns.io"; // or your specific DFNS API endpoint
const DFNS_APP_ID = "YOUR_DFNS_APP_ID";
const DFNS_PRIVATE_KEY = `-----BEGIN PRIVATE KEY-----
... YOUR PRIVATE KEY ...
-----END PRIVATE KEY-----`; // Load securely from environment variables, not hardcoded!
const DFNS_SIGNING_KEY_ID = "sk-xxxxxxxxxxxxxxxxx"; // Your API Signing Key ID from DFNS Console

async function signEvmTransactionDirectlyWithDFNS(walletId: string, chainId: string, rawTransaction: EvmTransaction) {
    try {
        // 1. Initialize DFNS API Client with Asymmetric Key Authentication
        const dfnsClient = new DfnsApiClient({
            baseUrl: DFNS_API_URL,
            appId: DFNS_APP_ID,
            authMethod: new AsymmetricKeys({
                privateKey: DFNS_PRIVATE_KEY,
                signingKeyId: DFNS_SIGNING_KEY_ID,
            })
        });

        // 2. Prepare the signing request
        const signRequest: SignTransactionRequest = {
            walletId: walletId,
            chainId: chainId, // e.g., "eip155:11155111" for Sepolia
            payload: JSON.stringify(rawTransaction), // DFNS expects stringified payload
            signatureMechanism: SignatureMechanism.JsonRpc, // Or other mechanism like Eip1559, Legacy
            transactionType: "EvmTransaction", // Specify EVM transaction
        };

        // 3. Send the signing request
        const response = await dfnsClient.wallet.signTransaction(signRequest);
        
        console.log("DFNS direct signed data:", response.signedData);
        // response.signedData contains the fully signed transaction in hex format (e.g., "0x...")
        return response.signedData;

    } catch (error) {
        console.error("DFNS direct signing failed:", error);
        throw error;
    }
}

// Example Raw EVM Transaction Payload (fields depend on transaction type)
// const exampleEvmTx: EvmTransaction = {
//     from: "0x...", // Optional, but good practice
//     to: "0xTargetContractAddress",
//     value: "0x00", // Hex value for 0 ETH
//     data: "0x...", // Calldata for contract interaction
//     gasLimit: "0x5208", // Hex value for 21000
//     nonce: "0x0", // Hex value for nonce
//     gasPrice: "0x...", // or maxFeePerGas/maxPriorityFeePerGas for EIP-1559
//     type: "0x0" // For legacy transaction, "0x2" for EIP-1559
// };

// Example Usage:
// (async () => {
//     try {
//         const dfnsWalletUuid = "YOUR_DFNS_WALLET_UUID";
//         const sepoliaChainId = "eip155:11155111"; // DFNS chain ID format
//         const signedTxHex = await signEvmTransactionDirectlyWithDFNS(dfnsWalletUuid, sepoliaChainId, exampleEvmTx);
//         console.log("Transaction ready to broadcast:", signedTxHex);
//     } catch (e) {
//         console.error("Direct DFNS Example Failed:", e.message);
//     }
// })();
```

## Security Considerations

-   **Master Key / Private Key Management**: Never hardcode Master Keys or private keys into your application. Use environment variables, secure secret management services (like AWS Secrets Manager, HashiCorp Vault), or DFNS itself for storing and accessing sensitive credentials.
-   **DFNS Policy Enforcement**: Thoroughly define and test your DFNS policies to ensure that transactions are only signed under approved conditions.
-   **API Key Rotation**: Regularly rotate your DFNS API keys and any other API credentials you use.
-   **Audit Logs**: Regularly review DFNS audit trails for any suspicious activity.
-   **Error Handling**: Implement robust error handling for all DFNS API calls, as failures could indicate policy violations, service issues, or incorrect requests.

## Related Documentation

-   [ integrators-guide/authentication.md ](../integrator-guide/authentication.md)
-   [Cloud Functions: DFNS Functions](../cloud-functions/dfns.md)
-   [DFNS Documentation](https://docs.dfns.co/) (External)