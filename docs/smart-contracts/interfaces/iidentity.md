# IIdentity Interface

The IIdentity interface defines the standard for identity management contracts in the Gemforce ecosystem. It provides a comprehensive framework for managing digital identities, key management, claims verification, and identity attestations based on ERC734 and ERC735 standards.

## Overview

IIdentity provides:

- **Key Management**: Secure management of cryptographic keys
- **Claims System**: Verifiable claims and attestations
- **Identity Verification**: Multi-level identity verification
- **Access Control**: Role-based access control for identity operations
- **Interoperability**: Standard interface for identity interactions

## Key Features

### Key Management
- **Multi-Key Support**: Support for multiple key types and purposes
- **Key Rotation**: Secure key rotation and revocation
- **Hierarchical Keys**: Support for key hierarchies and delegation
- **Recovery Mechanisms**: Account recovery through trusted keys

### Claims and Attestations
- **Verifiable Claims**: Create and verify identity claims
- **Third-Party Attestations**: Support for external attestations
- **Claim Revocation**: Revoke invalid or expired claims
- **Proof Generation**: Generate cryptographic proofs for claims

### Identity Operations
- **Identity Creation**: Create new identity contracts
- **Profile Management**: Manage identity profiles and metadata
- **Verification Levels**: Support multiple verification levels
- **Compliance**: Support for regulatory compliance requirements

## Interface Definition

