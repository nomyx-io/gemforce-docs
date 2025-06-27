# IERC734 Interface

## Overview

The `IERC734` interface defines the Key Manager standard for decentralized identity management within the Gemforce platform. This interface implements the ERC-734 standard for managing cryptographic keys associated with identity contracts, enabling multi-signature operations, role-based access control, and secure execution of transactions through key-based authorization.

## Key Features

- **Multi-Key Management**: Support for multiple cryptographic keys with different purposes
- **Purpose-Based Authorization**: Keys can be assigned specific purposes (management, action, claim, encryption)
- **Key Type Support**: Different cryptographic key types (ECDSA, RSA, etc.)
- **Execution Management**: Secure transaction execution with multi-signature approval
- **Event Tracking**: Comprehensive event emission for key and execution operations

## Interface Definition

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;

import { IERC165 } from "./IERC165.sol";

interface IERC734 is IERC165 {
    // Events
    event KeyAdded(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
    event KeyRemoved(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
    event ExecutionRequested(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
    event Executed(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
    event ExecutionFailed(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
    event Approved(uint256 indexed executionId, bool approved);
    
    // Key Management Functions
    function addKey(bytes32 _key, uint256 _purpose, uint256 _keyType) external;
    function removeKey(bytes32 _key, uint256 _purpose) external;
    function approve(uint256 _id, bool _approve) external;
    
    // Query Functions
    function getKey(bytes32 _key) external view returns(uint256[] memory purposes, uint256 keyType, bytes32 key);
    function getKeyPurposes(bytes32 _key) external view returns(uint256[] memory);
    function getKeysByPurpose(uint256 _purpose) external view returns(bytes32[] memory);
    function getExecution(uint256 _id) external view returns(address to, uint256 value, bytes memory data, bool approved, uint256 executionType);
}
```

## Key Purpose Constants

```solidity
// Standard key purposes as defined in ERC-734
uint256 public constant MANAGEMENT_KEY = 1;    // Can manage the identity
uint256 public constant ACTION_KEY = 2;        // Can perform actions
uint256 public constant CLAIM_SIGNER_KEY = 3;  // Can sign claims
uint256 public constant ENCRYPTION_KEY = 4;    // Can encrypt/decrypt data
```

## Key Type Constants

```solidity
// Standard key types
uint256 public constant ECDSA_TYPE = 1;        // ECDSA key type
uint256 public constant RSA_TYPE = 2;          // RSA key type
uint256 public constant MULTISIG_TYPE = 3;     // Multi-signature key type
```

## Core Functions

### Key Management

#### `addKey()`
```solidity
function addKey(bytes32 _key, uint256 _purpose, uint256 _keyType) external
```

**Purpose**: Add a new key to the identity with specified purpose and type.

**Parameters**:
- `_key` (bytes32): The key identifier (typically keccak256 hash of the public key)
- `_purpose` (uint256): The purpose of the key (MANAGEMENT_KEY, ACTION_KEY, etc.)
- `_keyType` (uint256): The cryptographic type of the key (ECDSA_TYPE, RSA_TYPE, etc.)

**Requirements**:
- Caller must have MANAGEMENT_KEY purpose
- Key must not already exist with the same purpose
- Purpose and key type must be valid

**Events Emitted**:
- `KeyAdded` with key, purpose, and key type

**Example Usage**:
```solidity
// Add a new action key for transaction execution
bytes32 newKey = keccak256(abi.encodePacked(publicKey));
uint256 purpose = ACTION_KEY;
uint256 keyType = ECDSA_TYPE;

IERC734(identityContract).addKey(newKey, purpose, keyType);
console.log("Added action key:", newKey);
```

#### `removeKey()`
```solidity
function removeKey(bytes32 _key, uint256 _purpose) external
```

**Purpose**: Remove a key from the identity for a specific purpose.

**Parameters**:
- `_key` (bytes32): The key identifier to remove
- `_purpose` (uint256): The purpose for which to remove the key

**Requirements**:
- Caller must have MANAGEMENT_KEY purpose
- Key must exist with the specified purpose
- Cannot remove the last management key

**Events Emitted**:
- `KeyRemoved` with key, purpose, and key type

**Example Usage**:
```solidity
// Remove an action key
bytes32 keyToRemove = 0x1234567890abcdef...;
uint256 purpose = ACTION_KEY;

IERC734(identityContract).removeKey(keyToRemove, purpose);
console.log("Removed key:", keyToRemove, "for purpose:", purpose);
```

### Execution Management

#### `approve()`
```solidity
function approve(uint256 _id, bool _approve) external
```

**Purpose**: Approve or reject a pending execution request.

**Parameters**:
- `_id` (uint256): The execution ID to approve or reject
- `_approve` (bool): True to approve, false to reject

**Requirements**:
- Caller must have appropriate key purpose for the execution
- Execution must exist and be pending
- Caller must not have already voted on this execution

**Events Emitted**:
- `Approved` with execution ID and approval status
- `Executed` if execution is approved and executed successfully
- `ExecutionFailed` if execution is approved but fails

**Example Usage**:
```solidity
// Approve a pending execution
uint256 executionId = 1;
bool approve = true;

IERC734(identityContract).approve(executionId, approve);
console.log("Approved execution:", executionId);
```

## Query Functions

### Key Information

#### `getKey()`
```solidity
function getKey(bytes32 _key) external view returns(uint256[] memory purposes, uint256 keyType, bytes32 key)
```

**Purpose**: Get detailed information about a specific key.

**Parameters**:
- `_key` (bytes32): The key identifier to query

**Returns**:
- `purposes` (uint256[]): Array of purposes assigned to this key
- `keyType` (uint256): The cryptographic type of the key
- `key` (bytes32): The key identifier (same as input)

**Example Usage**:
```solidity
// Get key information
bytes32 keyId = 0x1234567890abcdef...;
(uint256[] memory purposes, uint256 keyType, bytes32 key) = IERC734(identityContract).getKey(keyId);

console.log("Key purposes:", purposes.length);
console.log("Key type:", keyType);
```

#### `getKeyPurposes()`
```solidity
function getKeyPurposes(bytes32 _key) external view returns(uint256[] memory)
```

**Purpose**: Get all purposes assigned to a specific key.

**Parameters**:
- `_key` (bytes32): The key identifier to query

**Returns**: Array of purposes assigned to the key

**Example Usage**:
```solidity
// Get key purposes
bytes32 keyId = 0x1234567890abcdef...;
uint256[] memory purposes = IERC734(identityContract).getKeyPurposes(keyId);

for (uint256 i = 0; i < purposes.length; i++) {
    console.log("Purpose:", purposes[i]);
}
```

#### `getKeysByPurpose()`
```solidity
function getKeysByPurpose(uint256 _purpose) external view returns(bytes32[] memory)
```

**Purpose**: Get all keys assigned to a specific purpose.

**Parameters**:
- `_purpose` (uint256): The purpose to query keys for

**Returns**: Array of key identifiers with the specified purpose

**Example Usage**:
```solidity
// Get all management keys
uint256 purpose = MANAGEMENT_KEY;
bytes32[] memory managementKeys = IERC734(identityContract).getKeysByPurpose(purpose);

console.log("Management keys count:", managementKeys.length);
for (uint256 i = 0; i < managementKeys.length; i++) {
    console.log("Management key:", managementKeys[i]);
}
```

### Execution Information

#### `getExecution()`
```solidity
function getExecution(uint256 _id) external view returns(address to, uint256 value, bytes memory data, bool approved, uint256 executionType)
```

**Purpose**: Get detailed information about a specific execution request.

**Parameters**:
- `_id` (uint256): The execution ID to query

**Returns**:
- `to` (address): The target address for the execution
- `value` (uint256): The ETH value to send with the execution
- `data` (bytes): The call data for the execution
- `approved` (bool): Whether the execution has been approved
- `executionType` (uint256): The type of execution

**Example Usage**:
```solidity
// Get execution details
uint256 executionId = 1;
(
    address to,
    uint256 value,
    bytes memory data,
    bool approved,
    uint256 executionType
) = IERC734(identityContract).getExecution(executionId);

console.log("Execution target:", to);
console.log("Execution value:", value);
console.log("Execution approved:", approved);
```

## Integration Examples

### Multi-Signature Identity Wallet
```solidity
// Multi-signature wallet using ERC-734 key management
contract MultiSigIdentityWallet {
    IERC734 public keyManager;
    
    struct Transaction {
        address to;
        uint256 value;
        bytes data;
        bool executed;
        uint256 confirmations;
        mapping(address => bool) isConfirmed;
    }
    
    mapping(uint256 => Transaction) public transactions;
    uint256 public transactionCount;
    uint256 public required; // Required confirmations
    
    event TransactionSubmitted(uint256 indexed transactionId, address indexed to, uint256 value);
    event TransactionConfirmed(uint256 indexed transactionId, address indexed confirmer);
    event TransactionExecuted(uint256 indexed transactionId);
    
    constructor(address _keyManager, uint256 _required) {
        keyManager = IERC734(_keyManager);
        required = _required;
    }
    
    function submitTransaction(
        address to,
        uint256 value,
        bytes memory data
    ) external onlyActionKey returns (uint256 transactionId) {
        transactionId = transactionCount++;
        
        Transaction storage txn = transactions[transactionId];
        txn.to = to;
        txn.value = value;
        txn.data = data;
        txn.executed = false;
        txn.confirmations = 0;
        
        emit TransactionSubmitted(transactionId, to, value);
        
        // Auto-confirm from submitter
        confirmTransaction(transactionId);
    }
    
    function confirmTransaction(uint256 transactionId) public onlyActionKey {
        Transaction storage txn = transactions[transactionId];
        require(!txn.executed, "Transaction already executed");
        require(!txn.isConfirmed[msg.sender], "Transaction already confirmed by sender");
        
        txn.isConfirmed[msg.sender] = true;
        txn.confirmations++;
        
        emit TransactionConfirmed(transactionId, msg.sender);
        
        if (txn.confirmations >= required) {
            executeTransaction(transactionId);
        }
    }
    
    function executeTransaction(uint256 transactionId) internal {
        Transaction storage txn = transactions[transactionId];
        require(!txn.executed, "Transaction already executed");
        require(txn.confirmations >= required, "Insufficient confirmations");
        
        txn.executed = true;
        
        (bool success, ) = txn.to.call{value: txn.value}(txn.data);
        require(success, "Transaction execution failed");
        
        emit TransactionExecuted(transactionId);
    }
    
    function getTransactionInfo(uint256 transactionId) external view returns (
        address to,
        uint256 value,
        bytes memory data,
        bool executed,
        uint256 confirmations,
        bool canExecute
    ) {
        Transaction storage txn = transactions[transactionId];
        to = txn.to;
        value = txn.value;
        data = txn.data;
        executed = txn.executed;
        confirmations = txn.confirmations;
        canExecute = !executed && confirmations >= required;
    }
    
    modifier onlyActionKey() {
        bytes32 senderKey = keccak256(abi.encodePacked(msg.sender));
        uint256[] memory purposes = keyManager.getKeyPurposes(senderKey);
        
        bool hasActionKey = false;
        for (uint256 i = 0; i < purposes.length; i++) {
            if (purposes[i] == 2) { // ACTION_KEY
                hasActionKey = true;
                break;
            }
        }
        require(hasActionKey, "Sender does not have action key");
        _;
    }
}
```

### Identity-Based Access Control
```solidity
// Access control system using ERC-734 identity management
contract IdentityAccessControl {
    IERC734 public identityRegistry;
    
    struct AccessPolicy {
        uint256 requiredPurpose;
        uint256 minKeyType;
        bool active;
        string description;
    }
    
    mapping(bytes32 => AccessPolicy) public accessPolicies;
    mapping(address => bytes32) public userIdentities;
    mapping(bytes32 => bool) public authorizedIdentities;
    
    event AccessPolicyCreated(bytes32 indexed policyId, uint256 requiredPurpose, string description);
    event IdentityAuthorized(bytes32 indexed identityId, address indexed user);
    event AccessGranted(address indexed user, bytes32 indexed policyId);
    event AccessDenied(address indexed user, bytes32 indexed policyId, string reason);
    
    constructor(address _identityRegistry) {
        identityRegistry = IERC734(_identityRegistry);
    }
    
    function createAccessPolicy(
        bytes32 policyId,
        uint256 requiredPurpose,
        uint256 minKeyType,
        string memory description
    ) external onlyAdmin {
        accessPolicies[policyId] = AccessPolicy({
            requiredPurpose: requiredPurpose,
            minKeyType: minKeyType,
            active: true,
            description: description
        });
        
        emit AccessPolicyCreated(policyId, requiredPurpose, description);
    }
    
    function authorizeIdentity(bytes32 identityId, address user) external onlyAdmin {
        authorizedIdentities[identityId] = true;
        userIdentities[user] = identityId;
        
        emit IdentityAuthorized(identityId, user);
    }
    
    function checkAccess(address user, bytes32 policyId) external returns (bool) {
        AccessPolicy memory policy = accessPolicies[policyId];
        require(policy.active, "Access policy not active");
        
        bytes32 identityId = userIdentities[user];
        if (identityId == bytes32(0)) {
            emit AccessDenied(user, policyId, "No identity registered");
            return false;
        }
        
        if (!authorizedIdentities[identityId]) {
            emit AccessDenied(user, policyId, "Identity not authorized");
            return false;
        }
        
        // Check if user has required key purpose
        bytes32 userKey = keccak256(abi.encodePacked(user));
        uint256[] memory purposes = identityRegistry.getKeyPurposes(userKey);
        
        bool hasPurpose = false;
        for (uint256 i = 0; i < purposes.length; i++) {
            if (purposes[i] == policy.requiredPurpose) {
                hasPurpose = true;
                break;
            }
        }
        
        if (!hasPurpose) {
            emit AccessDenied(user, policyId, "Insufficient key purpose");
            return false;
        }
        
        // Check key type if specified
        if (policy.minKeyType > 0) {
            (, uint256 keyType, ) = identityRegistry.getKey(userKey);
            if (keyType < policy.minKeyType) {
                emit AccessDenied(user, policyId, "Insufficient key type");
                return false;
            }
        }
        
        emit AccessGranted(user, policyId);
        return true;
    }
    
    function getUserIdentityInfo(address user) external view returns (
        bytes32 identityId,
        bool authorized,
        uint256[] memory keyPurposes,
        uint256 keyType
    ) {
        identityId = userIdentities[user];
        authorized = authorizedIdentities[identityId];
        
        if (identityId != bytes32(0)) {
            bytes32 userKey = keccak256(abi.encodePacked(user));
            keyPurposes = identityRegistry.getKeyPurposes(userKey);
            (, keyType, ) = identityRegistry.getKey(userKey);
        }
    }
    
    modifier onlyAdmin() {
        // Implementation would check for admin role
        _;
    }
}
```

### Decentralized Identity Provider
```solidity
// Decentralized Identity Provider for issuing claims
contract DecentralizedIdentityProvider {
    IERC735 public identityContract;
    
    event ClaimIssued(address indexed identity, bytes32 indexed claimId, bytes32 claimTopic);
    
    constructor(address _identityContract) {
        identityContract = IERC735(_identityContract);
    }
    
    function issueClaim(
        address targetIdentity,
        bytes32 claimId,
        uint256 claimTopic,
        uint256 scheme,
        address issuer,
        bytes memory signature,
        bytes memory data,
        string memory uri
    ) external onlyTrustedIssuer {
        // Ensure the issuer is a trusted issuer for the claimTopic
        require(identityContract.isTrustedIssuer(issuer, claimTopic), "Issuer not trusted for this topic");
        
        identityContract.addClaim(
            claimId,
            claimTopic,
            scheme,
            issuer,
            signature,
            data,
            uri
        );
        emit ClaimIssued(targetIdentity, claimId, claimTopic);
    }
    
    function revokeClaim(address targetIdentity, bytes32 claimId) external onlyTrustedIssuer {
        identityContract.removeClaim(claimId);
        emit ClaimRevoked(targetIdentity, claimId);
    }
    
    function getClaim(address targetIdentity, bytes32 claimId) external view returns (
        uint256 claimTopic,
        uint256 scheme,
        address issuer,
        bytes memory signature,
        bytes memory data,
        string memory uri
    ) {
        (claimTopic, scheme, issuer, signature, data, uri) = identityContract.getClaim(claimId);
    }
    
    modifier onlyTrustedIssuer() {
        // Check if msg.sender is a trusted issuer
        _;
    }
}
```

## Security Considerations

### Key Management Security
- **Secure Key Storage**: Advise users on secure storage of private keys.
- **Key Rotation**: Implement key rotation mechanisms to mitigate compromised keys.
- **Multi-Factor Authentication**: Encourage MFA for sensitive operations.
- **Hardware Security Modules**: Recommend HSMs for high-value keys.

### Access Control
- **Purpose Enforcement**: Strictly enforce key purposes for authorized operations.
- **Role-Based Access**: Implement role-based access control for administrative functions.
- **Least Privilege**: Grant only necessary permissions to keys and roles.

### Multi-Signature Security
- **Threshold Control**: Configure appropriate multi-signature thresholds.
- **Disaster Recovery**: Implement multi-sig procedures for disaster recovery.
- **Audit Trails**: Maintain comprehensive audit trails for multi-sig operations.

## Best Practices

### Decentralized Identity Best Practices
- **Self-Sovereign Identity**: Empower users with control over their identity data.
- **Privacy by Design**: Incorporate privacy-preserving features from concept to deployment.
- **Interoperability**: Adhere to open standards (ERC-734, ERC-735, W3C DID) for broad compatibility.
- **User Experience**: Design intuitive interfaces for identity management.

### Implementation Guidelines
- **Modular Design**: Build identities with modular components for flexibility.
- **Upgradeable Contracts**: Design for upgradeability to adapt to evolving standards.
- **Comprehensive Testing**: Rigorous testing for all identity functionalities and edge cases.
- **Security Audits**: Conduct regular security audits on identity contracts and systems.

## Related Documentation

- [ERC-735 Interface](ierc735.md)
- [Identity Registry Facet](../facets/identity-registry-facet.md)
- [Trusted Issuers Registry Facet](../facets/trusted-issuers-registry-facet.md)
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)
- [Developer Guides: Performance Optimization](../../developer-guides/performance-optimization.md)

## Standards Compliance

- **ERC-734**: Key Manager Standard
- **ERC-735**: Claim Holder Standard
- **ERC-165**: Interface detection support