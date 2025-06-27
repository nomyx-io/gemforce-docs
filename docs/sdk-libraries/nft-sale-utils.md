# SDK & Libraries: NFT Sale Utilities

This document describes the `nft-sale-utils.ts` library, a utility module within the Gemforce SDK designed to simplify interactions related to NFT sales, especially those leveraging the `IMultiSale` interface or `MultiSaleFacet`. It provides helper functions for common tasks such as preparing sale parameters, interacting with payment tokens, and managing NFT transfers within a sale context.

## Overview

The `nft-sale-utils.ts` library offers:

-   **Sale Parameter Preparation**: Helpers to format and validate data for NFT sale contracts.
-   **Token Operations**: Simplifies interactions with ERC-20 and ERC-721/1155 tokens relevant to sales (e.g., approvals, balance checks).
-   **Price Calculation**: Integrates with variable pricing mechanisms (if used) to determine current NFT prices.
-   **Sale Status Query**: Functions to query the status of active sales.

This library aims to reduce the complexity of developing applications that participate in or manage NFT sales on the Gemforce platform.

## Key Functions

### 1. `prepareSaleConfig(config: SaleConfigDraft)`

Prepares a `SaleConfig` object for creating a new NFT sale, converting amounts to appropriate formats (e.g., Wei) and performing basic validations.

**Function Signature**:
`prepareSaleConfig(config: SaleConfigDraft): IMultiSale.SaleConfigStruct`

**Parameters**:

-   `config` (SaleConfigDraft, required): A draft configuration object containing sale details.

**Returns**:
-   `IMultiSale.SaleConfigStruct`: A fully prepared `SaleConfigStruct` ready to be sent to an `IMultiSale` contract or `MultiSaleFacet`.

**Example Usage**:

```typescript
import { prepareSaleConfig } from '@gemforce-sdk/nft-sale-utils'; // Assuming this import path
import { ethers } from 'ethers';
import { IMultiSale } from '@gemforce/interfaces'; // Import IMultiSale for type definition

interface SaleConfigDraft {
    sellerAddress: string;
    nftContractAddress: string;
    tokenId?: number; // For ERC721 specific listings, leave undefined for general sales
    tokenType: 'ERC721' | 'ERC1155' | 'ERC20';
    totalAmount?: number; // Total number of items for sale (for ERC721 total supply, or ERC1155/ERC20 amount)
    pricePerUnit: string; // As a decimal string (e.g., "0.1", "100")
    paymentTokenSymbol: string; // e.g., "ETH", "USDC"
    durationInDays: number;
    maxPurchasePerAddress?: number;
    isWhitelistedSale?: boolean;
    initialReceiverAddress?: string;
    feePercentage?: number; // In basis points, e.g., 500 for 5%
    name: string;
    description: string;
}

async function createNewNFTSale(saleDraft: SaleConfigDraft) {
    try {
        const preparedConfig = prepareSaleConfig(saleDraft);
        
        // Example: Now send this preparedConfig to your MultiSaleFacet via a Cloud Function
        // or directly to the smart contract.
        // await callCloudFunction("createNFTSale", { preparedConfig, network: "sepolia" });
        console.log("Prepared Sale Config:", preparedConfig);
        return preparedConfig;
    } catch (error) {
        console.error("Error preparing sale config:", error);
        throw error;
    }
}

// Example:
// createNewNFTSale({
//     sellerAddress: "0x...",
//     nftContractAddress: "0x...",
//     tokenType: "ERC721",
//     pricePerUnit: "0.05",
//     paymentTokenSymbol: "ETH",
//     durationInDays: 7,
//     name: "Limited Edition NFT",
//     description: "A cool new NFT collection."
// });
```

### 2. `approveNFTForSale(nftContractAddress: string, tokenId: string, spenderAddress: string, network: string, signer: ethers.Signer)`

