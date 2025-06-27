# Cloud Functions: Identity Tasks

This document details the cloud functions related to managing `IIdentity` smart contracts within the Gemforce platform. These functions provide a convenient API for creating, managing keys, and verifying claims associated with decentralized identities, abstracting away the underlying blockchain complexities.

## Overview

Identity tasks enable:

-   **Identity Creation**: Deploying new `IIdentity` contracts for users.
-   **Key Management**: Adding and removing cryptographic keys for an identity.
-   **Claim Management**: Adding, retrieving, and removing verifiable claims for an identity.
-   **Profile Management**: Updating an identity's associated off-chain profile data.
-   **Verification Flow**: Initiating and processing identity verification requests.

These functions are crucial for applications that need to interact with Gemforce's identity system without directly handling smart contract interactions.

## Key Functions

### 1. `createIdentity`

Deploy a new `IIdentity` contract for a user.

**Function Name**: `createIdentity`
**Method**: `POST`

**Parameters**:

-   `ownerAddress` (String, optional): The address that will initially control the new identity contract. If not provided, the caller's associated wallet address from the Parse session is used.
-   `profile` (Object, optional): An object containing initial off-chain profile data for the identity (e.g., `{ name: "John Doe", email: "john.doe@example.com" }`).
    -   `name` (String)
    -   `email` (String)
    -   `country` (String)
    -   `metadataURI` (String): URI to external metadata.
-   `network` (String, required): The blockchain network to deploy the identity on (e.g., "base-sepolia").

**Returns**:

-   `identityAddress` (String): The deployed address of the new `IIdentity` contract.
-   `transactionHash` (String): The transaction hash of the deployment.
-   `identityHash` (String): A unique hash representing the identity within the Gemforce registry.

**Example Request**:

```json
{
    "functionName": "createIdentity",
    "parameters": {
        "ownerAddress": "0xYourUserWalletAddress",
        "profile": {
            "name": "Alice Smith",
            "email": "alice@example.com",
            "country": "USA"
        },
        "network": "sepolia"
    }
}
```

**Example Response**:

```json
{
    "result": {
        "identityAddress": "0xabc...123",
        "transactionHash": "0xdef...456",
        "identityHash": "0xghi...789"
    }
}
```

### 2. `addKeyToIdentity`

Add a new cryptographic key to an existing identity.

**Function Name**: `addKeyToIdentity`
**Method**: `POST`

**Parameters**:

-   `identityAddress` (String, required): The address of the `IIdentity` contract.
-   `key` (String, required): The hex string representation of the public key to add (e.g., `0x<publicKeyHex>`).
-   `purpose` (Number, required): An integer representing the key's purpose (e.g., `1` for MANAGEMENT, `2` for ACTION, `3` for CLAIM, `4` for ENCRYPTION). See `IIdentity` interface for full details.
-   `keyType` (Number, required): An integer representing the key's type (e.g., `1` for ECDSA, `2` for RSA).
-   `network` (String, required): The blockchain network where the identity is deployed.

**Returns**: `transactionHash` (String)

### 3. `addClaimToIdentity`

Add a verifiable claim to an identity.

**Function Name**: `addClaimToIdentity`
**Method**: `POST`

**Parameters**:

-   `identityAddress` (String, required): The address of the `IIdentity` contract.
-   `topic` (Number, required): The topic of the claim (e.g., `1` for "KYC Approved", `2` for "Accredited Investor").
-   `scheme` (Number, required): The signature scheme used for the claim (e.g., `1` for ECDSA signature on `data`).
-   `issuerAddress` (String, required): The address of the claim issuer (must be a trusted issuer).
-   `signature` (String, required): The hex string of the cryptographic signature of the claim data.
-   `data` (String, required): The hex string of the ABI-encoded claim data.
-   `uri` (String, optional): A URI pointing to external details about the claim.
-   `network` (String, required): The blockchain network where the identity is deployed.

**Returns**: `transactionHash` (String)

## Error Handling

Identity tasks can fail due to:

-   **Invalid Parameters**: Incorrect key formats, invalid purpose/key types.
-   **Blockchain Transaction Errors**: Reverted transactions (e.g., `IIdentity: Key already exists`), out-of-gas.
-   **Access Control**: Caller not authorized to add keys or claims to the specified identity.
-   **Trusted Issuer Issues**: `issuerAddress` not recognized as a trusted issuer.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Key Exposure**: Never expose private keys of identity owners on the client-side. Utilize Cloud Functions or DFNS integration for signing on behalf of identity owners.
-   **Access Control**: Thoroughly implement access control within your Cloud Functions to ensure only authorized users can modify their identities or perform privileged operations.
-   **Trusted Issuers**: Claims are only meaningful if the `issuerAddress` is recognized as trusted. Ensure your system verifies `issuerAddress` against the `ITrustedIssuersRegistry`.
-   **Input Validation**: Sanitize and validate all inputs, especially `key`, `signature`, and `data` parameters, to prevent malicious data injection.

## Related Documentation

-   [Smart Contracts: Identity Factory](../../smart-contracts/identity-factory.md)
-   [Smart Contracts: IIdentity Interface](../../smart-contracts/interfaces/iidentity.md)
-   [Smart Contracts: ITrustedIssuersRegistry Interface](../../smart-contracts/interfaces/itrusted-issuers-registry.md)
-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)