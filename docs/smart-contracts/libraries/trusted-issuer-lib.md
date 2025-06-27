# TrustedIssuerLib Library

## Overview

[`TrustedIssuerLib`](/Users/sschepis/Development/gem-base/contracts/libraries/TrustedIssuerLib.sol) is a comprehensive library for managing trusted claim issuers in the Gemforce identity system. This library implements the trusted issuer registry functionality, allowing the platform to maintain a list of authorized entities that can issue identity claims for KYC/AML compliance and other verification purposes.

## Key Features

- **Trusted Issuer Management**: Add, remove, and update trusted claim issuers
- **Claim Topic Authorization**: Control which claim topics each issuer can validate
- **Identity Verification**: Integration with ERC734/ERC735 identity standards
- **Access Control**: Owner-based permissions for issuer management
- **Efficient Lookups**: Optimized storage for fast issuer validation
- **Batch Operations**: Support for multiple claim topics per issuer

## Storage Structure

### TrustedIssuerContract

```solidity
struct TrustedIssuerContract {
    mapping(address => TrustedIssuer) trustedIssuers;
    address[] trustedIssuerAddresses;
    address owner;
}
```

### TrustedIssuerStorage

```solidity
struct TrustedIssuerStorage {
    TrustedIssuerContract trustedIssuerContract;
}
```

### Storage Position

```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nomyx.gemforce.TrustedIssuerStorage.storage");
```

## Core Functions

### Storage Access

#### `trustedIssuerStorage()`
```solidity
function trustedIssuerStorage() internal pure returns (TrustedIssuerStorage storage ds)
```
Returns the trusted issuer storage struct using assembly for gas efficiency.

### Issuer Management

#### `_addTrustedIssuer()`
```solidity
function _addTrustedIssuer(
    TrustedIssuerContract storage,
    address _trustedIssuer, 
    uint[] calldata _claimTopics
) internal
```

Adds a new trusted issuer with specified claim topics.

**Parameters:**
- `_trustedIssuer`: Address of the claim issuer contract
- `_claimTopics`: Array of claim topic IDs the issuer can validate

**Features:**
- **Duplicate Prevention**: Only adds issuer if not already present
- **Topic Assignment**: Associates specific claim topics with the issuer
- **Address Tracking**: Maintains array of issuer addresses for enumeration

#### `removeTrustedIssuer()`
```solidity
function removeTrustedIssuer(TrustedIssuerContract storage self, address _trustedIssuer) internal
```

Removes a trusted issuer from the registry.

**Parameters:**
- `_trustedIssuer`: Address of the issuer to remove

**Note:** This function only removes the issuer mapping but doesn't clean up the address array for gas efficiency.

#### `updateIssuerClaimTopics()`
```solidity
function updateIssuerClaimTopics(
    TrustedIssuerContract storage self, 
    address _trustedIssuer, 
    uint[] calldata _claimTopics
) internal
```

Updates the claim topics that a trusted issuer is authorized to validate.

**Parameters:**
- `_trustedIssuer`: Address of the issuer to update
- `_claimTopics`: New array of authorized claim topics

### Issuer Queries

#### `_getTrustedIssuer()`
```solidity
function _getTrustedIssuer(
    TrustedIssuerContract storage,
    address issuerAddress
) internal view returns (TrustedIssuer memory trustedIssuer)
```

Retrieves the trusted issuer struct for a given address.

#### `getTrustedIssuers()`
```solidity
function getTrustedIssuers(TrustedIssuerContract storage self) internal view returns (TrustedIssuer[] memory trustedIssuers)
```

Returns an array of all trusted issuers in the registry.

**Returns:**
- Array of `TrustedIssuer` structs containing issuer details

#### `isTrustedIssuer()`
```solidity
function isTrustedIssuer(address _issuer) internal view returns(bool isTrusted)
```

Checks if an address is a trusted issuer.

**Parameters:**
- `_issuer`: Address to check

**Returns:**
- `true` if the address is a trusted issuer, `false` otherwise

### Claim Topic Management

#### `getTrustedIssuerClaimTopics()`
```solidity
function getTrustedIssuerClaimTopics(address _trustedIssuer) external view returns(uint[] memory)
```

Returns the claim topics that a trusted issuer is authorized to validate.

**Parameters:**
- `_trustedIssuer`: Address of the trusted issuer

**Returns:**
- Array of claim topic IDs

#### `hasTrustedIssuerClaimTopic()`
```solidity
function hasTrustedIssuerClaimTopic(address _issuer, uint _claimTopic) external view returns(bool hasTopic)
```

Checks if a trusted issuer is authorized for a specific claim topic.

**Parameters:**
- `_issuer`: Address of the issuer
- `_claimTopic`: Claim topic ID to check

**Returns:**
- `true` if the issuer is authorized for the topic, `false` otherwise

### Internal Helper Functions

#### `_setTrustedIssuer()`
```solidity
function _setTrustedIssuer(
    TrustedIssuerContract storage self,
    address issuerAddress,
    TrustedIssuer memory trustedIssuer
) internal
```

Internal function to set or update a trusted issuer.

**Features:**
- **Address Tracking**: Adds to address array if new issuer
- **Mapping Update**: Updates the issuer mapping
- **Efficient Storage**: Avoids duplicate address entries

## Data Structures

### TrustedIssuer

```solidity
struct TrustedIssuer {
    address claimIssuer;
    uint[] claimTopics;
}
```

**Fields:**
- `claimIssuer`: Address of the claim issuer contract
- `claimTopics`: Array of claim topic IDs the issuer can validate

## Usage Examples

### Adding a Trusted Issuer

