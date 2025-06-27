# IdentityRegistryFacet

The `IdentityRegistryFacet` is a key component of the Gemforce Diamond smart contract system, responsible for managing decentralized identities within the platform. It enables the registration, verification, and revocation of identities, leveraging the ERC-735 standard for claim management and ERC-734 for key management.

## Purpose

The primary purpose of the `IdentityRegistryFacet` is to provide a robust and secure framework for self-sovereign identities. It allows users to control their own identity data, issue claims about themselves or others, and manage the keys associated with their identity. This facet is crucial for building trust, enabling compliant operations, and fostering a verifiable ecosystem on the Gemforce platform.

## Key Features

*   **Identity Registration**: Allows users or authorized entities to register new decentralized identities.
*   **Key Management (ERC-734)**: Supports adding, removing, and managing keys associated with an identity, enabling different levels of access and control.
*   **Claim Management (ERC-735)**: Facilitates the issuance, adding, removal, and getting of claims about an identity, issued by various claim issuers.
*   **Identity Status**: Enables setting and checking the status of an identity (e.g., active, revoked).
*   **Proxy Functionality**: Allows an identity owner to execute transactions through their identity contract.
*   **Event Emission**: Emits events for every significant identity-related action, providing transparency and traceability.

## Functions

### `addIdentity(address _identity)`

Registers a new identity with the registry. This function typically sets up a new ERC-735 identity contract.

*   **Parameters**:
    *   `_identity` (address): The address of the identity contract to register.
*   **Requirements**:
    *   `_identity` must not already be registered.
*   **Emits**: `IdentityAdded(address indexed identity)`

### `removeIdentity(address _identity)`

Removes an identity from the registry. This may also trigger the deactivation or revocation of the associated identity contract.

*   **Parameters**:
    *   `_identity` (address): The address of the identity contract to remove.
*   **Requirements**:
    *   Only an authorized entity (e.g., registry owner) can call this function.
    *   `_identity` must be a registered identity.
*   **Emits**: `IdentityRemoved(address indexed identity)`

### `getIdentity(address _identity)`

Retrieves information about a registered identity, such as its associated keys and claims.

*   **Parameters**:
    *   `_identity` (address): The address of the identity contract.
*   **Returns**: (address) The identity's contract address. Additional information can be queried directly from the ERC-735 identity contract.
*   **Requirements**:
    *   `_identity` must be a registered identity.

### `isIdentity(address _identity)`

Checks if a given address is a registered identity.

*   **Parameters**:
    *   `_identity` (address): The address to check.
*   **Returns**: (bool) `true` if it's a registered identity, `false` otherwise.

### ERC-735 Claim-Related Functions (via Identity Contract Proxy)

The following functions are generally called on the specific identity contract itself, but the `IdentityRegistryFacet` facilitates interaction or acts as a proxy for management by an authorized entity.

#### `addClaim(uint256 _topic, uint256 _scheme, address _issuer, bytes _signature, bytes _data, string _uri)`

Adds a claim to the identity.

*   **Parameters**: Standard ERC-735 claim parameters.
*   **Emits**: `ClaimAdded(bytes32 indexed claimId, uint256 indexed topic, uint256 indexed scheme, address indexed issuer, bytes signature, bytes data, string uri)`

#### `removeClaim(bytes32 _claimId)`

Removes a claim from the identity.

*   **Parameters**:
    *   `_claimId` (bytes32): The ID of the claim to remove.
*   **Emits**: `ClaimRemoved(bytes32 indexed claimId, uint256 indexed topic, uint256 indexed scheme, address indexed issuer)`

#### `getClaim(bytes32 _claimId)`

Retrieves a claim from the identity.

*   **Parameters**:
    *   `_claimId` (bytes32): The ID of the claim.
*   **Returns**: Claim details.

### ERC-734 Key-Related Functions (via Identity Contract Proxy)

#### `addKey(bytes32 _key, uint256 _purpose, uint256 _keyType)`

Adds a key to the identity.

*   **Parameters**: Standard ERC-734 key parameters.
*   **Emits**: `KeyAdded(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType)`

#### `removeKey(bytes32 _key, uint256 _purpose)`

Removes a key from the identity.

*   **Parameters**:
    *   `_key` (bytes32): The key to remove.
    *   `_purpose` (uint256): The purpose of the key.
*   **Emits**: `KeyRemoved(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType)`

## Events

### `IdentityAdded(address indexed identity)`

Emitted when a new identity is successfully registered.

*   **Parameters**:
    *   `identity` (address): The address of the newly registered identity contract.

### `IdentityRemoved(address indexed identity)`

Emitted when an identity is removed from the registry.

*   **Parameters**:
    *   `identity` (address): The address of the identity contract that was removed.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Register a new identity contract
address newIdentityAddress = 0xSomeNewIdentityContractAddress;
IDiamond(diamond).addIdentity(newIdentityAddress);

// Assuming the identity contract has self-management or is managed by a trusted issuer
// Adding a key to the new identity (example call on the identity contract directly)
// IIdentity(newIdentityAddress).addKey(keyHash, 1, 1); // Key for management purpose

// Add a claim to the new identity (example call on the identity contract directly)
// IIdentity(newIdentityAddress).addClaim(topic, scheme, issuerAddress, signature, data, uri);

// Check if an address is an identity
bool isId = IDiamond(diamond).isIdentity(newIdentityAddress);