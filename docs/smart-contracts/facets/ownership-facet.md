# Ownership Facet

The Ownership Facet provides ownership management functionality for diamond contracts, implementing access control patterns that restrict administrative operations to authorized users.

## Overview

The Ownership Facet is a critical security component that provides:

- **Access Control**: Restrict sensitive operations to contract owner
- **Ownership Transfer**: Safe transfer of contract ownership
- **Administrative Functions**: Foundation for other facet access controls
- **Security Patterns**: Standard ownership verification mechanisms

## Key Features

### Ownership Management
- **Owner Verification**: Check if address is contract owner
- **Ownership Transfer**: Transfer ownership to new address
- **Renounce Ownership**: Remove ownership (irreversible)
- **Pending Ownership**: Two-step ownership transfer for safety

### Access Control Integration
- **Modifier Support**: Provides `onlyOwner` functionality
- **Facet Integration**: Other facets can use ownership checks
- **Administrative Gates**: Controls access to upgrade functions
- **Emergency Controls**: Owner-only emergency functions

## Interface

```solidity
interface IOwnership {
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    function owner() external view returns (address owner_);
    function transferOwnership(address _newOwner) external;
    function renounceOwnership() external;
}
```

## Core Functions

### owner()
Returns the current owner of the contract.

**Returns:**
- `address`: Current owner address

**Usage:**
```solidity
address currentOwner = IOwnership(diamond).owner();
```

### transferOwnership()
Transfers ownership to a new address.

**Parameters:**
- `_newOwner`: Address of the new owner

**Access Control:**
- Only current owner can call this function

**Events:**
- Emits `OwnershipTransferred` event

**Usage:**
```solidity
IOwnership(diamond).transferOwnership(newOwnerAddress);
```

### renounceOwnership()
Renounces ownership, leaving the contract without an owner.

**Warning:** This is irreversible and will disable all owner-only functions.

**Access Control:**
- Only current owner can call this function

**Events:**
- Emits `OwnershipTransferred` with new owner as `address(0)`

## Implementation Patterns

### Basic Ownership Check

```solidity
contract ExampleFacet {
    modifier onlyOwner() {
        require(LibDiamond.contractOwner() == msg.sender, "Not owner");
        _;
    }
    
    function adminFunction() external onlyOwner {
        // Admin-only logic here
    }
}
```

### Safe Ownership Transfer

```solidity
contract SafeOwnershipFacet {
    address private pendingOwner;
    
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Invalid owner");
        pendingOwner = newOwner;
    }
    
    function acceptOwnership() external {
        require(msg.sender == pendingOwner, "Not pending owner");
        address oldOwner = LibDiamond.contractOwner();
        LibDiamond.setContractOwner(pendingOwner);
        pendingOwner = address(0);
        emit OwnershipTransferred(oldOwner, msg.sender);
    }
}
```

### Multi-Signature Integration

```solidity
contract MultiSigOwnership {
    mapping(address => bool) public isOwner;
    uint256 public requiredSignatures;
    
    struct Transaction {
        address to;
        bytes data;
        uint256 confirmations;
        mapping(address => bool) confirmed;
        bool executed;
    }
    
    mapping(uint256 => Transaction) public transactions;
    uint256 public transactionCount;
    
    modifier onlyMultiSigOwner() {
        require(isOwner[msg.sender], "Not authorized");
        _;
    }
    
    function submitTransaction(address to, bytes memory data) 
        external 
        onlyMultiSigOwner 
        returns (uint256 transactionId) 
    {
        transactionId = transactionCount++;
        Transaction storage txn = transactions[transactionId];
        txn.to = to;
        txn.data = data;
        
        confirmTransaction(transactionId);
    }
    
    function confirmTransaction(uint256 transactionId) public onlyMultiSigOwner {
        Transaction storage txn = transactions[transactionId];
        require(!txn.confirmed[msg.sender], "Already confirmed");
        
        txn.confirmed[msg.sender] = true;
        txn.confirmations++;
        
        if (txn.confirmations >= requiredSignatures) {
            executeTransaction(transactionId);
        }
    }
}
```

## Security Considerations

### Access Control
- **Owner Verification**: Always verify caller is owner
- **Zero Address Check**: Prevent ownership transfer to zero address
- **Reentrancy Protection**: Consider reentrancy in ownership functions
- **Event Emission**: Always emit events for ownership changes

### Ownership Transfer Safety
- **Two-Step Transfer**: Use pending ownership pattern for safety
- **Address Validation**: Validate new owner addresses
- **Confirmation Required**: Require new owner to accept ownership
- **Timelock Integration**: Consider timelock for ownership changes