Ensures that the `IMultiSale` contract (or `MultiSaleFacet`) has the necessary approval to transfer a user's NFT.

**Function Signature**:
`approveNFTForSale(nftContractAddress: string, tokenId: string, spenderAddress: string, network: string, signer: ethers.Signer): Promise<ethers.providers.TransactionReceipt>`

**Parameters**:

-   `nftContractAddress` (String, required): The address of the NFT contract (ERC721 or ERC1155).
-   `tokenId` (String, required): The ID of the specific NFT. (For ERC1155, this might be a type ID).
-   `spenderAddress` (String, required): The address of the marketplace or sale contract that needs approval.
-   `network` (String, required): The blockchain network.
-   `signer` (ethers.Signer, required): The `ethers.Signer` of the NFT owner.

**Returns**:
-   `Promise<ethers.providers.TransactionReceipt>`: A Promise that resolves to the transaction receipt for the approval.

### 3. `getNFTCurrentPrice(saleId: string, network: string, quantity?: number)`

Retrieves the current price of an NFT from a sale, considering potential dynamic pricing mechanisms.

**Function Signature**:
`getNFTCurrentPrice(saleId: string, network: string, quantity?: number): Promise<ethers.BigNumber>`

**Parameters**:

-   `saleId` (String, required): The unique ID of the sale.
-   `network` (String, required): The blockchain network.
-   `quantity` (Number, optional): The quantity for which to calculate the price (defaults to 1).

**Returns**:
-   `Promise<ethers.BigNumber>`: A Promise that resolves to the current price in Wei or base units of the payment token.

### 4. `buyNFT(saleId: string, quantity: number, network: string, signer: ethers.Signer, value?: ethers.BigNumberish)`

Facilitates the purchase of NFTs from a sale. This function handles sending the transaction to the appropriate marketplace contract.

**Function Signature**:
`buyNFT(saleId: string, quantity: number, network: string, signer: ethers.Signer, value?: ethers.BigNumberish): Promise<ethers.providers.TransactionReceipt>`

**Parameters**:

-   `saleId` (String, required): The ID of the sale from which to buy.
-   `quantity` (Number, required): The number of NFTs to purchase.
-   `network` (String, required): The blockchain network.
-   `signer` (ethers.Signer, required): The `ethers.Signer` of the buyer.
-   `value` (ethers.BigNumberish, optional): The amount of native currency (ETH) to send with the transaction, if the payment token is ETH.

**Returns**:
-   `Promise<ethers.providers.TransactionReceipt>`: A Promise that resolves to the transaction receipt.

## Error Handling

NFT Sale Utilities can encounter various errors:

-   **Approval Failures**: If the necessary token approvals are missing.
-   **Insufficient Funds/Tokens**: Buyer or seller lacks balance.
-   **Sale Logic Errors**: Issues from the underlying `IMultiSale` contract (e.g., sale ended, invalid quantity).
-   **Network/Transaction Issues**: Standard blockchain errors.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Token Approvals**: Always guide users through the approval process securely. Ensure approvals are granted to the correct, audited marketplace contract address.
-   **Price Sanity Checks**: On the client-side, perform sanity checks on the `getNFTCurrentPrice` return value to prevent unexpected high costs.
-   **Phishing/Scam Prevention**: Educate users about verifying contract addresses and ensuring they are interacting with the genuine Gemforce marketplace.
-   **Private Key Management**: All transaction-sending functions (`approveNFTForSale`, `buyNFT`) rely on a `signer`. Ensure private keys backing this signer are managed with utmost security.

## Related Documentation

-   [Smart Contracts: IMultiSale Interface](../smart-contracts/interfaces/imultisale.md)
-   [Smart Contracts: Multi Sale Facet](../smart-contracts/facets/multi-sale-facet.md)
-   [SDK & Libraries: Blockchain Utilities](blockchain.md)
-   [SDK & Libraries: Contract Utilities](contract.md)