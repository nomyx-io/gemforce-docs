# IERC735 Interface

## Overview

The `IERC735` interface defines the Claim Holder standard for decentralized identity management within the Gemforce platform. This interface implements the ERC-735 standard for managing identity claims, enabling attestations, verifications, and credential management through cryptographically signed claims from trusted issuers.

## Key Features

- **Identity Claims Management**: Add, modify, and remove identity claims
- **Cryptographic Verification**: Support for multiple signature schemes
- **Trusted Issuer System**: Claims issued by verified and trusted entities
- **Topic-Based Organization**: Claims categorized by topics (KYC, credentials, etc.)
- **URI-Based Metadata**: Support for off-chain claim data and documentation
- **Event Tracking**: Comprehensive event emission for claim lifecycle operations

## Interface Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.16;

interface IERC735 {
    // Events
    event ClaimRequested(uint256 indexed claimRequestId, uint256 indexed topic, uint256 scheme, address indexed issuer, bytes signature, bytes data, string uri);
    event ClaimAdded(bytes32 indexed claimId, uint256 indexed topic, uint256 scheme, address indexed issuer, bytes signature, bytes data, string uri);
    event ClaimRemoved(bytes32 indexed claimId, uint256 indexed topic, uint256 scheme, address indexed issuer, bytes signature, bytes data, string uri);
    event ClaimChanged(bytes32 indexed claimId, uint256 indexed topic, uint256 scheme, address indexed issuer, bytes signature, bytes data, string uri);
    
    // Core Functions
    function getClaim(bytes32 _claimId) external returns(uint256 topic, uint256 scheme, address issuer, bytes memory signature, bytes memory data, string memory uri);
    function getClaimIdsByTopic(uint256 _topic) external returns(bytes32[] memory claimIds);
    function addClaim(uint256 _topic, uint256 _scheme, address _issuer, bytes memory _signature, bytes memory _data, string memory _uri) external returns (uint256 claimRequestId);
    function changeClaim(bytes32 _claimId, uint256 _topic, uint256 _scheme, address _issuer, bytes memory _signature, bytes memory _data, string memory _uri) external returns (bool success);
    function removeClaim(bytes32 _claimId) external returns (bool success);
}
```

## Claim Topic Constants

```solidity
// Standard claim topics as defined in ERC-735
uint256 public constant BIOMETRIC_TOPIC = 1;        // Biometric verification
uint256 public constant RESIDENCE_TOPIC = 2;        // Proof of residence
uint256 public constant REGISTRY_TOPIC = 3;         // Registry inclusion
uint256 public constant PROFILE_TOPIC = 4;          // Profile information
uint256 public constant CONTRACT_TOPIC = 5;         // Contract interaction
uint256 public constant KYC_TOPIC = 10001;          // Know Your Customer
uint256 public constant AML_TOPIC = 10002;          // Anti-Money Laundering
uint256 public constant ACCREDITED_INVESTOR_TOPIC = 10003; // Accredited investor status
uint256 public constant PROFESSIONAL_TOPIC = 10004;  // Professional credentials
```

## Signature Scheme Constants

```solidity
// Standard signature schemes
uint256 public constant ECDSA_SCHEME = 1;           // ECDSA signature
uint256 public constant RSA_SCHEME = 2;             // RSA signature
uint256 public constant CONTRACT_SCHEME = 3;        // Contract-based verification
uint256 public constant MULTISIG_SCHEME = 4;        // Multi-signature scheme
```

## Core Functions

### Claim Management

#### `addClaim()`
```solidity
function addClaim(uint256 _topic, uint256 _scheme, address _issuer, bytes memory _signature, bytes memory _data, string memory _uri) external returns (uint256 claimRequestId)
```

**Purpose**: Add a new identity claim to the claim holder.

**Parameters**:
- `_topic` (uint256): The claim topic identifier (KYC, AML, etc.)
- `_scheme` (uint256): The signature scheme used for verification
- `_issuer` (address): The address of the claim issuer
- `_signature` (bytes): The cryptographic signature of the claim
- `_data` (bytes): The claim data payload
- `_uri` (string): URI pointing to additional claim metadata

**Returns**: 
- `claimRequestId` (uint256): Unique identifier for the claim request

**Requirements**:
- Issuer must be authorized for the claim topic
- Signature must be valid for the claim data
- Claim topic must be supported
- Caller must have appropriate permissions

**Events Emitted**:
- `ClaimRequested` initially when claim is submitted
- `ClaimAdded` when claim is approved and added

**Example Usage**:
```solidity
// Add a KYC claim from a trusted issuer
uint256 topic = KYC_TOPIC;
uint256 scheme = ECDSA_SCHEME;
address issuer = 0x742d35Cc6634C0532925a3b8D4C9db96590c6C8C;
bytes memory signature = abi.encodePacked(r, s, v); // ECDSA signature components
bytes memory data = abi.encode("John Doe", "US", block.timestamp);
string memory uri = "https://kyc-provider.com/claims/12345";