```solidity
interface IIdentity {
    // Events
    event KeyAdded(
        bytes32 indexed key,
        uint256 indexed purpose,
        uint256 indexed keyType
    );
    
    event KeyRemoved(
        bytes32 indexed key,
        uint256 indexed purpose,
        uint256 indexed keyType
    );
    
    event ClaimAdded(
        bytes32 indexed claimId,
        uint256 indexed topic,
        uint256 scheme,
        address indexed issuer,
        bytes signature,
        bytes data,
        string uri
    );
    
    event ClaimRemoved(
        bytes32 indexed claimId,
        uint256 indexed topic,
        uint256 scheme,
        address indexed issuer
    );
    
    event ClaimChanged(
        bytes32 indexed claimId,
        uint256 indexed topic,
        uint256 scheme,
        address indexed issuer,
        bytes signature,
        bytes data,
        string uri
    );
    
    event IdentityVerified(
        address indexed identity,
        uint256 indexed level,
        address indexed verifier
    );
    
    event IdentityRevoked(
        address indexed identity,
        address indexed revoker,
        string reason
    );
    
    // Enums
    enum KeyPurpose {
        MANAGEMENT,     // 1
        ACTION,         // 2
        CLAIM,          // 3
        ENCRYPTION      // 4
    }
    
    enum KeyType {
        ECDSA,          // 1
        RSA,            // 2
        AES             // 3
    }
    
    enum VerificationLevel {
        NONE,           // 0
        BASIC,          // 1
        ENHANCED,       // 2
        PREMIUM         // 3
    }
    
    // Structs
    struct Key {
        uint256 purpose;
        uint256 keyType;
        bytes32 key;
        bool active;
        uint256 addedAt;
        uint256 revokedAt;
    }
    
    struct Claim {
        uint256 topic;
        uint256 scheme;
        address issuer;
        bytes signature;
        bytes data;
        string uri;
        bool valid;
        uint256 issuedAt;
        uint256 expiresAt;
    }
    
    struct IdentityProfile {
        string name;
        string email;
        string country;
        uint256 verificationLevel;
        bool active;
        uint256 createdAt;
        uint256 updatedAt;
    }
    
    struct VerificationRequest {
        address identity;
        uint256 requestedLevel;
        address verifier;
        bytes evidence;
        uint256 requestedAt;
        uint256 expiresAt;
        bool processed;
    }
    
    // Key Management Functions
    function addKey(
        bytes32 _key,
        uint256 _purpose,
        uint256 _keyType
    ) external returns (bool success);
    
    function removeKey(
        bytes32 _key,
        uint256 _purpose
    ) external returns (bool success);
    
    function getKey(bytes32 _key) 
        external 
        view 
        returns (
            uint256 purpose,
            uint256 keyType,
            bytes32 key,
            bool active
        );
    
    function getKeysByPurpose(uint256 _purpose) 
        external 
        view 
        returns (bytes32[] memory keys);
    
    function keyHasPurpose(bytes32 _key, uint256 _purpose) 
        external 
        view 
        returns (bool exists);
    
    // Claim Management Functions
    function addClaim(
        uint256 _topic,
        uint256 _scheme,
        address _issuer,
        bytes calldata _signature,
        bytes calldata _data,
        string calldata _uri
    ) external returns (bytes32 claimRequestId);
    
    function removeClaim(bytes32 _claimId) 
        external 
        returns (bool success);
    
    function getClaim(bytes32 _claimId) 
        external 
        view 
        returns (
            uint256 topic,
            uint256 scheme,
            address issuer,
            bytes memory signature,
            bytes memory data,
            string memory uri
        );
    
    function getClaimIdsByTopic(uint256 _topic) 
        external 
        view 
        returns (bytes32[] memory claimIds);
    
    function getClaimsByTopic(uint256 _topic) 
        external 
        view 
        returns (Claim[] memory claims);
    
    // Identity Management Functions
    function createIdentity(
        address _owner,
        IdentityProfile calldata _profile
    ) external returns (address identity);
    
    function updateProfile(IdentityProfile calldata _profile) external;
    
    function getProfile() 
        external 
        view 
        returns (IdentityProfile memory profile);
    
    function requestVerification(
        uint256 _level,
        bytes calldata _evidence
    ) external returns (bytes32 requestId);
    
    function processVerification(
        bytes32 _requestId,
        bool _approved,
        string calldata _reason
    ) external;
    
    function getVerificationLevel() 
        external 
        view 
        returns (uint256 level);
    
    function isVerified(uint256 _level) 
        external 
        view 
        returns (bool verified);
    
    // Utility Functions
    function execute(
        address _to,
        uint256 _value,
        bytes calldata _data
    ) external returns (uint256 executionId);
    
    function approve(
        uint256 _id,
        bool _approve
    ) external returns (bool success);
    
    function isValidSignature(
        bytes32 _hash,
        bytes calldata _signature
    ) external view returns (bytes4 magicValue);
    
    // Query Functions
    function owner() external view returns (address);
    
    function isOwner(address _addr) external view returns (bool);
    
    function getExecutionNonce() external view returns (uint256);
    
    function getClaimCount() external view returns (uint256);
    
    function getKeyCount() external view returns (uint256);
    
    function supportsInterface(bytes4 interfaceId) 
        external 
        view 
        returns (bool);
}
```

## Core Functions

### addKey()
Adds a new key to the identity with specified purpose and type.

**Parameters:**
- `_key`: The key identifier (keccak256 hash of the key)
- `_purpose`: The purpose of the key (MANAGEMENT, ACTION, CLAIM, ENCRYPTION)
- `_keyType`: The type of key (ECDSA, RSA, AES)

**Returns:**
- `bool`: Success status

**Usage:**
```solidity
bytes32 keyHash = keccak256(abi.encodePacked(publicKey));
bool success = identity.addKey(
    keyHash,
    uint256(IIdentity.KeyPurpose.MANAGEMENT),
    uint256(IIdentity.KeyType.ECDSA)
);
```

### addClaim()
Adds a new claim to the identity.

**Parameters:**
- `_topic`: The claim topic identifier
- `_scheme`: The signature scheme used
- `_issuer`: The address of the claim issuer
- `_signature`: The claim signature
- `_data`: The claim data
- `_uri`: URI for additional claim information

**Returns:**
- `bytes32`: Claim request ID

### createIdentity()
Creates a new identity contract for a user.

**Parameters:**
- `_owner`: The owner address of the identity
- `_profile`: Initial identity profile information

**Returns:**
- `address`: Address of the created identity contract

