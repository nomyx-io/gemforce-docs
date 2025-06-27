# Cloud Functions: Trade Deal Tasks

This document details the cloud functions designed for managing `TradeDeal` smart contracts within the Gemforce platform. These functions provide an abstracted API for creating, managing, and resolving collateralized finance instruments, streamlining the trade deal lifecycle for users and applications.

## Overview

Trade Deal tasks enable:

-   **Creation**: Initiating new trade deals with specified terms and collateral.
-   **Execution/Resolution**: Progressing the trade deal through its stages, including releasing collateral.
-   **Default/Dispute**: Managing scenarios where a trade deal defaults or enters a dispute state.
-   **Querying**: Retrieving detailed information about ongoing and historical trade deals.

These functions are critical for applications that facilitate decentralized finance agreements and asset-backed transactions on the Gemforce platform.

## Key Functions

### 1. `createTradeDeal`

Create a new trade deal with specified terms.

**Function Name**: `createTradeDeal`
**Method**: `POST`

**Parameters**:

-   `principalAmount` (String, required): The principal amount of the trade deal (as string to handle large numbers).
-   `collateralAmount` (String, required): The amount of collateral required.
-   `principalTokenAddress` (String, required): Address of the ERC20 token for the principal.
-   `collateralTokenAddress` (String, required): Address of the ERC20 token for the collateral.
-   `borrowerAddress` (String, required): The address of the borrower.
-   `lenderAddress` (String, required): The address of the lender.
-   `maturityDate` (Number, required): Unix timestamp for the deal's maturity.
-   `interestRate` (Number, optional): Annual interest rate (in basis points, e.g., 500 = 5%).
-   `collateralRatio` (Number, optional): The collateralization ratio (in percentage, e.g., 150 = 150%).
-   `network` (String, required): The blockchain network where the trade deal will be created.

**Returns**:

-   `tradeDealId` (String): A unique identifier (bytes32 hex string) for the new trade deal.
-   `transactionHash` (String): The transaction hash of the deal creation.

**Example Request**:

```json
{
    "functionName": "createTradeDeal",
    "parameters": {
        "principalAmount": "100000000000000000000", // 100 principal tokens
        "collateralAmount": "150000000000000000000", // 150 collateral tokens
        "principalTokenAddress": "0xPrincipalTokenAddress",
        "collateralTokenAddress": "0xCollateralTokenAddress",
        "borrowerAddress": "0xBorrowerWalletAddress",
        "lenderAddress": "0xLenderWalletAddress",
        "maturityDate": 1735689600, // Jan 1, 2025, 00:00:00 GMT
        "interestRate": 750, // 7.5%
        "network": "optimism-sepolia"
    }
}
```

**Example Response**:

```json
{
    "result": {
        "tradeDealId": "0xabc...efg",
        "transactionHash": "0xhij...klm"
    }
}
```

**Workflow**:

1.  Client application calls `createTradeDeal` Cloud Function.
2.  Cloud Function validates parameters.
3.  Calls the `create` function on the `TradeDealManagementFacet` (or analogous contract) of the Diamond.
4.  Returns the `tradeDealId` and transaction details.

### 2. `fundTradeDeal`

Lender funds a newly created trade deal by depositing the principal amount.

**Function Name**: `fundTradeDeal`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the trade deal to fund.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 3. `depositCollateral`

Borrower deposits the required collateral for a trade deal.

**Function Name**: `depositCollateral`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the trade deal.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 4. `resolveTradeDeal`

Resolves a trade deal, allowing the lender to claim principal + interest and releasing collateral to the borrower (if repaid).

**Function Name**: `resolveTradeDeal`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the trade deal to resolve.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 5. `triggerDefault`

Triggers a default on a trade deal, allowing the lender to claim collateral.

**Function Name**: `triggerDefault`
**Method**: `POST`

**Parameters**:

-   `tradeDealId` (String, required): The ID of the trade deal to default.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

## Error Handling

Trade Deal tasks can encounter various errors, including:

-   **Invalid Parameters**: Missing fields, incorrect token addresses.
-   **Blockchain Transaction Errors**: Transaction reverts (e.g., deal not found, insufficient funds, collateral not approved).
-   **State Transitions**: Attempting to fund an already funded deal, or resolve a defaulted deal.
-   **Authorization**: Caller not authorized to perform the action (e.g., only lender can fund).

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Collateral Management**: Ensure the collateral is securely held and released only under correct conditions.
-   **Token Approvals**: The borrower and lender must explicitly approve the `TradeDeal` contract (or a `TradeDeal` facet) to transfer their tokens before `depositCollateral` or `fundTradeDeal` can succeed. Your application should guide users through this step.
-   **Interest Calculation**: Verify the accuracy of interest rate calculations on-chain.
-   **Oracle Dependency**: If trade deals depend on external price feeds (e.g., for liquidation), ensure the oracle is secure and reliable.
-   **Access Control**: Strictly enforce roles (borrower, lender) for each action within the cloud functions.

## Related Documentation

-   [Smart Contracts: Trade Deal Management Facet](../../smart-contracts/facets/trade-deal-management-facet.md)
-   [Smart Contracts: ITradeDeal Interface](../../smart-contracts/interfaces/itradedeal.md)
-   [Smart Contracts: Trade Deal Operations Facet](../../smart-contracts/facets/trade-deal-operations-facet.md)
-   [EIP: Collateralized Trade Deal Standard](../../eips/EIP-DRAFT-Collateralized-Trade-Deal-Standard.md)
-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)