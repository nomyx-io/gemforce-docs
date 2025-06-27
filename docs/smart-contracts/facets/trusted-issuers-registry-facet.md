# TrustedIssuersRegistryFacet

The `TrustedIssuersRegistryFacet` is a vital administrative component within the Gemforce Diamond smart contract system, responsible for managing a whitelist of trusted entities capable of issuing claims or attestations within the platform's identity and credentialing infrastructure.

## Purpose

The primary purpose of the `TrustedIssuersRegistryFacet` is to establish and maintain a high-trust environment for on-chain identity and data verification. By restricting claim issuance capabilities to a curated list of trusted parties, the facet helps prevent spam, malicious attestations, and ensures the integrity and reliability of credentials issued within the Gemforce ecosystem. This is critical for applications that rely on verifiable claims, such as KYC/AML processes, reputation systems, or verifiable credentials for real-world assets.

## Key Features

*   **Whitelisted Issuers**: Maintains a list of addresses authorized to issue claims or attestations.
*   **Permissioned Control**: Only authorized administrators (Diamond owner) can add or remove trusted issuers.
*   **Queryable Status**: Allows any entity to check if a given address is a trusted issuer.
*   **Event Emission**: Emits events for every addition and removal of a trusted issuer, providing transparency and auditability.

## Functions

### `addTrustedIssuer(address _issuer)`

Adds an address to the list of trusted claim issuers.

*   **Parameters**:
    *   `_issuer` (address): The address to be added as a trusted issuer.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_issuer` must not already be in the trusted list.
    *   `_issuer` must not be the zero address.
*   **Emits**: `TrustedIssuerAdded(address indexed issuer)`

### `removeTrustedIssuer(address _issuer)`

Removes an address from the list of trusted claim issuers.

*   **Parameters**:
    *   `_issuer` (address): The address to be removed from the trusted list.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_issuer` must be present in the trusted list.
*   **Emits**: `TrustedIssuerRemoved(address indexed issuer)`

### `isTrustedIssuer(address _issuer)`

Checks if a given address is currently a trusted claim issuer.

*   **Parameters**:
    *   `_issuer` (address): The address to check.
*   **Returns**: (bool) `true` if the address is a trusted issuer, `false` otherwise.

## Events

### `TrustedIssuerAdded(address indexed issuer)`

Emitted when an address is successfully added to the trusted issuers list.

*   **Parameters**:
    *   `issuer` (address): The address that was added.

### `TrustedIssuerRemoved(address indexed issuer)`

Emitted when an address is successfully removed from the trusted issuers list.

*   **Parameters**:
    *   `issuer` (address): The address that was removed.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Example: Add a new trusted issuer
address newTrustedOrgAddress = 0xSomeOrganizationAddress;
IDiamond(diamond).addTrustedIssuer(newTrustedOrgAddress);

// Example: Check if an address is a trusted issuer
bool isOrgTrusted = IDiamond(diamond).isTrustedIssuer(newTrustedOrgAddress);

// Example: Remove a trusted issuer
IDiamond(diamond).removeTrustedIssuer(newTrustedOrgAddress);