### requestVerification()
Requests identity verification at a specific level.

**Parameters:**
- `_level`: Requested verification level
- `_evidence`: Supporting evidence for verification

**Returns:**
- `bytes32`: Verification request ID

## Implementation Example

### Basic Identity Contract

```solidity
contract Identity is IIdentity, ERC165 {
    using ECDSA for bytes32;
    
    // Storage
    mapping(bytes32 => Key) private keys;
    mapping(uint256 => bytes32[]) private keysByPurpose;
    mapping(bytes32 => Claim) private claims;
    mapping(uint256 => bytes32[]) private claimsByTopic;
    
    IdentityProfile private profile;
    address private _owner;
    uint256 private executionNonce;
    uint256 private verificationLevel;
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == _owner, "Not owner");
        _;
    }
    
    modifier onlyManagementKey() {
        require(
            keyHasPurpose(keccak256(abi.encodePacked(msg.sender)), 1),
            "No management key"
        );
        _;
    }
    
    constructor(address owner, IdentityProfile memory _profile) {
        _owner = owner;
        profile = _profile;
        
        // Add owner as management key
        bytes32 ownerKey = keccak256(abi.encodePacked(owner));
        _addKey(ownerKey, 1, 1); // MANAGEMENT, ECDSA
        
        emit IdentityVerified(address(this), 0, address(0));
    }
    
    function addKey(
        bytes32 _key,
        uint256 _purpose,
        uint256 _keyType
    ) external override onlyManagementKey returns (bool success) {
        return _addKey(_key, _purpose, _keyType);
    }
    
    function _addKey(
        bytes32 _key,
        uint256 _purpose,
        uint256 _keyType
    ) internal returns (bool success) {
        require(_key != bytes32(0), "Invalid key");
        require(_purpose >= 1 && _purpose <= 4, "Invalid purpose");
        require(_keyType >= 1 && _keyType <= 3, "Invalid key type");
        
        Key storage key = keys[_key];
        require(!key.active, "Key already exists");
        
        key.purpose = _purpose;
        key.keyType = _keyType;
        key.key = _key;
        key.active = true;
        key.addedAt = block.timestamp;
        
        keysByPurpose[_purpose].push(_key);
        
        emit KeyAdded(_key, _purpose, _keyType);
        return true;
    }
    
    function removeKey(
        bytes32 _key,
        uint256 _purpose
    ) external override onlyManagementKey returns (bool success) {
        Key storage key = keys[_key];
        require(key.active, "Key not found");
        require(key.purpose == _purpose, "Purpose mismatch");
        
        key.active = false;
        key.revokedAt = block.timestamp;
        
        // Remove from purpose array
        _removeFromArray(keysByPurpose[_purpose], _key);
        
        emit KeyRemoved(_key, _purpose, key.keyType);
        return true;
    }
    
    function addClaim(
        uint256 _topic,
        uint256 _scheme,
        address _issuer,
        bytes calldata _signature,
        bytes calldata _data,
        string calldata _uri
    ) external override returns (bytes32 claimRequestId) {
        require(_issuer != address(0), "Invalid issuer");
        
        claimRequestId = keccak256(
            abi.encodePacked(_topic, _scheme, _issuer, _data, block.timestamp)
        );
        
        Claim storage claim = claims[claimRequestId];
        require(claim.issuedAt == 0, "Claim already exists");
        
        claim.topic = _topic;
        claim.scheme = _scheme;
        claim.issuer = _issuer;
        claim.signature = _signature;
        claim.data = _data;
        claim.uri = _uri;
        claim.valid = true;
        claim.issuedAt = block.timestamp;
        claim.expiresAt = block.timestamp + 365 days; // Default 1 year
        
        claimsByTopic[_topic].push(claimRequestId);
        
        emit ClaimAdded(claimRequestId, _topic, _scheme, _issuer, _signature, _data, _uri);
    }
    
    function requestVerification(
        uint256 _level,
        bytes calldata _evidence
    ) external override returns (bytes32 requestId) {
        require(_level > verificationLevel, "Level not higher");
        require(_level <= 3, "Invalid level");
        
        requestId = keccak256(
            abi.encodePacked(address(this), _level, _evidence, block.timestamp)
        );
        
        // Emit event for off-chain processing
        emit IdentityVerified(address(this), _level, msg.sender);
        
        return requestId;
    }
    
    function processVerification(
        bytes32 _requestId,
        bool _approved,
        string calldata _reason
    ) external override {
        // Only authorized verifiers can process
        require(isAuthorizedVerifier(msg.sender), "Not authorized verifier");
        
        if (_approved) {
            // Update verification level based on request
            // Implementation depends on specific verification logic
            verificationLevel = getRequestedLevel(_requestId);
            emit IdentityVerified(address(this), verificationLevel, msg.sender);
        } else {
            emit IdentityRevoked(address(this), msg.sender, _reason);
        }
    }
    
    function execute(
        address _to,
        uint256 _value,
        bytes calldata _data
    ) external override returns (uint256 executionId) {
        require(
            keyHasPurpose(keccak256(abi.encodePacked(msg.sender)), 2) ||
            keyHasPurpose(keccak256(abi.encodePacked(msg.sender)), 1),
            "No execution permission"
        );
        
        executionId = executionNonce++;
        
        (bool success, ) = _to.call{value: _value}(_data);
        require(success, "Execution failed");
        
        return executionId;
    }
    
    function isValidSignature(
        bytes32 _hash,
        bytes calldata _signature
    ) external view override returns (bytes4 magicValue) {
        address signer = _hash.recover(_signature);
        bytes32 signerKey = keccak256(abi.encodePacked(signer));
        
        if (keyHasPurpose(signerKey, 1) || keyHasPurpose(signerKey, 2)) {
            return 0x1626ba7e; // EIP-1271 magic value
        }
        
        return 0xffffffff;
    }
    
    // View functions
    function getKey(bytes32 _key) 
        external 
        view 
        override 
        returns (
            uint256 purpose,
            uint256 keyType,
            bytes32 key,
            bool active
        ) 
    {
        Key storage k = keys[_key];
        return (k.purpose, k.keyType, k.key, k.active);
    }
    
    function keyHasPurpose(bytes32 _key, uint256 _purpose) 
        public 
        view 
        override 
        returns (bool exists) 
    {
        Key storage key = keys[_key];
        return key.active && key.purpose == _purpose;
    }
    
    function getClaim(bytes32 _claimId) 
        external 
        view 
        override 
        returns (
            uint256 topic,
            uint256 scheme,
            address issuer,
            bytes memory signature,
            bytes memory data,
            string memory uri
        ) 
    {
        Claim storage claim = claims[_claimId];
        return (
            claim.topic,
            claim.scheme,
            claim.issuer,
            claim.signature,
            claim.data,
            claim.uri
        );
    }
    
    function getProfile() 
        external 
        view 
        override 
        returns (IdentityProfile memory) 
    {
        return profile;
    }
    
    function getVerificationLevel() 
        external 
        view 
        override 
        returns (uint256 level) 
    {
        return verificationLevel;
    }
    
    function isVerified(uint256 _level) 
        external 
        view 
        override 
        returns (bool verified) 
    {
        return verificationLevel >= _level;
    }
    
    function owner() external view override returns (address) {
        return _owner;
    }
    
    function isOwner(address _addr) external view override returns (bool) {
        return _addr == _owner;
    }
    
    function supportsInterface(bytes4 interfaceId) 
        public 
        view 
        virtual 
        override 
        returns (bool) 
    {
        return
            interfaceId == type(IIdentity).interfaceId ||
            super.supportsInterface(interfaceId);
    }
}
```