### Emergency Considerations
- **Recovery Mechanisms**: Plan for owner key loss scenarios
- **Multi-Signature**: Use multi-sig for high-value contracts
- **Governance Integration**: Consider DAO governance for ownership
- **Upgrade Restrictions**: Limit upgrade capabilities appropriately

## Integration with Diamond Standard

### LibDiamond Integration

```solidity
library LibDiamond {
    bytes32 constant DIAMOND_STORAGE_POSITION = keccak256("diamond.standard.diamond.storage");
    
    struct DiamondStorage {
        mapping(bytes4 => bytes32) facets;
        mapping(uint256 => bytes32) selectorSlots;
        uint256 selectorCount;
        mapping(bytes4 => bool) supportedInterfaces;
        address contractOwner;
    }
    
    function diamondStorage() internal pure returns (DiamondStorage storage ds) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly {
            ds.slot := position
        }
    }
    
    function setContractOwner(address _newOwner) internal {
        DiamondStorage storage ds = diamondStorage();
        address previousOwner = ds.contractOwner;
        ds.contractOwner = _newOwner;
        emit OwnershipTransferred(previousOwner, _newOwner);
    }
    
    function contractOwner() internal view returns (address contractOwner_) {
        contractOwner_ = diamondStorage().contractOwner;
    }
    
    function enforceIsContractOwner() internal view {
        require(msg.sender == diamondStorage().contractOwner, "LibDiamond: Must be contract owner");
    }
}
```

### Diamond Cut Integration

```solidity
contract DiamondCutFacet is IDiamondCut {
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external override {
        LibDiamond.enforceIsContractOwner();
        LibDiamond.diamondCut(_diamondCut, _init, _calldata);
    }
}
```

## Best Practices

### Development Guidelines
1. **Always Use Modifiers**: Use `onlyOwner` modifier consistently
2. **Event Emission**: Emit events for all ownership changes
3. **Address Validation**: Validate addresses before ownership transfer
4. **Documentation**: Document all owner-only functions clearly

### Production Deployment
1. **Multi-Signature**: Use multi-sig wallets for production ownership
2. **Timelock Contracts**: Consider timelock for sensitive operations
3. **Governance Integration**: Plan for decentralized governance
4. **Emergency Procedures**: Establish emergency response procedures

### Security Auditing
1. **Access Control Review**: Audit all access control mechanisms
2. **Ownership Transfer Testing**: Test ownership transfer scenarios
3. **Emergency Function Testing**: Test emergency and recovery functions
4. **Multi-Signature Validation**: Validate multi-sig implementations

## Common Patterns

### Role-Based Access Control

```solidity
contract RoleBasedOwnership {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    
    mapping(bytes32 => mapping(address => bool)) private roles;
    
    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "AccessControl: account missing role");
        _;
    }
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return roles[role][account] || (role == ADMIN_ROLE && account == LibDiamond.contractOwner());
    }
    
    function grantRole(bytes32 role, address account) external onlyOwner {
        roles[role][account] = true;
    }
    
    function revokeRole(bytes32 role, address account) external onlyOwner {
        roles[role][account] = false;
    }
}
```

### Pausable Contract

```solidity
contract PausableOwnership {
    bool private paused;
    
    event Paused(address account);
    event Unpaused(address account);
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier whenPaused() {
        require(paused, "Contract is not paused");
        _;
    }
    
    function pause() external onlyOwner whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }
    
    function unpause() external onlyOwner whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
}
```

## Error Handling

### Common Errors
- `"Not owner"`: Caller is not the contract owner
- `"Invalid owner"`: Attempting to set invalid owner address
- `"Already owner"`: Attempting to transfer to current owner
- `"Zero address"`: Attempting to use zero address as owner

### Troubleshooting
1. **Verify Caller**: Ensure calling address is current owner
2. **Check Address**: Validate target addresses are not zero
3. **Transaction Status**: Verify transactions are properly executed
4. **Event Logs**: Check event logs for ownership changes

## Related Documentation

- [Diamond Standard Overview](../diamond.md)
- [Diamond Cut Facet](diamond-cut-facet.md)
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)
- [Security: Overview](../../security/overview.md)
- [Integrator's Guide: Smart Contracts](../../integrator-guide/smart-contracts.md)

## Standards Compliance

- **OpenZeppelin Ownable**: Compatible with OpenZeppelin patterns
- **EIP-173**: Contract Ownership Standard
- **Diamond Standard**: Integrated with EIP-2535
- **Access Control**: Industry-standard access control patterns