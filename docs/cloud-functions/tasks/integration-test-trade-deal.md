# Cloud Functions: Trade Deal Integration Test Tasks

This document specifically details cloud functions designed for comprehensive integration testing of the Gemforce Trade Deal functionality. These functions provide a streamlined way to set up, operate, and verify various trade deal scenarios in a test environment, from creation to resolution or default.

## Overview

Trade Deal integration test tasks enable:

-   **Test Trade Deal Creation**: Creating test trade deals with predefined terms and participants.
-   **Test Fund/Collateral Operations**: Simulating funding by lender and collateral deposit by borrower.
-   **Test Resolution/Default**: Advancing trade deals to resolution or default states.
-   **Test Data Cleanup**: Rapidly clearing trade deal-related test data.
-   **State Verification**: Checking the active state of trade deals, token balances, and ownership after simulated interactions.

These functions are invaluable for ensuring the robustness and correctness of trade deal logic and integrations.

## Key Functions

### 1. `createTestTradeDeal`

Creates a test trade deal for integration testing purposes. This function might abstract away token approvals and other setup steps.

**Function Name**: `createTestTradeDeal`
**Method**: `POST`

**Parameters**:

-   `principalAmount` (String, required): Principal amount for the test trade deal.
-   `principalTokenAddress` (String, required): Address of the principal token.
-   `collateralAmount` (String, required): Collateral amount.
-   `collateralTokenAddress` (String, required): Address of the collateral token.
-   `borrowerAddress` (String, required): The address of the test borrower account.
-   `lenderAddress` (String, required): The address of the test lender account.
-   `maturityDate` (Number, required): Unix timestamp for maturity.
-   `network` (String, required): The blockchain network.
-   `options` (Object, optional):
    -   `autoApproveTokens` (Boolean, default: true): Whether to automatically approve tokens for transfer.

**Returns**:

-   `tradeDealId` (String): The ID of the created test trade deal.
-   `transactionHash` (String): Transaction hash of the trade deal creation.

**Example Request**:

```json
{
    "functionName": "createTestTradeDeal",
    "parameters": {
        "principalAmount": "100000000000000000000", // 100 PrincipalTokens
        "principalTokenAddress": "0xTestPrincipalToken",
        "collateralAmount": "150000000000000000000", // 150 CollateralTokens
        "collateralTokenAddress": "0xTestCollateralToken",
        "borrowerAddress": "0xTestBorrowerWallet",
        "lenderAddress": "0xTestLenderWallet",
        "maturityDate": 1735689600, // Jan 1, 2025
        "network": "sepolia"
    }
}
```

### 2. `simulateFundTradeDeal`

Simulates the lender funding a test trade deal.

**Function Name**: `simulateFundTradeDeal`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the test trade deal.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 3. `simulateDepositCollateral`

Simulates the borrower depositing collateral for a test trade deal.

**Function Name**: `simulateDepositCollateral`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the test trade deal.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 4. `simulateResolveTradeDeal`

Simulates the resolution of a test trade deal (`resolveTradeDeal` function).

**Function Name**: `simulateResolveTradeDeal`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the test trade deal.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 5. `simulateDefaultTradeDeal`

Simulates a trade deal going into default (`triggerDefault` function).

**Function Name**: `simulateDefaultTradeDeal`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the test trade deal.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

## Error Handling

Errors specific to trade deal integration tests typically involve:

-   **Invalid Test Data**: Using non-existent `tradeDealId`, or incorrect token addresses.
-   **Trade Deal Logic Errors**: Issues arising from the trade deal smart contract's internal logic during simulated actions.
-   **Approval Failures**: If `autoApproveTokens` is false or fails for some reason.
-   **Balance Issues**: The test borrower/lender accounts having insufficient funds or tokens.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) and [Cloud Functions: Integration Test Tasks](./integration-test.md) for more general error handling strategies.

## Security Considerations

-   **Testnet Isolation**: These functions should ONLY be used on testnets. They are designed for test environment manipulation and should absolutely not be exposed or callable in production environments.
-   **Access Control**: Strictly limit who can call these `test-` prefixed Cloud Functions. They often require `Master Key` access.
-   **Data Contamination**: Ensure that test operations do not accidentally affect or expose any production data. Use dedicated test databases and contracts.

## Related Documentation

-   [Smart Contracts: Trade Deal Management Facet](../../smart-contracts/facets/trade-deal-management-facet.md)
-   [Smart Contracts: ITradeDeal Interface](../../smart-contracts/interfaces/itradedeal.md)
-   [Cloud Functions: Integration Test Tasks](./integration-test.md)
-   [Integrator's Guide: Testing](../../integrator-guide/testing.md)