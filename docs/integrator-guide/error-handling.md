# Integrator's Guide: Error Handling

Robust error handling is crucial for building reliable integrations with the Gemforce platform. This guide provides an overview of common error types you might encounter when interacting with Gemforce's smart contracts, REST API, and cloud functions, along with strategies and best practices for managing them.

## Overview of Error Types

Errors in the Gemforce ecosystem can primarily originate from three layers:

1.  **Blockchain/Smart Contract Errors**: Occur during on-chain transactions or calls. These include reverted transactions, out-of-gas errors, or custom revert messages from smart contracts.
2.  **REST API Errors (Parse Server)**: Standard HTTP errors and custom Parse Server errors returned by the backend API.
3.  **Cloud Function Errors**: Specific errors returned by Gemforce's custom Cloud Functions, often indicating business logic failures or invalid input.
4.  **Network/Infrastructure Errors**: General network connectivity issues, timeouts, or problems with RPC providers.

## 1. Smart Contract Error Handling

When interacting with Gemforce smart contracts, transactions can fail for several reasons. Web3 libraries like Ethers.js and Web3.js provide mechanisms to catch and interpret these errors.

### Common Smart Contract Error Scenarios:

-   **Transaction Revert**: The most common error. A smart contract function execution fails due to a `require()`, `revert()`, or other exception.
-   **Out of Gas**: The transaction runs out of gas before completing.
-   **Invalid Input**: Parameters passed to a contract function are incorrect (e.g., wrong type, out of range).
-   **Access Control Violations**: The calling address does not have the necessary permissions (e.g., `Ownable` modifier failing).

### Example (Ethers.js)

```typescript
import { ethers } from 'ethers';
// Assuming ABI for a Gemforce contract with a `performAction` function that might revert
import GemforceContractABI from './GemforceContract.json'; 

const CONTRACT_ADDRESS = "0x..."; // Your Gemforce Diamond/facet address
const PROVIDER_URL = "https://sepolia.base.org"; 
const PRIVATE_KEY = "YOUR_PRIVATE_KEY"; // For demonstration, use secure methods in production

async function executeSmartContractAction() {
    const provider = new ethers.JsonRpcProvider(PROVIDER_URL);
    const signer = new ethers.Wallet(PRIVATE_KEY, provider); 
    const myContract = new ethers.Contract(CONTRACT_ADDRESS, GemforceContractABI, signer);

    try {
        console.log("Attempting to execute smart contract action...");
        const tx = await myContract.performAction("invalid_param", { gasLimit: 300000 }); // Example with potentially invalid param
        const receipt = await tx.wait(); // Wait for transaction to be mined

        console.log("Transaction successful:", receipt);
    } catch (error: any) {
        console.error("Smart Contract Transaction Failed!");
        console.error("Error message:", error.message);
        
        // Ethers.js specific error parsing
        if (error.code === ethers.errors.CALL_EXCEPTION) {
            console.error("This is a contract call exception (revert).");
            // Attempt to decode revert message (if available and standard)
            if (error.data) {
                try {
                    // This often works for custom revert strings if using ethers v5+ and target chain supports it
                    const decodedError = myContract.interface.parseError(error.data);
                    if (decodedError) {
                        console.error("Decoded Revert Reason (Ethers.js v5+):", decodedError.args[0]);
                    }
                } catch (decodeError) {
                    console.error("Could not decode revert reason from error.data");
                    // Fallback for older ethers versions or non-standard reverts
                    const revertReasonMatch = error.message.match(/reverted with reason string '([^']*)'/);
                    if (revertReasonMatch && revertReasonMatch[1]) {
                        console.error("Fallback Revert Reason:", revertReasonMatch[1]);
                    }
                }
            } else if (error.reason) { // Ethers.js v5 error.reason
                console.error("Revert Reason:", error.reason);
            }
        } else if (error.code === ethers.errors.UNPREDICTABLE_GAS_LIMIT) {
            console.error("Gas limit could not be estimated. This often implies a revert in a dry-run.");
            console.error("Raw error response:", error.error?.message || error.data);
        } else if (error.code === 'INSUFFICIENT_FUNDS') {
            console.error("Account has insufficient funds for transaction.");
        } else if (error.code === 'NETWORK_ERROR') {
            console.error("Network error encountered:", error.reason);
        } else {
            console.error("An unknown blockchain error occurred.");
        }
        // Provide user-friendly feedback based on error type
    }
}

// executeSmartContractAction();
```

