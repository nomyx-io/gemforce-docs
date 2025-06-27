# Cloud Functions: Marketplace Integration Test Tasks

This document specifically details cloud functions designed for comprehensive integration testing of the Gemforce NFT Marketplace functionality. These functions extend general integration test capabilities to provide a streamlined way to set up, operate, and verify marketplace scenarios in a test environment.

## Overview

Marketplace integration test tasks enable:

-   **Test Listing Creation**: Creating various types of test NFT listings (fixed price, auction).
-   **Test Purchase Simulation**: Simulating purchases of listed NFTs.
-   **Test Data Cleanup**: Rapidly clearing marketplace-related test data.
-   **State Verification**: Checking the active state of listings, balances, and ownership after simulated interactions.

These functions are invaluable for ensuring the robustness and correctness of marketplace logic and integrations.

## Key Functions

### 1. `createTestListing`

Creates a test NFT listing for integration testing purposes. This function might abstract away approvals and other setup steps needed in a real scenario.

**Function Name**: `createTestListing`
**Method**: `POST`

**Parameters**:

-   `nftContractAddress` (String, required): Address of the test NFT contract.
-   `tokenId` (String, required): ID of the test NFT.
-   `price` (String, required): Listing price in Wei or base units.
-   `paymentTokenAddress` (String, required): Address of payment token (or `address(0)` for native ETH).
-   `sellerAddress` (String, required): The address of the test seller account.
-   `network` (String, required): The blockchain network.
-   `options` (Object, optional):
    -   `approveNFT` (Boolean, default: true): Whether to automatically approve the marketplace for NFT transfer.
    -   `listDuration` (Number, default: 86400): Duration of the listing in seconds.

**Returns**:

-   `listingId` (String): The ID of the created test listing.
-   `transactionHash` (String): Transaction hash of the listing.

**Example Request**:

```json
{
    "functionName": "createTestListing",
    "parameters": {
        "nftContractAddress": "0xTestNftContract",
        "tokenId": "1",
        "price": "100000000000000000", // 0.1 ETH
        "paymentTokenAddress": "0x0000000000000000000000000000000000000000",
        "sellerAddress": "0xTestSellerWallet",
        "network": "sepolia"
    }
}
```

### 2. `simulatePurchase`

Simulates a purchase of a test NFT listing.

**Function Name**: `simulatePurchase`
**Method**: `POST`

**Parameters**:

-   `listingId` (String, required): The ID of the test listing to purchase.
-   `buyerAddress` (String, required): The address of the test buyer account.
-   `value` (String, optional): The value to send with the transaction (for ETH payments).
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 3. `getTestListingDetails`

Retrieves details of a test listing.

**Function Name**: `getTestListingDetails`
**Method**: `GET` (or `POST` for Cloud Function)

**Parameters**:

-   `listingId` (String, required): The ID of the test listing.
-   `network` (String, required): The blockchain network.

**Returns**:

-   `listing` (Object): Detailed information about the test listing.

## Error Handling

Errors specific to marketplace integration tests typically involve:

-   **Invalid Test Data**: Using non-existent `nftContractAddress` or `tokenId`.
-   **Marketplace Logic Errors**: Issues arising from the marketplace smart contract's internal logic during simulated actions.
-   **Approval Failures**: If `approveNFT` is false or fails for some reason.
-   **Balance Issues**: The test seller/buyer accounts having insufficient funds or tokens.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) and [Cloud Functions: Integration Test Tasks](./integration-test.md) for more general error handling strategies.

## Security Considerations

-   **Testnet Isolation**: These functions should ONLY be used on testnets. They are designed for test environment manipulation and should absolutely not be exposed or callable in production environments.
-   **Access Control**: Strictly limit who can call these `test-` prefixed Cloud Functions. They often require `Master Key` access.
-   **Data Contamination**: Ensure that test operations do not accidentally affect or expose any production data. Use dedicated test databases and contracts.

## Related Documentation

-   [Smart Contracts: Marketplace Facet](../../smart-contracts/facets/marketplace-facet.md)
-   [Cloud Functions: Integration Test Tasks](./integration-test.md)
-   [Integrator's Guide: Testing](../../integrator-guide/testing.md)