uint256 requestId = IERC735(identityContract).addClaim(
    topic, scheme, issuer, signature, data, uri
);
console.log("KYC claim request ID:", requestId);
```

#### `changeClaim()`
```solidity
function changeClaim(bytes32 _claimId, uint256 _topic, uint256 _scheme, address _issuer, bytes memory _signature, bytes memory _data, string memory _uri) external returns (bool success)
```

**Purpose**: Modify an existing identity claim.

**Parameters**:
- `_claimId` (bytes32): The unique identifier of the claim to modify
- `_topic` (uint256): The updated claim topic
- `_scheme` (uint256): The updated signature scheme
- `_issuer` (address): The updated issuer address
- `_signature` (bytes): The updated cryptographic signature
- `_data` (bytes): The updated claim data
- `_uri` (string): The updated metadata URI

**Returns**: Boolean indicating success of the operation

**Requirements**:
- Claim must exist
- Caller must have permission to modify the claim
- New signature must be valid
- Issuer must be authorized for the topic

**Events Emitted**:
- `ClaimChanged` with old and new claim details

**Example Usage**:
```solidity
// Update an existing KYC claim with new information
bytes32 claimId = 0x1234567890abcdef...;
uint256 topic = KYC_TOPIC;
uint256 scheme = ECDSA_SCHEME;
address issuer = 0x742d35Cc6634C0532925a3b8D4C9db96590c6C8C;
bytes memory newSignature = abi.encodePacked(r, s, v);
bytes memory newData = abi.encode("John Doe", "US", block.timestamp, "updated");
string memory newUri = "https://kyc-provider.com/claims/12345-updated";

bool success = IERC735(identityContract).changeClaim(
    claimId, topic, scheme, issuer, newSignature, newData, newUri
);
console.log("Claim update success:", success);
```

#### `removeClaim()`
```solidity
function removeClaim(bytes32 _claimId) external returns (bool success)
```

**Purpose**: Remove an identity claim from the claim holder.

**Parameters**:
- `_claimId` (bytes32): The unique identifier of the claim to remove

**Returns**: Boolean indicating success of the operation

**Requirements**:
- Claim must exist
- Caller must have permission to remove the claim
- Some claims may be required and cannot be removed

**Events Emitted**:
- `ClaimRemoved` with claim details

**Example Usage**:
```solidity
// Remove an expired or invalid claim
bytes32 claimId = 0x1234567890abcdef...;

bool success = IERC735(identityContract).removeClaim(claimId);
console.log("Claim removal success:", success);
```

## Query Functions

### Claim Information

#### `getClaim()`
```solidity
function getClaim(bytes32 _claimId) external returns(uint256 topic, uint256 scheme, address issuer, bytes memory signature, bytes memory data, string memory uri)
```

**Purpose**: Retrieve detailed information about a specific claim.

**Parameters**:
- `_claimId` (bytes32): The unique identifier of the claim to query

**Returns**:
- `topic` (uint256): The claim topic identifier
- `scheme` (uint256): The signature scheme used
- `issuer` (address): The address of the claim issuer
- `signature` (bytes): The cryptographic signature
- `data` (bytes): The claim data payload
- `uri` (string): The metadata URI

**Example Usage**:
```solidity
// Get detailed claim information
bytes32 claimId = 0x1234567890abcdef...;
(
    uint256 topic,
    uint256 scheme,
    address issuer,
    bytes memory signature,
    bytes memory data,
    string memory uri
) = IERC735(identityContract).getClaim(claimId);