```solidity
function addKYCIssuer(address issuerAddress) external onlyOwner {
    uint[] memory kycTopics = new uint[](2);
    kycTopics[0] = 1; // KYC topic
    kycTopics[1] = 2; // AML topic
    
    TrustedIssuerLib._addTrustedIssuer(
        trustedIssuerStorage().trustedIssuerContract,
        issuerAddress,
        kycTopics
    );
}
```

### Validating an Issuer

```solidity
function validateClaim(address issuer, uint claimTopic) external view returns (bool) {
    return TrustedIssuerLib.isTrustedIssuer(issuer) && 
           TrustedIssuerLib.hasTrustedIssuerClaimTopic(issuer, claimTopic);
}
```

### Getting All Trusted Issuers

```solidity
function getAllTrustedIssuers() external view returns (TrustedIssuer[] memory) {
    return TrustedIssuerLib.getTrustedIssuers(
        trustedIssuerStorage().trustedIssuerContract
    );
}
```

### Updating Issuer Permissions

```solidity
function updateIssuerTopics(address issuer, uint[] calldata newTopics) external onlyOwner {
    TrustedIssuerLib.updateIssuerClaimTopics(
        trustedIssuerStorage().trustedIssuerContract,
        issuer,
        newTopics
    );
}
```

## Integration with Identity System

### ERC734/ERC735 Integration

```solidity
// Validate a claim during identity verification
function validateIdentityClaim(
    address identity,
    uint claimTopic,
    address issuer
) external view returns (bool) {
    // Check if issuer is trusted
    if (!TrustedIssuerLib.isTrustedIssuer(issuer)) {
        return false;
    }
    
    // Check if issuer can validate this topic
    if (!TrustedIssuerLib.hasTrustedIssuerClaimTopic(issuer, claimTopic)) {
        return false;
    }
    
    // Additional validation logic...
    return true;
}
```

### Claim Topic Constants

Common claim topics used in the system:

```solidity
uint256 constant KYC_TOPIC = 1;
uint256 constant AML_TOPIC = 2;
uint256 constant ACCREDITED_INVESTOR_TOPIC = 3;
uint256 constant RESIDENCE_TOPIC = 4;
uint256 constant SANCTIONS_TOPIC = 5;
```

## Security Features

### Access Control
- **Owner-Only Operations**: Critical functions require owner permissions
- **Issuer Validation**: Comprehensive validation before adding issuers
- **Topic Authorization**: Granular control over claim topic permissions

### Validation Mechanisms
- **Address Verification**: Ensures issuer addresses are valid contracts
- **Topic Consistency**: Validates claim topic assignments
- **State Integrity**: Maintains consistent storage state

### Gas Optimization
- **Efficient Lookups**: O(1) issuer validation using mappings
- **Batch Operations**: Support for multiple topics in single transaction
- **Storage Patterns**: Optimized storage layout for minimal gas costs

## Common Claim Topics

### Identity Verification
- **KYC (Know Your Customer)**: Topic ID 1
- **AML (Anti-Money Laundering)**: Topic ID 2
- **Sanctions Screening**: Topic ID 5

### Financial Compliance
- **Accredited Investor**: Topic ID 3
- **Residence Verification**: Topic ID 4
- **Tax Status**: Topic ID 6

### Platform-Specific
- **Whitelist Status**: Topic ID 100
- **VIP Status**: Topic ID 101
- **Trading Permissions**: Topic ID 102

## Best Practices

### Issuer Management
1. **Thorough Vetting**: Carefully vet all trusted issuers before adding
2. **Regular Audits**: Periodically review and update issuer list
3. **Topic Specificity**: Assign only necessary claim topics to each issuer
4. **Documentation**: Maintain clear documentation of issuer purposes

### Security Considerations
1. **Multi-Sig Control**: Use multi-signature wallets for owner operations
2. **Gradual Rollout**: Add new issuers gradually and monitor performance
3. **Emergency Procedures**: Have procedures for quickly removing compromised issuers
4. **Backup Systems**: Maintain backup issuer systems for critical topics

### Gas Optimization
1. **Batch Updates**: Group multiple issuer updates in single transactions
2. **Efficient Queries**: Use view functions for validation checks
3. **Storage Cleanup**: Periodically clean up removed issuer addresses
4. **Topic Grouping**: Group related claim topics for efficiency

## Error Handling

### Common Validation Checks

```solidity
modifier onlyTrustedIssuer(address issuer, uint claimTopic) {
    require(
        TrustedIssuerLib.isTrustedIssuer(issuer),
        "TrustedIssuerLib: Issuer not trusted"
    );
    require(
        TrustedIssuerLib.hasTrustedIssuerClaimTopic(issuer, claimTopic),
        "TrustedIssuerLib: Issuer not authorized for topic"
    );
    _;
}
```

### Event Logging

```solidity
event TrustedIssuerAdded(address indexed issuer, uint[] claimTopics);
event TrustedIssuerRemoved(address indexed issuer);
event IssuerTopicsUpdated(address indexed issuer, uint[] newTopics);
```

## Related Documentation

- [Identity Registry Facet](../facets/identity-registry-facet.md) - Identity management interface
- [Trusted Issuers Registry Facet](../facets/trusted-issuers-registry-facet.md) - Registry management interface
- [ERC734 Interface](../interfaces/ierc734.md) - Key Manager interface
- [ERC735 Interface](../interfaces/ierc735.md) - Claim Holder interface
- [Enhanced Identity System EIP](../../eips/EIP-DRAFT-Enhanced-Identity-System.md) - Identity system specification

## Migration and Upgrades

### Storage Compatibility
- Maintains backward compatibility with existing issuer data
- Supports incremental updates to claim topic assignments
- Provides migration paths for issuer address changes

### Upgrade Considerations
- Storage position remains constant across upgrades
- New claim topics can be added without affecting existing issuers
- Issuer removal requires careful consideration of dependent systems