# SDK & Libraries: Diamond Utilities

This document describes the `diamond.ts` library, a crucial utility module within the Gemforce SDK specifically designed to facilitate interactions with the Gemforce Diamond smart contract architecture. It provides functions to query Diamond structure, interact with its facets, and manage various aspects of a Diamond deployment programmatically.

## Overview

The `diamond.ts` library offers:

-   **Diamond Introspection**: Functions to discover facets and their associated function selectors.
-   **Facet Interaction**: Simplified methods to call functions on specific facets through the Diamond.
-   **Helper Utilities**: Tools for common Diamond-related operations, like parsing Diamond ABI.

This library aims to abstract the complexities of the Diamond Standard for developers, allowing them to focus on business logic rather than the intricate details of Diamond proxy patterns.

## Key Functions

### 1. `getFacetAddresses(diamondAddress: string, network: string)`

Retrieves a list of all facet addresses currently connected to a given Diamond contract.

**Function Signature**:
`getFacetAddresses(diamondAddress: string, network: string): Promise<string[]>`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract.
-   `network` (String, required): The blockchain network where the Diamond is deployed.

**Returns**:
-   `Promise<string[]>`: A Promise that resolves to an array of facet addresses (hex strings).

**Example Usage**:

```typescript
import { getFacetAddresses } from '@gemforce-sdk/diamond'; // Assuming this import path

async function listAllFacets(diamondAddr: string, network: string) {
    try {
        const addresses = await getFacetAddresses(diamondAddr, network);
        console.log(`Facets for Diamond ${diamondAddr} on ${network}:`, addresses);
        return addresses;
    } catch (error) {
        console.error(`Error getting facet addresses for ${diamondAddr}:`, error);
        throw error;
    }
}

// Example: listAllFacets("0xYourDiamondAddress", "sepolia");
```

### 2. `getFacetFunctionSelectors(diamondAddress: string, facetAddress: string, network: string)`

Retrieves all function selectors associated with a specific facet address within a Diamond.

**Function Signature**:
`getFacetFunctionSelectors(diamondAddress: string, facetAddress: string, network: string): Promise<string[]>`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract.
-   `facetAddress` (String, required): The address of the facet to query.
-   `network` (String, required): The blockchain network.

**Returns**:
-   `Promise<string[]>`: A Promise that resolves to an array of function selectors (hex strings).

### 3. `getDiamondCut(diamondAddress: string, network: string)`

Returns the `DiamondCut` information, including which facets are associated with which selectors.

**Function Signature**:
`getDiamondCut(diamondAddress: string, network: string): Promise<FacetCut[]>`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract.
-   `network` (String, required): The blockchain network.

**Returns**:
-   `Promise<FacetCut[]>`: A Promise that resolves to an array of `FacetCut` objects, each detailing a facet's functions.

### 4. `getDiamondABI(diamondAddress: string, network: string, options?: { includeLibraries?: boolean })`

Constructs a consolidated ABI for a Diamond contract by combining the ABIs of all its currently attached facets.

**Function Signature**:
`getDiamondABI(diamondAddress: string, network: string, options?: { includeLibraries?: boolean }): Promise<any[]>`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract.
-   `network` (String, required): The blockchain network.
-   `options` (Object, optional): Configuration for building the ABI.
    -   `includeLibraries` (Boolean): Whether to include ABIs of linked libraries if available.

**Returns**:
-   `Promise<any[]>`: A Promise that resolves to a combined ABI array suitable for `ethers.Contract` instantiation.

**Example Usage**:

```typescript
import { getDiamondABI } from '@gemforce-sdk/diamond';
import { ethers } from 'ethers';

async function interactWithDiamond(diamondAddr: string, network: string) {
    try {
        const provider = new ethers.providers.JsonRpcProvider("YOUR_RPC_URL");
        const signer = new ethers.Wallet("YOUR_PRIVATE_KEY", provider); // Replace securely

        // Get the combined ABI for the Diamond
        const combinedAbi = await getDiamondABI(diamondAddr, network);
        
        // Instantiate the Diamond contract using the combined ABI
        const diamondContract = new ethers.Contract(diamondAddr, combinedAbi, signer);

        // Now you can call any function from any attached facet directly through 'diamondContract'
        const owner = await diamondContract.owner(); // Assuming an OwnershipFacet is attached
        console.log(`Diamond owner: ${owner}`);

        // Example: Call a marketplace function (assuming MarketplaceFacet is attached)
        // const listings = await diamondContract.getAllListings();
        // console.log("Current listings:", listings);

        return diamondContract;
    } catch (error) {
        console.error(`Error interacting with Diamond ${diamondAddr}:`, error);
        throw error;
    }
}

// Example: interactWithDiamond("0xYourDiamondAddress", "sepolia");
```

## Error Handling

Errors from the Diamond utilities library typically include:

-   **Network Issues**: Problems connecting to RPC providers.
-   **Invalid Addresses**: Non-existent Diamond or facet addresses.
-   **ABI Fetching Failures**: Unable to retrieve ABIs for facets.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **ABI Source**: Ensure that the ABIs used for `getDiamondABI` are retrieved from a trusted source to prevent malicious function signature manipulation.
-   **Network Trust**: All interactions rely on the trustworthiness of the chosen blockchain network and RPC provider.
-   **Private Key Management**: While this library doesn't manage private keys directly, the usage examples demonstrate that any transaction-sending operations require a securely managed `Signer`.

## Related Documentation

-   [Smart Contracts: Diamond Contract](../smart-contracts/diamond.md)
-   [Smart Contracts: Diamond Loupe Facet](../smart-contracts/facets/diamond-loupe-facet.md)
-   [Integrator's Guide: Smart Contracts](../integrator-guide/smart-contracts.md)
-   [SDK & Libraries: Blockchain Utilities](blockchain.md)