### Advanced Identity Features

```solidity
contract AdvancedIdentity is Identity {
    // Multi-signature support
    mapping(bytes32 => mapping(address => bool)) public approvals;
    mapping(bytes32 => uint256) public approvalCount;
    mapping(bytes32 => uint256) public requiredApprovals;
    
    // Delegation support
    mapping(address => mapping(uint256 => bool)) public delegations;
    mapping(address => uint256) public delegationExpiry;
    
    // Recovery mechanisms
    mapping(address => bool) public recoveryKeys;
    uint256 public recoveryThreshold;
    mapping(bytes32 => uint256) public recoveryVotes;
    
    function addMultiSigKey(
        bytes32 _key,
        uint256 _purpose,
        uint256 _keyType,
        uint256 _requiredApprovals
    ) external onlyManagementKey {
        _addKey(_key, _purpose, _keyType);
        requiredApprovals[_key] = _requiredApprovals;
    }
    
    function executeWithApproval(
        address _to,
        uint256 _value,
        bytes calldata _data,
        bytes32 _executionId
    ) external returns (bool success) {
        bytes32 signerKey = keccak256(abi.encodePacked(msg.sender));
        require(keyHasPurpose(signerKey, 2), "No action key");
        
        approvals[_executionId][msg.sender] = true;
        approvalCount[_executionId]++;
        
        if (approvalCount[_executionId] >= requiredApprovals[signerKey]) {
            (success, ) = _to.call{value: _value}(_data);
            delete approvals[_executionId];
            delete approvalCount[_executionId];
        }
        
        return success;
    }
    
    function delegateKey(
        address _delegate,
        uint256 _purpose,
        uint256 _duration
    ) external onlyManagementKey {
        delegations[_delegate][_purpose] = true;
        delegationExpiry[_delegate] = block.timestamp + _duration;
    }
    
    function revokeDelegation(address _delegate) external onlyManagementKey {
        delete delegations[_delegate][1];
        delete delegations[_delegate][2];
        delete delegationExpiry[_delegate];
    }
    
    function addRecoveryKey(address _recoveryKey) external onlyOwner {
        recoveryKeys[_recoveryKey] = true;
    }
    
    function initiateRecovery(address _newOwner) external {
        require(recoveryKeys[msg.sender], "Not recovery key");
        
        bytes32 recoveryId = keccak256(abi.encodePacked(_newOwner, block.timestamp));
        recoveryVotes[recoveryId]++;
        
        if (recoveryVotes[recoveryId] >= recoveryThreshold) {
            _owner = _newOwner;
            
            // Add new owner as management key
            bytes32 newOwnerKey = keccak256(abi.encodePacked(_newOwner));
            _addKey(newOwnerKey, 1, 1);
        }
    }
}
```