console.log("Claim topic:", topic);
console.log("Claim issuer:", issuer);
console.log("Claim URI:", uri);
```

#### `getClaimIdsByTopic()`
```solidity
function getClaimIdsByTopic(uint256 _topic) external returns(bytes32[] memory claimIds)
```

**Purpose**: Get all claim IDs associated with a specific topic.

**Parameters**:
- `_topic` (uint256): The topic to query claims for

**Returns**: Array of claim IDs for the specified topic

**Example Usage**:
```solidity
// Get all KYC claims for an identity
uint256 topic = KYC_TOPIC;
bytes32[] memory kycClaims = IERC735(identityContract).getClaimIdsByTopic(topic);

console.log("KYC claims count:", kycClaims.length);
for (uint256 i = 0; i < kycClaims.length; i++) {
    console.log("KYC claim ID:", kycClaims[i]);
}
```

## Integration Examples

### KYC/AML Verification System
```solidity
// KYC/AML verification system using ERC-735 claims
contract KYCAMLVerification {
    IERC735 public claimHolder;
    
    struct VerificationRequirement {
        uint256[] requiredTopics;
        address[] trustedIssuers;
        uint256 minValidityPeriod;
        bool active;
    }
    
    mapping(bytes32 => VerificationRequirement) public verificationLevels;
    mapping(address => mapping(uint256 => bool)) public issuerTopicAuthorization;
    mapping(bytes32 => uint256) public claimExpirationTime;
    
    event VerificationLevelCreated(bytes32 indexed levelId, uint256[] requiredTopics);
    event IssuerAuthorized(address indexed issuer, uint256 indexed topic, bool authorized);
    event IdentityVerified(address indexed identity, bytes32 indexed levelId, uint256 timestamp);
    event VerificationFailed(address indexed identity, bytes32 indexed levelId, string reason);
    
    constructor(address _claimHolder) {
        claimHolder = IERC735(_claimHolder);
    }
    
    function createVerificationLevel(
        bytes32 levelId,
        uint256[] memory requiredTopics,
        address[] memory trustedIssuers,
        uint256 minValidityPeriod
    ) external onlyAdmin {
        verificationLevels[levelId] = VerificationRequirement({
            requiredTopics: requiredTopics,
            trustedIssuers: trustedIssuers,
            minValidityPeriod: minValidityPeriod,
            active: true
        });
        
        emit VerificationLevelCreated(levelId, requiredTopics);
    }
    
    function authorizeIssuerForTopic(
        address issuer,
        uint256 topic,
        bool authorized
    ) external onlyAdmin {
        issuerTopicAuthorization[issuer][topic] = authorized;
        emit IssuerAuthorized(issuer, topic, authorized);
    }
    
    function verifyIdentity(
        address identity,
        bytes32 levelId
    ) external returns (bool verified) {
        VerificationRequirement memory requirement = verificationLevels[levelId];
        require(requirement.active, "Verification level not active");
        
        // Check each required topic
        for (uint256 i = 0; i < requirement.requiredTopics.length; i++) {
            uint256 topic = requirement.requiredTopics[i];
            
            bytes32[] memory claimIds = claimHolder.getClaimIdsByTopic(topic);
            if (claimIds.length == 0) {
                emit VerificationFailed(identity, levelId, "Missing required claim topic");
                return false;
            }
            
            bool validClaimFound = false;
            for (uint256 j = 0; j < claimIds.length; j++) {
                if (_isClaimValid(claimIds[j], requirement)) {
                    validClaimFound = true;
                    break;
                }
            }
            
            if (!validClaimFound) {
                emit VerificationFailed(identity, levelId, "No valid claim found for topic");
                return false;
            }
        }
        
        emit IdentityVerified(identity, levelId, block.timestamp);
        return true;
    }
    
    function _isClaimValid(
        bytes32 claimId,
        VerificationRequirement memory requirement
    ) internal returns (bool) {
        (
            uint256 topic,
            uint256 scheme,
            address issuer,
            bytes memory signature,
            bytes memory data,
            string memory uri
        ) = claimHolder.getClaim(claimId);
        
        // Check if issuer is authorized for this topic
        if (!issuerTopicAuthorization[issuer][topic]) {
            return false;
        }
        
        // Check if claim is not expired
        uint256 expirationTime = claimExpirationTime[claimId];
        if (expirationTime > 0 && block.timestamp > expirationTime) {
            return false;
        }
        
        // Verify signature (simplified - would implement full signature verification)
        if (signature.length == 0) {
            return false;
        }
        
        return true;
    }
    
    function getVerificationStatus(
        address identity,
        bytes32 levelId
    ) external view returns (
        bool canVerify,
        uint256[] memory missingTopics,
        uint256[] memory expiredClaims
    ) {
        VerificationRequirement memory requirement = verificationLevels[levelId];
        
        uint256[] memory missing = new uint256[](requirement.requiredTopics.length);
        uint256[] memory expired = new uint256[](requirement.requiredTopics.length);
        uint256 missingCount = 0;
        uint256 expiredCount = 0;
        
        canVerify = true;
        
        for (uint256 i = 0; i < requirement.requiredTopics.length; i++) {
            uint256 topic = requirement.requiredTopics[i];
            bytes32[] memory claimIds = claimHolder.getClaimIdsByTopic(topic);
            
            if (claimIds.length == 0) {
                missing[missingCount++] = topic;
                canVerify = false;
            } else {
                bool hasValidClaim = false;
                for (uint256 j = 0; j < claimIds.length; j++) {
                    uint256 expirationTime = claimExpirationTime[claimIds[j]];
                    if (expirationTime == 0 || block.timestamp <= expirationTime) {
                        hasValidClaim = true;
                        break;
                    }
                }
                
                if (!hasValidClaim) {
                    expired[expiredCount++] = topic;
                    canVerify = false;
                }
            }
        }
        
        // Resize arrays to actual counts
        missingTopics = new uint256[](missingCount);
        expiredClaims = new uint256[](expiredCount);
        
        for (uint256 i = 0; i < missingCount; i++) {
            missingTopics[i] = missing[i];
        }
        
        for (uint256 i = 0; i < expiredCount; i++) {
            expiredClaims[i] = expired[i];
        }
    }
    
    modifier onlyAdmin() {
        // Implementation would check for admin role
        _;
    }
}
```

### Professional Credentials System
```solidity
// Professional credentials and certification system
contract ProfessionalCredentials {
    IERC735 public claimHolder;
    
    struct Credential {
        string name;
        string description;
        uint256 topic;
        address issuingAuthority;
        uint256 validityPeriod;
        bool requiresRenewal;
        uint256 renewalPeriod;
    }
    
    struct IssuedCredential {
        bytes32 credentialId;
        address holder;
        uint256 issuedAt;
        uint256 expiresAt;
        bool active;
        bytes32 claimId;
    }
    
    mapping(bytes32 => Credential) public credentials;
    mapping(bytes32 => IssuedCredential) public issuedCredentials;
    mapping(address => bytes32[]) public holderCredentials;
    mapping(address => bool) public authorizedIssuers;
    
    event CredentialDefined(bytes32 indexed credentialId, string name, uint256 topic);
    event CredentialIssued(bytes32 indexed credentialId, address indexed holder, bytes32 claimId);
    event CredentialRevoked(bytes32 indexed credentialId, address indexed holder, string reason);
    event CredentialRenewed(bytes32 indexed credentialId, address indexed holder, uint256 newExpirationTime);
    
    constructor(address _gameCards) {
        claimHolder = IERC735(_gameCards);
    }
    
    function defineCredential(
        bytes32 credentialId,
        string memory name,
        string memory description,
        uint256 topic,
        address issuingAuthority,
        uint256 validityPeriod,
        bool requiresRenewal,
        uint256 renewalPeriod
    ) external onlyAdmin {
        credentials[credentialId] = Credential({
            name: name,
            description: description,
            topic: topic,
            issuingAuthority: issuingAuthority,
            validityPeriod: validityPeriod,
            requiresRenewal: requiresRenewal,
            renewalPeriod: renewalPeriod
        });
        
        authorizedIssuers[issuingAuthority] = true;
        emit CredentialDefined(credentialId, name, topic);
    }
    
    function issueCredential(
        bytes32 credentialId,
        address holder,
        bytes memory credentialData,
        bytes memory signature,
        string memory uri
    ) external onlyAuthorizedIssuer(credentialId) {
        Credential memory credential = credentials[credentialId];
        require(bytes(credential.name).length > 0, "Credential not defined");