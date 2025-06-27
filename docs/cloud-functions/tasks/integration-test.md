# Cloud Functions: Integration Test Tasks

This document describes the cloud functions specifically designed to facilitate integration testing within the Gemforce ecosystem. These functions provide utilities to set up, manipulate, and tear down test environments, making it easier to perform end-to-end and integration tests across smart contracts, cloud functions, and external services.

## Overview

Integration Test tasks enable:

-   **Test Data Setup**: Creating and populating test data in Parse Server.
-   **Environment Manipulation**: Resetting specific components or states for clean test runs.
-   **Mocking External Services**: Directing test calls to mock services instead of live ones.
-   **Status Querying**: Retrieving the state of test-related data or processes.

These functions are invaluable for automated testing pipelines and for developers performing manual integration tests.

## Key Functions

### 1. `prepareTestEnvironment`

Sets up a clean or specific test environment state. This might involve clearing Parse collections, deploying specific smart contract configurations on a testnet, or seeding initial data.

**Function Name**: `prepareTestEnvironment`
**Method**: `POST`

**Parameters**:

-   `scenario` (String, required): Defines the test scenario to prepare (e.g., "empty", "initialUsers", "marketplaceSetup").
-   `network` (String, optional): The blockchain network to target if smart contract operations are involved.
-   `options` (Object, optional): Additional key-value pairs for scenario-specific configurations.

**Returns**:

-   `status` (String): "success" or "failure".
-   `message` (String): A descriptive message about the setup outcome.
-   `details` (Object, optional): Any test-specific details (e.g., deployed contract addresses).

**Example Request**:

```json
{
    "functionName": "prepareTestEnvironment",
    "parameters": {
        "scenario": "marketplaceSetup",
        "network": "sepolia",
        "options": {
            "initialNFTs": 5,
            "testAccountBalance": "1000000000000000000" // 1 ETH
        }
    }
}
```

**Example Response**:

```json
{
    "result": {
        "status": "success",
        "message": "Marketplace test environment prepared.",
        "details": {
            "marketplaceAddress": "0xabc...xyz",
            "testUserWallet": "0xuser...wallet"
        }
    }
}
```

### 2. `resetTestData`

Clears specific test data from Parse Server collections or resets smart contract states to a known clean state.

**Function Name**: `resetTestData`
**Method**: `POST`

**Parameters**:

-   `collections` (Array of Strings, optional): An array of Parse collection names to clear (e.g., ["_User", "Project", "Transactions"]). If empty, might reset default collections.
-   `network` (String, optional): The blockchain network to reset smart contract states on.

**Returns**:

-   `status` (String): "success" or "failure".
-   `message` (String): Outcome message.

### 3. `getTestState`

Retrieves the current state of test-related data or configurations. Useful for assertions within test cases.

**Function Name**: `getTestState`
**Method**: `GET` (or `POST` for Cloud Function)

**Parameters**:

-   `query` (Object, optional): A query object to specify what state to retrieve (e.g., `{ type: "userCount" }`).
-   `network` (String, optional)

**Returns**:

-   `data` (Object): The requested state data.

### 4. `finalizeTestRun`

Performs cleanup operations after a test run. This might involve deleting all test data, deactivating test accounts, or undeploying temporary smart contracts.

**Function Name**: `finalizeTestRun`
**Method**: `POST`

**Parameters**:

-   `options` (Object, optional): Options for cleanup (e.g., `deleteAllData: true`).

**Returns**:

-   `status` (String): "success" or "failure".

## Error Handling

Errors in integration test tasks typically stem from:

-   **Configuration Issues**: Invalid `scenario` or `options`.
-   **Permissions**: The Cloud Function lacks the necessary Parse Master Key or blockchain permissions to perform resets/deploys.
-   **External Service Failures**: Issues with the blockchain network or third-party test services.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Restricted Access**: Integration test functions often have powerful capabilities (e.g., deleting data, deploying contracts). They should be highly restricted in production environments, accessible only by specific administrative tools or during CI/CD pipelines.
-   **Master Key Usage**: Many of these functions will require the Parse Master Key. Ensure its secure management.
-   **Testnet Isolation**: Perform all integration testing on dedicated testnets (e.g., Sepolia, Base Sepolia) and never on a mainnet environment.
-   **Data Sensitivity**: Be cautious with any sensitive data used or generated during testing. Ensure it is properly cleaned up.

## Related Documentation

-   [Integrator's Guide: Testing](../../integrator-guide/testing.md)
-   [Deployment Guides: Multi-Network Deployment](../../deployment-guides/multi-network-deployment.md) (if dealing with testnet deployments)
-   [Cloud Functions: Deploy Functions](../../cloud-functions/deploy.md)