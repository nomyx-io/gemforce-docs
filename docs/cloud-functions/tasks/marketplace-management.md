# Cloud Functions: Marketplace Management Tasks

This document details the cloud functions designed for managing the Gemforce NFT Marketplace. These functions provide an abstracted API for listing, unlisting, buying, and managing NFTs within the marketplace, streamlining interactions for both creators and consumers.

## Overview

Marketplace management tasks enable:

-   **Listing Management**: Creating and updating NFT listings.
-   **Purchase Execution**: Facilitating the buying of listed NFTs.
-   **Offer Management**: Handling bids and offers for NFTs.
-   **Royalty Distribution**: Ensuring proper distribution of royalties from sales.
-   **Marketplace Configuration**: Adjusting global marketplace settings (e.g., fees, whitelists).

These functions are critical for building responsive and user-friendly marketplace UIs and for automating marketplace operations.

## Key Functions

### 1. `createListing`

Create a new NFT listing on the Gemforce Marketplace.

**Function Name**: `createListing`
**Method**: `POST`

**Parameters**:

-   `nftContractAddress` (String, required): The address of the NFT contract (ERC721 or ERC1155).
-   `tokenId` (String, required): The ID of the NFT to list.
-   `price` (String, required): The listing price in Wei (or equivalent smallest unit of the payment token).
-   `paymentTokenAddress` (String, required): The address of the ERC20 token used for payment, or `address(0)` for native blockchain currency (e.g., ETH).
-   `duration` (Number, optional): The duration of the listing in seconds.
-   `currency` (String, optional): The symbol or name of the payment token (e.g., "ETH", "USDC"). For display purposes.
-   `network` (String, required): The blockchain network where the NFT and Marketplace are deployed.

**Returns**:

-   `listingId` (String): A unique identifier for the created listing.
-   `transactionHash` (String): The transaction hash of the listing creation on-chain.

**Example Request**:

```json
{
    "functionName": "createListing",
    "parameters": {
        "nftContractAddress": "0xabc...123",
        "tokenId": "456",
        "price": "100000000000000000", // 0.1 ETH
        "paymentTokenAddress": "0x0000000000000000000000000000000000000000", // ETH
        "duration": 86400, // 24 hours
        "network": "base-sepolia"
    }
}
```

**Example Response**:

```json
{
    "result": {
        "listingId": "0x789...def",
        "transactionHash": "0xghi...jkl"
    }
}
```

**Workflow**:

1.  Client application calls `createListing` Cloud Function.
2.  Before calling the MarketplaceFacet on-chain, the Cloud Function likely performs approvals (if required, e.g., for ERC721 `setApprovalForAll`).
3.  Calls the `listItem` function on the `MarketplaceFacet` of the Diamond contract.
4.  Returns the listing ID and transaction details.

### 2. `purchaseListing`

Purchase a listed NFT from the Gemforce Marketplace.

**Function Name**: `purchaseListing`
**Method**: `POST`

**Parameters**:

-   `listingId` (String, required): The unique identifier of the listing to purchase.
-   `buyerAddress` (String, optional): The address of the buyer. Defaults to the caller's associated wallet address from the Parse session.
-   `amount` (String, optional): The amount of NFTs to purchase (for ERC1155 or multi-edition NFTs). Defaults to 1 for ERC721.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 3. `cancelListing`

Cancel an active NFT listing.

**Function Name**: `cancelListing`
**Method**: `POST`

**Parameters**:

-   `listingId` (String, required): The unique identifier of the listing to cancel.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 4. `makeOffer`

Make an offer for an unlisted NFT or a specific listing.

**Function Name**: `makeOffer`
**Method**: `POST`

**Parameters**:

-   (Details about offer parameters such as `nftContractAddress`, `tokenId`, `offerAmount`, `expiration`, `offerorAddress`, `paymentTokenAddress`, `network`)

**Returns**: `offerId` (String), `transactionHash` (String)

## Error Handling

Marketplace tasks can encounter various errors, including:

-   **Invalid Parameters**: Incorrect listing IDs, invalid prices, or unsupported tokens.
-   **Blockchain Transaction Errors**: Transaction reverts (e.g., NFT not approved, insufficient funds, listing expired).
-   **Approval Issues**: The NFT contract has not granted allowance to the marketplace.
-   **Business Logic Violations**: Attempting to buy an already sold item, or a price mismatch.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Approvals**: Ensure your application gracefully handles `ERC721` and `ERC1155` approvals before attempting to list assets. Often, the user needs to approve the Marketplace contract to transfer their NFTs.
-   **Secure Funds Handling**: If the marketplace functions involve handling user funds (e.g., for escrow or direct transfers), verify that the smart contract logic is robust and audited.
-   **Marketplace Fees**: Be transparent about any marketplace fees and ensure they are correctly applied and distributed.
-   **Fraud Prevention**: Implement server-side validation to prevent common marketplace frauds (e.g., fake listings, price manipulations beyond smart contract logic).
-   **Spam Prevention**: Consider rate limits on listing creation to prevent marketplace spam.

## Related Documentation

-   [Smart Contracts: Marketplace Facet](../../smart-contracts/facets/marketplace-facet.md)
-   [Smart Contracts: IMultiSale Interface](../../smart-contracts/interfaces/imultisale.md)
-   [Integrator's Guide: Smart Contracts](../../integrator-guide/smart-contracts.md)
-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)