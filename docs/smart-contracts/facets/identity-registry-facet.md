# IdentityRegistryFacet

## Overview

The [`IdentityRegistryFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/IdentityRegistryFacet.sol) provides comprehensive decentralized identity management functionality within the Gemforce diamond system. This facet implements a claims-based identity system where trusted issuers can create, verify, and manage digital identities with associated attestations.

## Contract Details

- **Contract Name**: `IdentityRegistryFacet`
- **Inheritance**: `IIdentityRegistry`, `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Decentralized Identity Management
- Register and manage digital identities on-chain
- Associate user addresses with identity contracts
- Batch operations for efficient identity management
- Complete identity lifecycle management

### 🔹 Claims-Based Verification System
- Add, remove, and manage identity claims (attestations)
- Topic-based claim organization
- Trusted issuer authorization for specific claim types
- Comprehensive claim verification and querying

### 🔹 Access Control & Security
- Multi-level trusted issuer system
- Topic-specific issuer permissions
- Identity ownership validation
- Secure claim management with proper authorization

### 🔹 Registry Operations
- Complete user registry with enumeration
- Identity existence and verification checks
- Efficient batch processing capabilities
- Gas-optimized storage patterns

## Core Data Structures

### Identity System Components
- **Identity Contracts**: ERC734/ERC735-compliant identity contracts
- **Claims**: Verifiable attestations about identities
- **Trusted Issuers**: Authorized entities that can manage identities and claims
- **Claim Topics**: Categorized types of claims (KYC, accreditation, etc.)

### Storage Pattern
Uses diamond storage pattern via [`IdentitySystemStorage`](../libraries/identity-system-storage.md):
- Prevents storage collisions in diamond architecture
- Efficient mappings for identity and claim data
- Persistent storage across contract upgrades

## Access Control Modifiers

### `isTrustedIssuer()`
```solidity
modifier isTrustedIssuer()
```
**Purpose**: Restricts function access to approved trusted issuers only.

**Validation**: Checks if `msg.sender` is registered as a trusted issuer in the system.

**Usage**: Applied to identity management functions that require issuer authorization.

---

### `isTrustedIssuerForClaimTopic(uint256 _claimTopic)`
```solidity
modifier isTrustedIssuerForClaimTopic(uint256 _claimTopic)
```
**Purpose**: Restricts access to issuers authorized for specific claim topics.

**Parameters**:
- `_claimTopic` (uint256): The claim topic requiring authorization

**Validation**: 
- Verifies caller is a trusted issuer
- Confirms issuer is authorized for the specific claim topic

**Usage**: Applied to claim management functions for topic-specific authorization.

---

### `isTrustedIssuerForClaimTopics(uint256[] memory _claimTopics)`
```solidity
modifier isTrustedIssuerForClaimTopics(uint256[] memory _claimTopics)
```
**Purpose**: Restricts access to issuers authorized for multiple claim topics.

**Parameters**:
- `_claimTopics` (uint256[]): Array of claim topics requiring authorization

**Validation**: Ensures caller is authorized for ALL specified claim topics.

**Usage**: Applied to batch claim operations requiring multiple topic permissions.

## Core Functions

### Identity Management Functions

#### `addIdentity()`
```solidity
function addIdentity(address _address, IIdentity identityData) external override isTrustedIssuer
```

**Purpose**: Registers a new identity in the system.

**Parameters**:
- `_address` (address): User address that will own the identity
- `identityData` (IIdentity): Identity contract instance

**Access Control**: Trusted issuers only

**Validation**:
- Address cannot be zero
- Identity must not already exist for the address
- Identity contract must be valid

**Process**:
1. Validates input parameters
2. Checks for existing identity
3. Updates identity mappings
4. Adds to identity owners list
5. Emits IdentityAdded event

**Events**: `IdentityAdded(issuer, address, identityData)`

**Example Usage**:
```solidity
// Deploy new identity contract
IIdentity newIdentity = new Identity(userAddress, managementKey);

// Register identity in the system
IIdentityRegistry(diamond).addIdentity(userAddress, newIdentity);
```

---

#### `batchAddIdentity()`
```solidity
function batchAddIdentity(
    address[] calldata _addresses, 
    IIdentity[] calldata identityDatas
) external override isTrustedIssuer
```

**Purpose**: Efficiently registers multiple identities in a single transaction.

**Parameters**:
- `_addresses` (address[]): Array of user addresses
- `identityDatas` (IIdentity[]): Array of corresponding identity contracts

**Access Control**: Trusted issuers only

**Validation**: Arrays must have equal length

**Gas Optimization**: More efficient than multiple individual calls

**Example Usage**:
```solidity
// Batch register multiple users
address[] memory users = [user1, user2, user3];
IIdentity[] memory identities = [identity1, identity2, identity3];

IIdentityRegistry(diamond).batchAddIdentity(users, identities);
```

---

#### `removeIdentity()`
```solidity
function removeIdentity(address _ownerAddress) external override isTrustedIssuer
```

**Purpose**: Removes an identity from the registry and cleans up all associated data.

**Parameters**:
- `_ownerAddress` (address): Address of the identity owner to remove

**Access Control**: Trusted issuers only

**Validation**: Identity must exist for the specified owner

**Cleanup Process**:
1. Removes from identities mapping (owner → identity)
2. Removes from reverse mapping (identity → owner)
3. Removes from identity owners array using swap-and-pop
4. Emits IdentityRemoved event

**Events**: `IdentityRemoved(issuer, ownerAddress, identityData)`

**Example Usage**:
```solidity
// Remove user's identity from registry
IIdentityRegistry(diamond).removeIdentity(userAddress);
```

### Claim Management Functions

#### `addClaim()`
```solidity
function addClaim(
    address _address, 
    uint256 _claimTopic, 
    bytes calldata _claim
) external override isTrustedIssuerForClaimTopic(_claimTopic)
```

**Purpose**: Adds a new claim (attestation) to an identity.

**Parameters**:
- `_address` (address): User or identity address
- `_claimTopic` (uint256): Topic/type of the claim
- `_claim` (bytes): Claim data

**Access Control**: Trusted issuer for specific claim topic

**Validation**:
- Identity must exist
- Claim must not already exist for this topic
- Issuer must be authorized for the claim topic

**Process**:
1. Resolves address to identity
2. Validates claim doesn't exist
3. Adds claim to storage mappings
4. Updates claim list for the identity
5. Emits ClaimAdded event

**Events**: `ClaimAdded(issuer, identity, claimTopic, claimData)`

**Example Usage**:
```solidity
// Add KYC claim to user's identity
bytes memory kycData = abi.encode("KYC_VERIFIED", block.timestamp);
IIdentityRegistry(diamond).addClaim(userAddress, KYC_TOPIC, kycData);
```

---

#### `setClaims()`
```solidity
function setClaims(
    address _address, 
    uint256[] memory _claims
) external override isTrustedIssuerForClaimTopics(_claims)
```

**Purpose**: Sets multiple claims for an identity in a single transaction.

**Parameters**:
- `_address` (address): User or identity address
- `_claims` (uint256[]): Array of claim topics to set

**Access Control**: Trusted issuer for all specified claim topics

**Validation**:
- Identity must exist
- Claims must not already exist
- Claim topics must be valid (exist in topic list)
- Issuer must be authorized for all claim topics

**Process**:
1. Validates identity existence
2. Iterates through claim topics
3. Validates each claim topic against topic list
4. Adds valid claims to storage
5. Emits individual ClaimAdded events and batch ClaimsSet event

**Events**: `ClaimAdded(...)` for each claim, `ClaimsSet(issuer, identity, claims)`

**Example Usage**:
```solidity
// Set multiple claims for accredited investor
uint256[] memory claims = [KYC_TOPIC, ACCREDITED_INVESTOR_TOPIC, JURISDICTION_TOPIC];
IIdentityRegistry(diamond).setClaims(userAddress, claims);
```

---

#### `removeClaim()`
```solidity
function removeClaim(
    address _address, 
    uint256 _claimTopic
) external override isTrustedIssuerForClaimTopic(_claimTopic)
```

**Purpose**: Removes a specific claim from an identity.

**Parameters**:
- `_address` (address): User or identity address
- `_claimTopic` (uint256): Topic of the claim to remove

**Access Control**: Trusted issuer for specific claim topic

**Validation**:
- Identity must exist
- Cannot remove claims from self
- Claim must exist

**Process**:
1. Resolves address to identity
2. Finds claim in claim list
3. Removes using swap-and-pop method
4. Deletes from claims mapping
5. Emits ClaimRemoved event

**Events**: `ClaimRemoved(issuer, identity, claimTopic)`

**Example Usage**:
```solidity
// Remove expired KYC claim
IIdentityRegistry(diamond).removeClaim(userAddress, KYC_TOPIC);
```

---

#### `unsetClaims()`
```solidity
function unsetClaims(
    address _address, 
    uint256[] memory _claims
) external override isTrustedIssuerForClaimTopics(_claims)
```

**Purpose**: Removes multiple claims from an identity in a single transaction.

**Parameters**:
- `_address` (address): User or identity address
- `_claims` (uint256[]): Array of claim topics to remove

**Access Control**: Trusted issuer for all specified claim topics

**Process**:
1. Validates identity existence
2. Iterates through claims to remove
3. Uses efficient removal algorithm with array resizing
4. Emits ClaimsRemoved event with actually removed claims

**Events**: `ClaimsRemoved(issuer, identity, removedClaims)`

**Gas Optimization**: Batch removal more efficient than individual calls

### Query Functions

#### `identity()`
```solidity
function identity(address _userAddress) external view override returns (address)
```

**Purpose**: Returns the identity contract address for a user address.

**Parameters**:
- `_userAddress` (address): User address to query

**Returns**: Identity contract address (or zero if not found)

**Example Usage**:
```solidity
address identityContract = IIdentityRegistry(diamond).identity(userAddress);
if (identityContract != address(0)) {
    // User has registered identity
}
```

---

#### `contains()`
```solidity
function contains(address _userAddress) external view override returns (bool)
```

**Purpose**: Checks if an identity exists for the given address.

**Parameters**:
- `_userAddress` (address): User address to check

**Returns**: True if identity exists, false otherwise

**Example Usage**:
```solidity
bool hasIdentity = IIdentityRegistry(diamond).contains(userAddress);
require(hasIdentity, "User must have registered identity");
```

---

#### `isVerified()`
```solidity
function isVerified(address _address) external view override returns (bool)
```

**Purpose**: Checks if an address is associated with a verified identity.

**Parameters**:
- `_address` (address): Address to verify (user or identity address)

**Returns**: True if verified, false otherwise

**Example Usage**:
```solidity
bool verified = IIdentityRegistry(diamond).isVerified(userAddress);
if (verified) {
    // Allow access to verified-only features
}
```

---

#### `getClaims()`
```solidity
function getClaims(address _address) external view override returns (uint256[] memory)
```

**Purpose**: Returns all claim topics for an identity.

**Parameters**:
- `_address` (address): User or identity address

**Returns**: Array of claim topics

**Example Usage**:
```solidity
uint256[] memory claims = IIdentityRegistry(diamond).getClaims(userAddress);
for (uint256 i = 0; i < claims.length; i++) {
    console.log("User has claim topic:", claims[i]);
}
```

---

#### `getClaim()`
```solidity
function getClaim(address _address, uint256 _claimTopic) external view override returns (uint256)
```

**Purpose**: Returns a specific claim value for an identity.

**Parameters**:
- `_address` (address): User or identity address
- `_claimTopic` (uint256): Claim topic to query

**Returns**: Claim value (or 0 if not found)

**Example Usage**:
```solidity
uint256 kycClaim = IIdentityRegistry(diamond).getClaim(userAddress, KYC_TOPIC);
if (kycClaim != 0) {
    // User has KYC claim
}
```

---

#### `hasClaim()`
```solidity
function hasClaim(address _address, uint256 _claimTopic) external view override returns (bool)
```

**Purpose**: Checks if an identity has a specific claim.

**Parameters**:
- `_address` (address): User or identity address
- `_claimTopic` (uint256): Claim topic to check

**Returns**: True if claim exists, false otherwise

**Example Usage**:
```solidity
bool hasKYC = IIdentityRegistry(diamond).hasClaim(userAddress, KYC_TOPIC);
require(hasKYC, "KYC verification required");
```

---

#### `getRegistryUsers()`
```solidity
function getRegistryUsers() external view override returns (address[] memory)
```

**Purpose**: Returns all registered user addresses in the system.

**Returns**: Array of user addresses with registered identities

**Example Usage**:
```solidity
address[] memory users = IIdentityRegistry(diamond).getRegistryUsers();
console.log("Total registered users:", users.length);
```

---

#### `isRegistryUser()`
```solidity
function isRegistryUser(address _registryUser) external view override returns (bool)
```

**Purpose**: Checks if an address is a registered user in the system.

**Parameters**:
- `_registryUser` (address): Address to check

**Returns**: True if registered, false otherwise

**Example Usage**:
```solidity
bool isRegistered = IIdentityRegistry(diamond).isRegistryUser(userAddress);
if (!isRegistered) {
    // Prompt user to register identity
}
```

## Integration Examples

### KYC Verification System
```solidity
// Complete KYC verification workflow
contract KYCVerification {
    uint256 constant KYC_TOPIC = 1;
    uint256 constant ACCREDITED_TOPIC = 2;
    
    function verifyAndRegisterUser(
        address user,
        bytes calldata kycData,
        bool isAccredited
    ) external onlyKYCProvider {
        // 1. Deploy identity contract
        IIdentity identity = new Identity(user, getManagementKey(user));
        
        // 2. Register identity
        IIdentityRegistry(diamond).addIdentity(user, identity);
        
        // 3. Add KYC claim
        IIdentityRegistry(diamond).addClaim(user, KYC_TOPIC, kycData);
        
        // 4. Add accredited investor claim if applicable
        if (isAccredited) {
            bytes memory accreditedData = abi.encode("ACCREDITED", block.timestamp);
            IIdentityRegistry(diamond).addClaim(user, ACCREDITED_TOPIC, accreditedData);
        }
    }
}
```

### Access Control Integration
```solidity
// Use identity system for access control
modifier requiresKYC(address user) {
    require(
        IIdentityRegistry(diamond).hasClaim(user, KYC_TOPIC),
        "KYC verification required"
    );
    _;
}

modifier requiresAccreditation(address user) {
    require(
        IIdentityRegistry(diamond).hasClaim(user, ACCREDITED_TOPIC),
        "Accredited investor status required"
    );
    _;
}

function restrictedFunction() external requiresKYC(msg.sender) {
    // Function only accessible to KYC-verified users
}
```

### Batch User Management
```solidity
// Efficient batch user onboarding
function batchOnboardUsers(
    address[] calldata users,
    IIdentity[] calldata identities,
    uint256[][] calldata userClaims
) external onlyTrustedIssuer {
    // 1. Batch register identities
    IIdentityRegistry(diamond).batchAddIdentity(users, identities);
    
    // 2. Set claims for each user
    for (uint256 i = 0; i < users.length; i++) {
        if (userClaims[i].length > 0) {
            IIdentityRegistry(diamond).setClaims(users[i], userClaims[i]);
        }
    }
}
```

### Identity Portfolio Dashboard
```solidity
// Get complete user identity information
function getUserIdentityInfo(address user) external view returns (
    bool hasIdentity,
    address identityContract,
    uint256[] memory claims,
    bool isKYCVerified,
    bool isAccredited
) {
    hasIdentity = IIdentityRegistry(diamond).contains(user);
    
    if (hasIdentity) {
        identityContract = IIdentityRegistry(diamond).identity(user);
        claims = IIdentityRegistry(diamond).getClaims(user);
        isKYCVerified = IIdentityRegistry(diamond).hasClaim(user, KYC_TOPIC);
        isAccredited = IIdentityRegistry(diamond).hasClaim(user, ACCREDITED_TOPIC);
    }
}
```

## Events

### Identity Events
```solidity
event IdentityAdded(address indexed issuer, address indexed user, IIdentity indexed identity);
event IdentityRemoved(address indexed issuer, address indexed user, IIdentity indexed identity);
```

### Claim Events
```solidity
event ClaimAdded(address indexed issuer, address indexed identity, uint256 indexed claimTopic, bytes claimData);
event ClaimRemoved(address indexed issuer, address indexed identity, uint256 indexed claimTopic);
event ClaimsSet(address indexed issuer, address indexed identity, uint256[] claimTopics);
event ClaimsRemoved(address indexed issuer, address indexed identity, uint256[] claimTopics);
```

## Security Considerations

### Access Control
- Multi-level trusted issuer system prevents unauthorized identity management
- Topic-specific permissions ensure issuers can only manage authorized claim types
- Identity ownership validation prevents unauthorized claim modifications

### Data Integrity
- Diamond storage pattern prevents storage collisions
- Comprehensive validation ensures data consistency
- Event logging provides complete audit trail

### Privacy & Compliance
- Claims-based system supports privacy-preserving verification
- Flexible claim topics support various compliance requirements
- Immutable blockchain records provide audit trail

## Gas Optimization

### Batch Operations
- `batchAddIdentity()` for efficient bulk registration
- `setClaims()` and `unsetClaims()` for bulk claim management
- Optimized array operations using swap-and-pop

### Storage Efficiency
- Efficient mapping structures in IdentitySystemStorage
- Minimal storage footprint per identity
- Optimized for frequent read operations

## Error Handling

### Identity Errors
- `"Invalid identity address"` - Zero address provided
- `"Identity already exists"` - Duplicate registration attempt
- `"Identity does not exist"` - Operation on non-existent identity

### Claim Errors
- `"Claim already exists"` - Duplicate claim addition
- `"Claim does not exist"` - Operation on non-existent claim
- `"Cannot set claims for self"` - Self-claim prevention

### Authorization Errors
- `"Not trusted issuer"` - Unauthorized issuer access
- `"Issuer not authorized for claim topic"` - Topic-specific authorization failure
- `"Arrays length mismatch"` - Batch operation parameter mismatch

## Testing Considerations

### Unit Tests
- Identity registration and removal
- Claim addition, removal, and querying
- Access control validation
- Batch operation efficiency

### Integration Tests
- Multi-facet identity workflows
- KYC/compliance integration
- Access control integration
- Event emission verification

## Related Documentation

- [IIdentityRegistry](../interfaces/iidentity-registry.md) - Identity registry interface
- [IdentitySystemStorage](../libraries/identity-system-storage.md) - Storage library
- [TrustedIssuersRegistryFacet](./trusted-issuers-registry-facet.md) - Trusted issuer management
- [EIP-DRAFT-Enhanced-Identity-System](../../eips/EIP-DRAFT-Enhanced-Identity-System.md) - EIP specification
- [Identity Integration Guide](../../guides/identity-integration.md) - Implementation guide

---

*This facet implements the Enhanced Identity System as defined in the Gemforce EIP suite, providing comprehensive decentralized identity management with claims-based verification.*