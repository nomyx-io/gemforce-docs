# Cloud Functions: Diamond Tasks

This document details the cloud functions related to managing Gemforce Diamond smart contracts. These functions provide an abstracted API for deploying, configuring, and interacting with Diamond contracts on various blockchain networks, leveraging Parse Server's backend capabilities.

## Overview

Diamond tasks enable:

-   **Deployment**: Deploying new Diamond contracts with predefined facets.
-   **Facet Management**: Adding, replacing, or removing facets from an existing Diamond.
-   **Ownership Management**: Transferring ownership of a Diamond contract.
-   **Initialization**: Running post-deployment initialization logic.

These functions streamline the deployment and management of complex Diamond contracts, abstracting away direct blockchain interactions for frontend or backend applications.

## Key Functions

### 1. `deployDiamond`

Deploy a new Gemforce Diamond contract.

**Function Name**: `deployDiamond`
**Method**: `POST` (via Parse Cloud Function endpoint)

**Parameters**:

-   `templateName` (String, required): The name of a pre-registered Diamond template to use for deployment (e.g., "StandardNFTDiamond", "DeFiProtocolDiamond").
-   `salt` (String, optional): A unique identifier (hex string) for deterministic address generation using CREATE2. If not provided, a random salt will be generated.
-   `initData` (String, optional): ABI-encoded data (hex string) for the Diamond's `init` function after deployment.
-   `ownerAddress` (String, optional): The address to set as the initial owner of the Diamond. Defaults to the deployer's address if not provided.
-   `network` (String, required): The blockchain network to deploy to (e.g., "base-sepolia", "optimism-sepolia", "sepolia").

**Returns**:

-   `diamondAddress` (String): The deployed address of the new Diamond contract.
-   `transactionHash` (String): The transaction hash of the deployment.
-   `templateUsed` (String): The name of the template used.

**Example Request**:

```json
{
    "functionName": "deployDiamond",
    "parameters": {
        "templateName": "StandardNFTDiamond",
        "salt": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
        "ownerAddress": "0xYourDesiredOwnerAddress",
        "network": "base-sepolia"
    }
}
```

**Example Response**:

```json
{
    "result": {
        "diamondAddress": "0x789...abc",
        "transactionHash": "0xdef...ghi",
        "templateUsed": "StandardNFTDiamond"
    }
}
```

**Workflow**:

1.  Client application calls `deployDiamond` Cloud Function.
2.  Cloud Function retrieves the specified Diamond template from a configuration or registry.
3.  Utilizes the `DiamondFactory` smart contract to deploy a new Diamond instance.
4.  Optionally runs initialization logic and sets ownership.
5.  Returns the new Diamond's address and transaction details.

### 2. `addFacet`

Add a new facet to an existing Diamond contract.

**Function Name**: `addFacet`
**Method**: `POST`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract to modify.
-   `facetAddress` (String, required): The address of the facet contract to add.
-   `functionSelectors` (Array of Strings, required): An array of function selectors (e.g., `"0xdeadbeef"`) to add from the facet.
-   `initContract` (String, optional): Address of an initialization contract to call after the cut.
-   `initData` (String, optional): ABI-encoded data for the `initContract` call.
-   `network` (String, required): The blockchain network where the Diamond is deployed.

**Returns**:

-   `transactionHash` (String): The transaction hash of the facet addition.

**Example Request**:

```json
{
    "functionName": "addFacet",
    "parameters": {
        "diamondAddress": "0x789...abc",
        "facetAddress": "0x123...def",
        "functionSelectors": ["0xabcd1234", "0xefgh5678"],
        "network": "base-sepolia"
    }
}
```

### 3. `replaceFacet`

Replace an existing facet's functions with new ones from a different facet contract.

**Function Name**: `replaceFacet`
**Method**: `POST`

**Parameters**: (Similar to `addFacet`, but replaces existing selectors)

-   `diamondAddress` (String, required)
-   `facetAddress` (String, required): The address of the new facet contract.
-   `functionSelectors` (Array of Strings, required): Function selectors to replace.
-   `initContract` (String, optional)
-   `initData` (String, optional)
-   `network` (String, required)

**Returns**: `transactionHash` (String)

### 4. `removeFacet`

Remove functions from a Diamond contract.

**Function Name**: `removeFacet`
**Method**: `POST`

**Parameters**: (Similar to `addFacet`, but removes selectors)

-   `diamondAddress` (String, required)
-   `functionSelectors` (Array of Strings, required): Function selectors to remove.
-   `network` (String, required)

**Returns**: `transactionHash` (String)

## Error Handling

Diamond tasks can encounter various errors, including:

-   **Invalid Parameters**: Missing required fields or incorrect data types in the request.
-   **Blockchain Transaction Errors**: Transaction reverts, out-of-gas, insufficient funds, or network congestion.
-   **Access Control**: Caller not authorized to perform the operation on the Diamond (e.g., not the owner).
-   **Template Not Found**: The specified `templateName` for `deployDiamond` does not exist.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Access Control**: Ensure that only authorized users or services can call these Cloud Functions, either through Parse's CLP/ACLs or custom logic.
-   **Master Key Usage**: Cloud Functions that interact with smart contracts typically require `useMasterKey: true` for the Parse SDK calls, as they often sign transactions on behalf of an admin address. Securely manage your Parse Master Key.
-   **Input Sanitization**: Always sanitize and validate all inputs received by Cloud Functions to prevent injection attacks or unexpected behavior.
-   **DFNS Integration**: If DFNS is used for signing, ensure the policies associated with the DFNS wallets are robust and match your security requirements.

## Related Documentation

-   [Smart Contracts: Diamond Contract](../../smart-contracts/diamond.md)
-   [Smart Contracts: Diamond Factory](../../smart-contracts/diamond-factory.md)
-   [Smart Contracts: Facets (Diamond Cut Facet)](../../smart-contracts/facets/diamond-cut-facet.md)
-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)
-   [Integrator's Guide: DFNS Integration](../../integrator-guide/dfns.md)