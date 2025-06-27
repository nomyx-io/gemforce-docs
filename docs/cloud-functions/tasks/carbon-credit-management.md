# Cloud Functions: Carbon Credit Management Tasks

This document details the cloud functions designed for managing carbon credits within the Gemforce platform. These functions provide an abstracted API for interacting with on-chain carbon credit tokens, including issuance, retirement, and transfer operations, aligning with environmental asset standards.

## Overview

Carbon Credit Management tasks enable:

-   **Issuance**: Minting new carbon credit tokens.
-   **Retirement**: Burning or retiring carbon credit tokens to signify their use.
-   **Transfer**: Facilitating the transfer of carbon credit tokens between accounts.
-   **Traceability**: Querying the history and status of carbon credits.
-   **Registry Integration**: Interaction with external carbon registries or standards.

These functions are crucial for projects focused on environmental assets, allowing seamless integration with Gemforce's carbon credit framework.

## Key Functions

### 1. `issueCarbonCredits`

Issue new carbon credit tokens to a designated recipient.

**Function Name**: `issueCarbonCredits`
**Method**: `POST`

**Parameters**:

-   `recipientAddress` (String, required): The address that will receive the newly issued carbon credit tokens.
-   `amount` (String, required): The amount of carbon credit tokens to issue (as a string to handle large numbers).
-   `projectId` (String, optional): An identifier for the project associated with these carbon credits.
-   `vintageYear` (Number, optional): The vintage year of the carbon credits.
-   `metadataURI` (String, optional): URI pointing to additional metadata about the credits (e.g., certification details).
-   `network` (String, required): The blockchain network where the carbon credit contract is deployed.

**Returns**:

-   `transactionHash` (String): The transaction hash of the issuance.

**Example Request**:

```json
{
    "functionName": "issueCarbonCredits",
    "parameters": {
        "recipientAddress": "0xYourCompanyWalletAddress",
        "amount": "1000000000000000000", // 1 carbon credit token (1e18)
        "projectId": "GHG-Reduction-2023-A",
        "vintageYear": 2023,
        "network": "base-sepolia"
    }
}
```

**Example Response**:

```json
{
    "result": {
        "transactionHash": "0xabc...def"
    }
}
```

**Workflow**:

1.  Client application calls `issueCarbonCredits` Cloud Function.
2.  Cloud Function validates parameters and permissions (e.g., only authorized issuers).
3.  Calls the `issue` function on the `CarbonCreditFacet` of the Diamond contract.
4.  Returns the transaction hash.

### 2. `retireCarbonCredits`

Retire (burn) carbon credit tokens, signifying their permanent removal from circulation.

**Function Name**: `retireCarbonCredits`
**Method**: `POST`

**Parameters**:

-   `amount` (String, required): The amount of carbon credit tokens to retire.
-   `reason` (String, optional): A description of the reason for retirement.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

### 3. `transferCarbonCredits`

Transfer carbon credit tokens from one account to another.

**Function Name**: `transferCarbonCredits`
**Method**: `POST`

**Parameters**:

-   `fromAddress` (String, optional): Sender's address. Defaults to the caller's associated wallet address.
-   `toAddress` (String, required): Recipient's address.
-   `amount` (String, required): The amount of carbon credit tokens to transfer.
-   `network` (String, required): The blockchain network.

**Returns**: `transactionHash` (String)

## Error Handling

Carbon Credit Management tasks can encounter various errors, including:

-   **Invalid Parameters**: Invalid amounts, or unsupported networks.
-   **Blockchain Transaction Errors**: Transaction reverts (e.g., insufficient balance, unauthorized issuer/retirer).
-   **Access Control**: Caller not authorized to issue or retire credits.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Authorization**: Crucial that only authorized entities (e.g., certified project developers, auditors) can issue or retire carbon credits. Implement strict access control within the cloud functions.
-   **Double Counting Prevention**: Ensure that the process for issuance and retirement is robust to prevent double counting or re-use of retired credits.
-   **Token Standard Compliance**: Verify that on-chain operations comply with relevant carbon credit token standards (e.g., ERC-20, specific Carbon Credit EIPs).
-   **Input Validation**: Sanitize and validate all inputs to prevent malicious attempts or incorrect operations.

## Related Documentation

-   [Smart Contracts: Carbon Credit Facet](../../smart-contracts/facets/carbon-credit-facet.md)
-   [Smart Contracts: ICarbonCredit Interface](../../smart-contracts/interfaces/icarbon-credit.md)
-   [EIP: Carbon Credit Standard](../../eips/EIP-DRAFT-Carbon-Credit-Standard.md)
-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)