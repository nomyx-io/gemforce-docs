# EIP-DRAFT: Enhanced Identity System with Trusted Issuers and Attribute Management

## Simple Summary

An enhanced identity standard that extends ERC734/ERC735 with trusted issuer management, attribute-based access control, and integration with decentralized applications for compliance and verification purposes.

## Abstract

This EIP proposes an enhanced identity system that builds upon the ERC734 (Key Management) and ERC735 (Claim Holder) standards to provide:
- Trusted issuer registry with claim topic authorization
- Attribute-based identity management with typed attributes
- Integration with smart contract systems for automated verification
- Claim topic-based access control for decentralized applications
- Verification status tracking and management

## Motivation

While ERC734 and ERC735 provide basic identity functionality, they lack standardized mechanisms for trusted issuer management and attribute handling. This enhanced system addresses these gaps by providing a comprehensive identity framework suitable for regulatory compliance and sophisticated access control in DeFi and other applications.

## Specification

### Core Interface

```solidity
interface IEnhancedIdentity is IERC734, IERC735 {
    enum AttributeType { STRING, NUMBER, BOOLEAN, ADDRESS, BYTES }
    
    struct Attribute {
        string key;
        AttributeType attributeType;
        string value;
    }
    
    // Events
    event AttributeSet(string key, AttributeType attributeType, string value);
    event TrustedIssuerAdded(address indexed issuer, uint256[] claimTopics);
    event TrustedIssuerRemoved(address indexed issuer);
    event ClaimTopicAuthorized(address indexed issuer, uint256 indexed claimTopic);
    event ClaimTopicRevoked(address indexed issuer, uint256 indexed claimTopic);
    
    // Attribute Management
    function getAttribute(string memory key) external view returns (Attribute memory);
    function setAttribute(string memory key, AttributeType attributeType, string memory value) external;
    function getAttributesByType(AttributeType attributeType) external view returns (string[] memory keys);
    
    // Verification Status
    function isVerified() external view returns (bool);
    function getClaimTopics() external view returns (uint256[] memory);
    function hasClaimTopic(uint256 claimTopic) external view returns (bool);
    
    // Trusted Issuer Integration
    function isTrustedIssuer(address issuer) external view returns (bool);
    function hasTrustedIssuerClaimTopic(address issuer, uint256 claimTopic) external view returns (bool);
}

interface ITrustedIssuerRegistry {
    struct TrustedIssuer {
        address issuer;
        uint256[] claimTopics;
        bool active;
    }
    
    // Events
    event TrustedIssuerAdded(address indexed issuer, uint256[] claimTopics);
    event TrustedIssuerRemoved(address indexed issuer);
    event TrustedIssuerClaimTopicsUpdated(address indexed issuer, uint256[] claimTopics);
    
    // Core Functions
    function addTrustedIssuer(address issuer, uint256[] memory claimTopics) external;
    function removeTrustedIssuer(address issuer) external;
    function updateTrustedIssuerClaimTopics(address issuer, uint256[] memory claimTopics) external;
    function isTrustedIssuer(address issuer) external view returns (bool);
    function getTrustedIssuerClaimTopics(address issuer) external view returns (uint256[] memory);
    function hasTrustedIssuerClaimTopic(address issuer, uint256 claimTopic) external view returns (bool);
    function getAllTrustedIssuers() external view returns (address[] memory);
}

interface IIdentityRegistry {
    // Events
    event IdentityRegistered(address indexed wallet, address indexed identity);
    event IdentityUpdated(address indexed wallet, address indexed identity);
    event IdentityRemoved(address indexed wallet);
    
    // Core Functions
    function registerIdentity(address wallet, address identity) external;
    function updateIdentity(address wallet, address identity) external;
    function removeIdentity(address wallet) external;
    function getIdentity(address wallet) external view returns (address);
    function isRegistered(address wallet) external view returns (bool);
    function hasValidClaim(address wallet, uint256 claimTopic) external view returns (bool);
}
```

### Key Features

#### 1. Enhanced Attribute System
- **Typed Attributes**: Support for different data types (string, number, boolean, address, bytes)
- **Structured Storage**: Organized attribute management with type-based queries
- **Trusted Issuer Control**: Only trusted issuers can set attributes

#### 2. Trusted Issuer Management
- **Claim Topic Authorization**: Issuers are authorized for specific claim topics
- **Centralized Registry**: Shared trusted issuer registry across applications
- **Dynamic Management**: Add, remove, and update issuer permissions

#### 3. Verification Integration
- **Automated Verification**: Smart contracts can verify identity claims
- **Claim Topic Validation**: Check for specific required claims
- **Status Tracking**: Track overall verification status