### Best Practices for Smart Contract Errors:

-   **Catch All Errors**: Always wrap transaction calls in `try-catch` blocks.
-   **Parse Revert Messages**: If possible, extract and display the custom revert reason string defined in `require("reason string")` within the contract.
-   **Distinguish Errors**: Differentiate between network errors, gas errors, and transaction reverts to provide precise feedback.
-   **Gas Estimation**: Use `contract.estimateGas.functionName()` before sending transactions to catch potential reverts early. If `estimateGas` fails, the transaction will almost certainly revert.
-   **On-chain Validation**: Ensure comprehensive validation within smart contracts using `require()` and `revert()` with informative messages.

## 2. REST API Error Handling (Parse Server)

Gemforce's Parse Server REST API adheres to standard HTTP status codes but also includes specific JSON error payloads for more detail.

### Common REST API Error Status Codes:

-   **`400 Bad Request`**: Invalid input format, missing required parameters.
-   **`401 Unauthorized`**: Missing or invalid `X-Parse-Application-Id`, `X-Parse-Session-Token`, or other authentication headers.
-   **`403 Forbidden`**: Insufficient permissions (ACL/CLP violation) for the authenticated user/keys.
-   **`404 Not Found`**: Endpoint or object not found.
-   **`500 Internal Server Error`**: General server-side error, often caused by uncaught exceptions in Cloud Functions.

### Standard Parse Server Error Payload:

```json
{
  "code": 101, // Parse-specific error code
  "error": "Object not found." // Human-readable error message
}
```

### Example (JavaScript Fetch API)

```javascript
const PARSE_SERVER_URL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";
const APP_ID = "YOUR_PARSE_APP_ID";
const REST_API_KEY = "YOUR_PARSE_REST_API_KEY"; // or session token for user
const SESSION_TOKEN = "YOUR_SESSION_TOKEN"; // if authenticated user

async function fetchProjectData(projectId: string) {
    try {
        const response = await fetch(`${PARSE_SERVER_URL}/classes/Project/${projectId}`, {
            method: 'GET',
            headers: {
                'X-Parse-Application-Id': APP_ID,
                'X-Parse-REST-API-Key': REST_API_KEY,
                'X-Parse-Session-Token': SESSION_TOKEN // Include if fetching protected data
            }
        });

        if (!response.ok) { // Check for HTTP error status codes (4xx, 5xx)
            const errorData = await response.json();
            console.error(`API Error: HTTP Status ${response.status}, Parse Code ${errorData.code || 'N/A'}`);
            throw new Error(`Failed to fetch project: ${errorData.error || 'Unknown API error'}`);
        }

        const data = await response.json();
        console.log("Project data:", data);
        return data;

    } catch (error) {
        console.error("Network or parsing error:", error);
        throw error; // Re-throw to propagate for higher-level handling
    }
}

// Example usage:
// fetchProjectData("nonexistentProjectId")
//   .catch(err => console.error("Caught error:", err.message));
```

### Best Practices for REST API Errors:

-   **Check `response.ok`**: Always check `response.ok` (or equivalent in your HTTP client) before parsing the JSON body.
-   **Parse Error Payloads**: Always parse the JSON error payload to extract the `code` and `error` messages for specific handling and user feedback.
-   **Retry Logic**: For transient errors (e.g., network issues, `5xx` server errors), implement exponential backoff and retry mechanisms.
-   **Logging/Monitoring**: Log API errors on your server-side for debugging and monitoring. Set up alerts for critical errors.