## Verification Levels

### Level 0: None
- **Requirements**: Basic identity creation
- **Capabilities**: Basic key management and claims
- **Restrictions**: Limited transaction amounts

### Level 1: Basic
- **Requirements**: Email verification, basic KYC
- **Capabilities**: Standard operations, moderate transaction limits
- **Verification**: Automated verification process

### Level 2: Enhanced
- **Requirements**: Document verification, address proof
- **Capabilities**: Higher transaction limits, advanced features
- **Verification**: Manual review process

### Level 3: Premium
- **Requirements**: Full KYC/AML compliance, biometric verification
- **Capabilities**: Unlimited operations, institutional features
- **Verification**: Comprehensive compliance review

## Security Considerations

### Key Management Security
- **Key Rotation**: Regular key rotation policies
- **Multi-Signature**: Multi-signature requirements for critical operations
- **Recovery Mechanisms**: Secure account recovery procedures
- **Key Validation**: Validate key formats and cryptographic properties

### Claim Verification
- **Issuer Validation**: Verify claim issuer authenticity
- **Signature Verification**: Validate claim signatures
- **Expiration Handling**: Handle claim expiration properly
- **Revocation Checks**: Check for claim revocations

### Access Control
- **Permission Validation**: Validate permissions for all operations
- **Delegation Security**: Secure delegation mechanisms
- **Audit Trails**: Comprehensive audit logging
- **Rate Limiting**: Prevent abuse through rate limiting

