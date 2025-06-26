# TrustedIssuersRegistryFacet

## Overview

The [`TrustedIssuersRegistryFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/TrustedIssuersRegistryFacet.sol) manages the registry of trusted issuers within the Gemforce identity system. This facet provides comprehensive functionality for authorizing, managing, and querying trusted entities that can issue and manage identity claims within the decentralized identity framework.

## Contract Details

- **Contract Name**: `TrustedIssuersRegistryFacet`
- **Inheritance**: `ITrustedIssuersRegistry`, `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Trusted Issuer Management
- Add and remove trusted issuers from the registry
- Update issuer permissions and claim topic authorizations
- Complete lifecycle management of issuer credentials

### 🔹 Claim Topic Authorization
- Granular control over which claim topics each issuer can manage
- Add and remove specific claim topic permissions
- Batch claim topic management for efficient operations

### 🔹 Registry Operations
- Query all trusted issuers in the system
- Check issuer status and permissions
- Retrieve issuer-specific claim topic authorizations

### 🔹 Access Control Integration
- Owner-only administrative functions
- Integration with identity registry access control
- Secure permission management for claim operations

## Core Data Structures

### TrustedIssuer Struct
```solidity
struct TrustedIssuer {
    address issuerAddress;      // Address of the trusted issuer
    uint[] claimTopics;        // Array of authorized claim topics
    bool exists;               // Flag indicating if issuer exists
}
```

### Storage Pattern
Uses diamond storage pattern via [`IdentitySystemStorage`](../libraries/identity-system-storage.md):
- Prevents storage collisions in diamond architecture
- Efficient mappings for issuer data and permissions
- Persistent storage across contract upgrades

## Core Functions

### Issuer Management Functions

#### `addTrustedIssuer()`
```solidity
function addTrustedIssuer(
    address _trustedIssuer, 
    uint[] calldata _claimTopics
) external override onlyOwner
```

**Purpose**: Adds a new trusted issuer to the registry with specified claim topic permissions.

**Parameters**:
- `_trustedIssuer` (address): Address of the issuer to add
- `_claimTopics` (uint[]): Array of claim topics the issuer is authorized for

**Access Control**: Owner only

**Validation**:
- Issuer must not already exist in the registry
- Address must be valid (non-zero)

**Process**:
1. Validates issuer doesn't already exist
2. Adds issuer to storage with specified claim topics
3. Updates issuer address list
4. Emits TrustedIssuerAdded event

**Events**: `TrustedIssuerAdded(issuerAddress, claimTopics)`

**Example Usage**:
```solidity
// Add KYC provider as trusted issuer
uint[] memory kycTopics = new uint[](2);
kycTopics[0] = 1; // KYC_TOPIC
kycTopics[1] = 2; // ACCREDITED_INVESTOR_TOPIC

ITrustedIssuersRegistry(diamond).addTrustedIssuer(
    kycProviderAddress,
    kycTopics
);
```

**Error Conditions**:
- `"Trusted issuer already exists"` - Issuer is already registered

---

#### `removeTrustedIssuer()`
```solidity
function removeTrustedIssuer(address _trustedIssuer) external override onlyOwner
```

**Purpose**: Removes a trusted issuer from the registry, revoking all their permissions.

**Parameters**:
- `_trustedIssuer` (address): Address of the issuer to remove

**Access Control**: Owner only

**Validation**: Issuer must exist in the registry

**Process**:
1. Validates issuer exists
2. Removes issuer from storage
3. Cleans up issuer address list
4. Emits TrustedIssuerRemoved event

**Events**: `TrustedIssuerRemoved(issuerAddress)`

**Example Usage**:
```solidity
// Remove compromised or expired issuer
ITrustedIssuersRegistry(diamond).removeTrustedIssuer(expiredIssuerAddress);
```

**Error Conditions**:
- `"Trusted issuer does not exist"` - Issuer is not registered

---

#### `updateIssuerClaimTopics()`
```solidity
function updateIssuerClaimTopics(
    address _trustedIssuer, 
    uint[] calldata _claimTopics
) external override onlyOwner
```

**Purpose**: Updates the claim topics that a trusted issuer is authorized to manage.

**Parameters**:
- `_trustedIssuer` (address): Address of the issuer to update
- `_claimTopics` (uint[]): New array of authorized claim topics

**Access Control**: Owner only

**Validation**: Issuer must exist in the registry

**Process**:
1. Validates issuer exists
2. Replaces existing claim topics with new ones
3. Updates storage mappings
4. Emits ClaimTopicsUpdated event

**Events**: `ClaimTopicsUpdated(issuerAddress, claimTopics)`

**Example Usage**:
```solidity
// Update issuer permissions to include new claim types
uint[] memory updatedTopics = new uint[](3);
updatedTopics[0] = 1; // KYC_TOPIC
updatedTopics[1] = 2; // ACCREDITED_INVESTOR_TOPIC
updatedTopics[2] = 5; // JURISDICTION_TOPIC

ITrustedIssuersRegistry(diamond).updateIssuerClaimTopics(
    issuerAddress,
    updatedTopics
);
```

**Error Conditions**:
- `"Trusted issuer does not exist"` - Issuer is not registered

### Claim Topic Management Functions

#### `addTrustedIssuerClaimTopic()`
```solidity
function addTrustedIssuerClaimTopic(
    address _issuer, 
    uint _claimTopic
) external onlyOwner
```

**Purpose**: Adds a single claim topic to an existing trusted issuer's permissions.

**Parameters**:
- `_issuer` (address): Address of the trusted issuer
- `_claimTopic` (uint): Claim topic to add

**Access Control**: Owner only

**Validation**: Issuer must exist in the registry

**Process**:
1. Validates issuer exists
2. Adds claim topic to issuer's authorized topics
3. Updates storage mappings

**Example Usage**:
```solidity
// Grant additional claim topic permission to existing issuer
ITrustedIssuersRegistry(diamond).addTrustedIssuerClaimTopic(
    issuerAddress,
    NEW_CLAIM_TOPIC
);
```

**Error Conditions**:
- `"Trusted issuer does not exist"` - Issuer is not registered

---

#### `removeTrustedIssuerClaimTopic()`
```solidity
function removeTrustedIssuerClaimTopic(
    address _issuer, 
    uint _claimTopic
) external onlyOwner
```

**Purpose**: Removes a specific claim topic from a trusted issuer's permissions.

**Parameters**:
- `_issuer` (address): Address of the trusted issuer
- `_claimTopic` (uint): Claim topic to remove

**Access Control**: Owner only

**Validation**: Issuer must exist in the registry

**Process**:
1. Validates issuer exists
2. Removes claim topic from issuer's authorized topics
3. Updates storage mappings
4. Emits ClaimTopicRemoved event

**Events**: `ClaimTopicRemoved(issuer, claimTopic)`

**Example Usage**:
```solidity
// Revoke specific claim topic permission
ITrustedIssuersRegistry(diamond).removeTrustedIssuerClaimTopic(
    issuerAddress,
    REVOKED_CLAIM_TOPIC
);
```

**Error Conditions**:
- `"Trusted issuer does not exist"` - Issuer is not registered

### Query Functions

#### `getTrustedIssuer()`
```solidity
function getTrustedIssuer(address issuerAddress) external view override returns (TrustedIssuer memory)
```

**Purpose**: Returns the complete trusted issuer struct for a given address.

**Parameters**:
- `issuerAddress` (address): Address of the issuer to query

**Returns**: TrustedIssuer struct containing issuer details and permissions

**Example Usage**:
```solidity
TrustedIssuer memory issuer = ITrustedIssuersRegistry(diamond).getTrustedIssuer(issuerAddress);

if (issuer.exists) {
    console.log("Issuer has", issuer.claimTopics.length, "authorized claim topics");
}
```

---

#### `getTrustedIssuers()`
```solidity
function getTrustedIssuers() external view override returns (TrustedIssuer[] memory)
```

**Purpose**: Returns an array of all trusted issuers in the registry.

**Returns**: Array of TrustedIssuer structs

**Example Usage**:
```solidity
TrustedIssuer[] memory allIssuers = ITrustedIssuersRegistry(diamond).getTrustedIssuers();

for (uint256 i = 0; i < allIssuers.length; i++) {
    console.log("Issuer:", allIssuers[i].issuerAddress);
    console.log("Topics:", allIssuers[i].claimTopics.length);
}
```

---

#### `isTrustedIssuer()`
```solidity
function isTrustedIssuer(address _issuer) external view override returns (bool)
```

**Purpose**: Checks if an address is registered as a trusted issuer.

**Parameters**:
- `_issuer` (address): Address to check

**Returns**: True if the address is a trusted issuer, false otherwise

**Example Usage**:
```solidity
bool isTrusted = ITrustedIssuersRegistry(diamond).isTrustedIssuer(issuerAddress);

if (isTrusted) {
    // Allow issuer operations
} else {
    revert("Unauthorized issuer");
}
```

---

#### `getTrustedIssuerClaimTopics()`
```solidity
function getTrustedIssuerClaimTopics(address _trustedIssuer) external view override returns (uint[] memory)
```

**Purpose**: Returns all claim topics that a trusted issuer is authorized for.

**Parameters**:
- `_trustedIssuer` (address): Address of the trusted issuer

**Returns**: Array of authorized claim topic IDs

**Example Usage**:
```solidity
uint[] memory authorizedTopics = ITrustedIssuersRegistry(diamond).getTrustedIssuerClaimTopics(issuerAddress);

for (uint256 i = 0; i < authorizedTopics.length; i++) {
    console.log("Authorized for claim topic:", authorizedTopics[i]);
}
```

---

#### `hasTrustedIssuerClaimTopic()`
```solidity
function hasTrustedIssuerClaimTopic(
    address _issuer, 
    uint _claimTopic
) external view override returns (bool)
```

**Purpose**: Checks if a trusted issuer is authorized for a specific claim topic.

**Parameters**:
- `_issuer` (address): Address of the issuer
- `_claimTopic` (uint): Claim topic to check

**Returns**: True if the issuer is authorized for the claim topic, false otherwise

**Example Usage**:
```solidity
bool canIssueKYC = ITrustedIssuersRegistry(diamond).hasTrustedIssuerClaimTopic(
    issuerAddress,
    KYC_TOPIC
);

require(canIssueKYC, "Issuer not authorized for KYC claims");
```

## Integration Examples

### KYC Provider Management
```solidity
// Complete KYC provider setup
contract KYCProviderManager {
    uint256 constant KYC_TOPIC = 1;
    uint256 constant ACCREDITED_TOPIC = 2;
    uint256 constant JURISDICTION_TOPIC = 3;
    
    function setupKYCProvider(address provider) external onlyOwner {
        // 1. Define claim topics for KYC provider
        uint[] memory kycTopics = new uint[](3);
        kycTopics[0] = KYC_TOPIC;
        kycTopics[1] = ACCREDITED_TOPIC;
        kycTopics[2] = JURISDICTION_TOPIC;
        
        // 2. Add as trusted issuer
        ITrustedIssuersRegistry(diamond).addTrustedIssuer(provider, kycTopics);
        
        // 3. Verify setup
        require(
            ITrustedIssuersRegistry(diamond).isTrustedIssuer(provider),
            "KYC provider setup failed"
        );
    }
    
    function revokeKYCProvider(address provider) external onlyOwner {
        // Remove all permissions
        ITrustedIssuersRegistry(diamond).removeTrustedIssuer(provider);
    }
}
```

### Dynamic Permission Management
```solidity
// Manage issuer permissions dynamically
contract IssuerPermissionManager {
    function grantClaimTopicPermission(
        address issuer,
        uint claimTopic
    ) external onlyOwner {
        require(
            ITrustedIssuersRegistry(diamond).isTrustedIssuer(issuer),
            "Not a trusted issuer"
        );
        
        // Check if issuer already has this permission
        if (!ITrustedIssuersRegistry(diamond).hasTrustedIssuerClaimTopic(issuer, claimTopic)) {
            ITrustedIssuersRegistry(diamond).addTrustedIssuerClaimTopic(issuer, claimTopic);
        }
    }
    
    function revokeClaimTopicPermission(
        address issuer,
        uint claimTopic
    ) external onlyOwner {
        if (ITrustedIssuersRegistry(diamond).hasTrustedIssuerClaimTopic(issuer, claimTopic)) {
            ITrustedIssuersRegistry(diamond).removeTrustedIssuerClaimTopic(issuer, claimTopic);
        }
    }
}
```

### Issuer Authorization Middleware
```solidity
// Middleware for validating issuer permissions
contract IssuerAuthorizationMiddleware {
    modifier requiresTrustedIssuer(address issuer) {
        require(
            ITrustedIssuersRegistry(diamond).isTrustedIssuer(issuer),
            "Not a trusted issuer"
        );
        _;
    }
    
    modifier requiresClaimTopicPermission(address issuer, uint claimTopic) {
        require(
            ITrustedIssuersRegistry(diamond).hasTrustedIssuerClaimTopic(issuer, claimTopic),
            "Issuer not authorized for claim topic"
        );
        _;
    }
    
    function issueClaimWithValidation(
        address user,
        uint claimTopic,
        bytes calldata claimData
    ) external 
        requiresTrustedIssuer(msg.sender)
        requiresClaimTopicPermission(msg.sender, claimTopic)
    {
        // Proceed with claim issuance
        IIdentityRegistry(diamond).addClaim(user, claimTopic, claimData);
    }
}
```

### Registry Analytics
```solidity
// Analytics and reporting for trusted issuers
contract TrustedIssuerAnalytics {
    function getRegistryStats() external view returns (
        uint256 totalIssuers,
        uint256 totalClaimTopics,
        address[] memory issuerAddresses
    ) {
        TrustedIssuer[] memory issuers = ITrustedIssuersRegistry(diamond).getTrustedIssuers();
        
        totalIssuers = issuers.length;
        issuerAddresses = new address[](totalIssuers);
        
        uint256 uniqueTopics = 0;
        mapping(uint => bool) seenTopics;
        
        for (uint256 i = 0; i < issuers.length; i++) {
            issuerAddresses[i] = issuers[i].issuerAddress;
            
            // Count unique claim topics
            for (uint256 j = 0; j < issuers[i].claimTopics.length; j++) {
                if (!seenTopics[issuers[i].claimTopics[j]]) {
                    seenTopics[issuers[i].claimTopics[j]] = true;
                    uniqueTopics++;
                }
            }
        }
        
        totalClaimTopics = uniqueTopics;
    }
    
    function getIssuersByClaimTopic(uint claimTopic) external view returns (address[] memory) {
        TrustedIssuer[] memory allIssuers = ITrustedIssuersRegistry(diamond).getTrustedIssuers();
        address[] memory authorizedIssuers = new address[](allIssuers.length);
        uint256 count = 0;
        
        for (uint256 i = 0; i < allIssuers.length; i++) {
            if (ITrustedIssuersRegistry(diamond).hasTrustedIssuerClaimTopic(
                allIssuers[i].issuerAddress, 
                claimTopic
            )) {
                authorizedIssuers[count] = allIssuers[i].issuerAddress;
                count++;
            }
        }
        
        // Resize array to actual count
        assembly {
            mstore(authorizedIssuers, count)
        }
        
        return authorizedIssuers;
    }
}
```

## Events

### Issuer Management Events
```solidity
event TrustedIssuerAdded(address indexed issuer, uint[] claimTopics);
event TrustedIssuerRemoved(address indexed issuer);
event ClaimTopicsUpdated(address indexed issuer, uint[] claimTopics);
```

### Claim Topic Events
```solidity
event ClaimTopicRemoved(address indexed issuer, uint claimTopic);
```

## Security Considerations

### Access Control
- All administrative functions restricted to contract owner
- Prevents unauthorized issuer registration or permission changes
- Secure integration with identity registry access control

### Data Integrity
- Comprehensive validation ensures consistent issuer state
- Diamond storage pattern prevents storage collisions
- Event logging provides complete audit trail

### Permission Management
- Granular claim topic permissions prevent unauthorized claim issuance
- Clear separation between issuer existence and topic authorization
- Secure permission revocation without affecting other permissions

## Gas Optimization

### Storage Efficiency
- Efficient mapping structures for issuer data
- Minimal storage footprint per issuer
- Optimized array operations for claim topics

### Query Optimization
- Direct mapping lookups for permission checks
- Efficient enumeration of all issuers
- Batch operations where applicable

## Error Handling

### Issuer Management Errors
- `"Trusted issuer already exists"` - Duplicate registration attempt
- `"Trusted issuer does not exist"` - Operation on non-existent issuer

### Permission Errors
- Authorization failures handled by calling contracts
- Clear error messages for debugging and user feedback

## Testing Considerations

### Unit Tests
- Issuer addition, removal, and updates
- Claim topic permission management
- Query function accuracy
- Access control validation

### Integration Tests
- Integration with IdentityRegistryFacet
- End-to-end claim issuance workflows
- Permission validation in claim operations
- Event emission verification

## Related Documentation

- [ITrustedIssuersRegistry](../interfaces/itrusted-issuers-registry.md) - Trusted issuers registry interface
- [IdentityRegistryFacet](./identity-registry-facet.md) - Identity registry implementation
- [IdentitySystemStorage](../libraries/identity-system-storage.md) - Storage library
- [EIP-DRAFT-Enhanced-Identity-System](../../eips/EIP-DRAFT-Enhanced-Identity-System.md) - EIP specification
- [Identity Integration Guide](../../guides/identity-integration.md) - Implementation guide

---

*This facet provides the foundation for trusted issuer management within the Gemforce Enhanced Identity System, enabling secure and granular control over identity claim issuance permissions.*