## 3. Cloud Function Error Handling

Cloud Functions are custom code executed on the Parse Server. Errors from Cloud Functions are returned via the standard Parse REST API error payload, but their `error` messages can be more specific to your business logic.

### Example (JavaScript Parse SDK - Node.js)

```javascript
const Parse = require('parse/node');

// Assuming Parse SDK is initialized
Parse.initialize("YOUR_APP_ID", "YOUR_JAVASCRIPT_KEY", "YOUR_MASTER_KEY");
Parse.serverURL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";

async function callCustomCloudFunction(functionName: string, params: object) {
    try {
        // Assume this cloud function 'processTradeDeal' can throw specific errors
        const result = await Parse.Cloud.run(functionName, params, { useMasterKey: true }); // Example using Master Key
        console.log(`Cloud function ${functionName} result:`, result);
        return result;
    } catch (error: any) {
        console.error(`Cloud function ${functionName} failed:`);
        console.error(`Parse Error Code: ${error.code}`);
        console.error(`Error Message: ${error.message}`);

        // Handle specific Parse error codes or custom messages
        if (error.code === 141) { // Example: Cloud Code function failed with specific error
            if (error.message.includes("Invalid Trade Deal ID")) {
                console.error("User-friendly message: The trade deal ID provided is not valid.");
            } else if (error.message.includes("Insufficient collateral")) {
                console.error("User-friendly message: Not enough collateral for this trade deal.");
            }
        } else if (error.code === 209) { // Session Token Expired
            console.error("User-friendly message: Your session has expired. Please log in again.");
            // Trigger re-authentication flow
        }
        throw error; // Re-throw to propagate if needed
    }
}

// Example usage:
// callCustomCloudFunction("processTradeDeal", { tradeDealId: "invalid", amount: 100 })
//   .catch(err => console.error("Cloud function call failed at top level:", err.message));
```

### Best Practices for Cloud Function Errors:

-   **Consistent Error Responses**: Design your Cloud Functions to return consistent error structures (e.g., `throw new Parse.Error(Parse.Error.SCRIPT_FAILED, "Custom error message");`) to make client-side handling predictable.
-   **Informative Messages**: Provide clear, descriptive error messages that help integrators understand the cause of the failure.
-   **Specific Error Codes**: For critical business logic failures, consider mapping them to custom error codes if `Parse.Error` doesn't cover them.

## 4. Network and Infrastructure Error Handling

These are general issues that prevent successful communication with the Gemforce platform.

### Common Scenarios:

-   **Connection Refused**: Server is down or inaccessible.
-   **Timeout**: Request takes too long to respond.
-   **DNS Resolution Failure**: Domain name cannot be resolved.

### Best Practices:

-   **Retry Logic**: Implement robust retry mechanisms with exponential backoff and jitter for transient network failures.
-   **Circuit Breaker Pattern**: For persistent failures, consider a circuit breaker to prevent overwhelming a failing service and to fail fast.
-   **Monitoring**: Have monitoring and alerting in place for your integration points to detect widespread network issues.
-   **Fallback Mechanisms**: If critical, consider fallback mechanisms (e.g., alternative RPC providers, cached data) when the primary service is unavailable.

## General Error Handling Principles

-   **Fail Gracefully**: Your application should not crash due to an expected error.
-   **User Feedback**: Translate technical errors into user-friendly messages.
-   **Logging**: Implement comprehensive logging (client-side and server-side) to record errors for debugging and analysis.
-   **Alerting**: Set up automated alerts for critical errors in production environments.
-   **Debugging Tools**: Utilize browser developer tools, server logs, and blockchain explorers (for smart contract transactions) to debug issues.

## Related Documentation

-   [ integrators-guide/authentication.md ](../integrator-guide/authentication.md)
-   [Parse Server Error Codes](https://docs.parseplatform.org/js/guide/#error-codes) (External)
-   [Ethers.js Errors](https://docs.ethers.io/v5/api/utils/errors/) (External)