## Best Practices

### Identity Design
1. **Minimal Data**: Store minimal personal data on-chain
2. **Privacy Protection**: Implement privacy-preserving mechanisms
3. **Interoperability**: Design for cross-platform compatibility
4. **Upgradability**: Plan for identity contract upgrades

### Key Management
1. **Key Diversity**: Use different keys for different purposes
2. **Secure Storage**: Implement secure key storage mechanisms
3. **Regular Rotation**: Establish key rotation schedules
4. **Backup Procedures**: Implement secure backup procedures

### Verification Process
1. **Graduated Verification**: Implement graduated verification levels
2. **Evidence Validation**: Thoroughly validate verification evidence
3. **Compliance**: Ensure regulatory compliance
4. **User Experience**: Balance security with user experience

## Integration Examples

### Frontend Integration

```typescript
class IdentityService {
    private contract: Contract;
    
    constructor(address: string, provider: Provider) {
        this.contract = new Contract(address, IDENTITY_ABI, provider);
    }
    
    async createIdentity(profile: IdentityProfile): Promise<string> {
        const tx = await this.contract.createIdentity(
            await this.provider.getSigner().getAddress(),
            profile
        );
        const receipt = await tx.wait();
        return receipt.contractAddress;
    }
    
    async addKey(key: string, purpose: KeyPurpose, keyType: KeyType): Promise<void> {
        const keyHash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(key));
        await this.contract.addKey(keyHash, purpose, keyType);
    }
    
    async addClaim(claim: ClaimData): Promise<string> {
        const tx = await this.contract.addClaim(
            claim.topic,
            claim.scheme,
            claim.issuer,
            claim.signature,
            claim.data,
            claim.uri
        );
        const receipt = await tx.wait();
        return receipt.events[0].args.claimRequestId;
    }
    
    async requestVerification(level: number, evidence: string): Promise<string> {
        const tx = await this.contract.requestVerification(
            level,
            ethers.utils.toUtf8Bytes(evidence)
        );
        const receipt = await tx.wait();
        return receipt.events[0].args.requestId;
    }
    
    async getVerificationLevel(): Promise<number> {
        return await this.contract.getVerificationLevel();
    }
}
```

### Backend Integration

```javascript
class IdentityManager {
    constructor(web3, contractAddress) {
        this.web3 = web3;
        this.contract = new web3.eth.Contract(IDENTITY_ABI, contractAddress);
    }
    
    async processVerificationRequest(requestId, approved, reason) {
        const accounts = await this.web3.eth.getAccounts();
        
        return await this.contract.methods
            .processVerification(requestId, approved, reason)
            .send({ from: accounts[0] });
    }
    
    async validateClaim(claimId) {
        const claim = await this.contract.methods.getClaim(claimId).call();
        
        // Validate claim signature
        const messageHash = this.web3.utils.keccak256(claim.data);
        const recoveredAddress = this.web3.eth.accounts.recover(
            messageHash,
            claim.signature
        );
        
        return recoveredAddress.toLowerCase() === claim.issuer.toLowerCase();
    }
    
    async getIdentityProfile(identityAddress) {
        const identityContract = new this.web3.eth.Contract(
            IDENTITY_ABI,
            identityAddress
        );
        
        return await identityContract.methods.getProfile().call();
    }
}
```

## Related Documentation

- [ERC734 Key Manager](./ierc734.md)
- [ERC735 Claim Holder](./ierc735.md)
- [Identity Factory](../identity-factory.md)
- [Identity Registry](../facets/identity-registry-facet.md)
- [Verification Guide](../../developer-guides/automated-testing-setup.md)
- [Key Management Guide](../../developer-guides/development-environment-setup.md)

## Standards Compliance

- **ERC734**: Key Manager standard implementation
- **ERC735**: Claim Holder standard implementation
- **ERC165**: Interface detection standard
- **ERC1271**: Signature validation standard
- **EIP-712**: Typed structured data hashing and signing