#### 4. Access Control Integration
- **Claim-Based Access**: Use identity claims for application access control
- **Flexible Requirements**: Configure different claim requirements per application
- **Real-time Validation**: Validate claims during transaction execution

### Claim Topics

Standard claim topics for common use cases:

```solidity
library ClaimTopics {
    uint256 public constant KYC_VERIFIED = 1;
    uint256 public constant AML_CLEARED = 2;
    uint256 public constant ACCREDITED_INVESTOR = 3;
    uint256 public constant COUNTRY_VERIFICATION = 4;
    uint256 public constant AGE_VERIFICATION = 5;
    uint256 public constant PROFESSIONAL_VERIFICATION = 6;
    uint256 public constant INSTITUTION_VERIFICATION = 7;
    uint256 public constant SANCTIONS_CLEARED = 8;
}
```

### Integration Pattern

```solidity
contract DeFiApplication {
    IIdentityRegistry public identityRegistry;
    
    modifier requiresClaim(uint256 claimTopic) {
        require(identityRegistry.hasValidClaim(msg.sender, claimTopic), "Required claim not found");
        _;
    }
    
    function restrictedFunction() external requiresClaim(ClaimTopics.KYC_VERIFIED) {
        // Function logic here
    }
}
```

## Rationale

### Trusted Issuer System
A centralized trusted issuer registry enables consistent validation across applications while maintaining decentralization of the identity system itself.

### Typed Attributes
Supporting different attribute types enables more sophisticated identity data management and reduces parsing overhead in applications.

### Claim Topic Authorization
Restricting issuers to specific claim topics prevents unauthorized claim issuance and maintains the integrity of the verification system.

### Registry Pattern
Using a registry pattern for identities enables wallet-to-identity mapping while keeping identity contracts upgradeable and transferable.

## Implementation Details

### Attribute Storage

```solidity
mapping(string => Attribute) private attributes;
mapping(uint256 => string[]) private attributeKeys; // AttributeType => keys

function setAttribute(string memory key, AttributeType attributeType, string memory value) external onlyTrustedIssuer {
    attributes[key] = Attribute(key, attributeType, value);
    attributeKeys[uint256(attributeType)].push(key);
    emit AttributeSet(key, attributeType, value);
}
```

### Claim Validation

```solidity
function hasValidClaim(address wallet, uint256 claimTopic) external view returns (bool) {
    address identity = getIdentity(wallet);
    if (identity == address(0)) return false;
    
    bytes32[] memory claimIds = IEnhancedIdentity(identity).getClaimIdsByTopic(claimTopic);
    
    for (uint256 i = 0; i < claimIds.length; i++) {
        (, , address issuer, , ,) = IEnhancedIdentity(identity).getClaim(claimIds[i]);
        if (trustedIssuerRegistry.hasTrustedIssuerClaimTopic(issuer, claimTopic)) {
            return true;
        }
    }
    
    return false;
}
```

### Security Considerations

1. **Trusted Issuer Management**: Secure management of trusted issuer permissions
2. **Claim Validation**: Proper validation of claim signatures and issuer authorization
3. **Access Control**: Secure attribute and claim management functions
4. **Privacy**: Consider privacy implications of on-chain identity data
5. **Key Management**: Secure key management for identity operations

### Gas Optimization

- Efficient storage patterns for attributes and claims
- Batch operations for multiple claim validations
- Optimized lookup mechanisms for trusted issuers
- Minimal external calls during verification

## Backwards Compatibility

This standard extends ERC734 and ERC735 while maintaining compatibility with existing implementations. Applications can gradually adopt enhanced features while continuing to work with basic ERC734/ERC735 identities.

## Privacy Considerations

While this standard enables powerful identity verification, implementations should consider:
- Minimal data disclosure principles
- Off-chain storage for sensitive attributes
- Zero-knowledge proof integration for privacy-preserving verification
- User consent mechanisms for data sharing

## Test Cases

Comprehensive test cases should cover:
- Identity registration and management
- Trusted issuer operations
- Claim validation and verification
- Attribute management
- Access control integration
- Edge cases and error conditions

## Reference Implementation

The reference implementation includes:
- [`Identity.sol`](../contracts/identity/Identity.sol) - Core identity contract
- [`IIdentity.sol`](../contracts/interfaces/IIdentity.sol) - Interface definition
- [`TrustedIssuersRegistryFacet.sol`](../contracts/facets/TrustedIssuersRegistryFacet.sol) - Trusted issuer management
- [`IdentityRegistryFacet.sol`](../contracts/facets/IdentityRegistryFacet.sol) - Identity registry

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).