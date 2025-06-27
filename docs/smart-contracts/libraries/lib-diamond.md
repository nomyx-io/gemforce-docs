# LibDiamond Library

## Overview

[`LibDiamond`](/Users/sschepis/Development/gem-base/contracts/libraries/LibDiamond.sol) is the core implementation library for the EIP-2535 Diamond Standard. This library provides all the essential functionality for managing upgradeable smart contracts using the Diamond pattern, including facet management, function selector routing, ownership control, and timelock-based upgrade proposals.

## Key Features

- **Diamond Storage Management**: Implements the diamond storage pattern for upgradeable contracts
- **Facet Management**: Add, replace, and remove contract facets dynamically
- **Function Selector Routing**: Maps function selectors to their corresponding facet addresses
- **Ownership Control**: Manages contract ownership with proper access controls
- **Timelock Upgrades**: Implements secure upgrade proposals with configurable timelock periods
- **ERC-165 Support**: Interface detection for supported standards

## Core Data Structures

### DiamondStorage

The main storage structure that holds all diamond-related data:

```solidity
struct DiamondStorage {
    mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
    mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
    address[] facetAddresses;
    mapping(bytes4 => bool) supportedInterfaces;
    address contractOwner;
    uint256 upgradeTimelock;
    UpgradeProposal upgradeProposal;
}
```

### FacetAddressAndPosition

Maps function selectors to their facet addresses and positions:

```solidity
struct FacetAddressAndPosition {
    address facetAddress;
    uint96 functionSelectorPosition;
}
```

### UpgradeProposal

Stores timelock-based upgrade proposals:

```solidity
struct UpgradeProposal {
    IDiamondCut.FacetCut[] diamondCut;
    address initAddress;
    bytes initCalldata;
    uint256 proposalTime;
    bool exists;
}
```

## Core Functions

### Storage Access

#### `diamondStorage()`
```solidity
function diamondStorage() internal pure returns (DiamondStorage storage ds)
```
Returns the diamond storage struct using assembly for gas efficiency.

### Ownership Management

#### `setContractOwner(address _newOwner)`
```solidity
function setContractOwner(address _newOwner) internal
```
Sets a new contract owner and emits the `OwnershipTransferred` event.

#### `contractOwner()`
```solidity
function contractOwner() internal view returns (address contractOwner_)
```
Returns the current contract owner address.

#### `enforceIsContractOwner()`
```solidity
function enforceIsContractOwner() internal view
```
Reverts if the caller is not the contract owner.

### Diamond Cut Operations

#### `diamondCut()`
```solidity
function diamondCut(
    IDiamondCut.FacetCut[] memory _diamondCut,
    address _init,
    bytes memory _calldata
) internal
```
Executes diamond cut operations to add, replace, or remove functions.

**Parameters:**
- `_diamondCut`: Array of facet cuts to execute
- `_init`: Address of initialization contract (optional)
- `_calldata`: Initialization function call data (optional)

### Facet Management

#### `addFunctions()`
```solidity
function addFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal
```
Adds new functions to the diamond from a specific facet.

#### `replaceFunctions()`
```solidity
function replaceFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal
```
Replaces existing functions with new implementations from a facet.

#### `removeFunctions()`
```solidity
function removeFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal
```
Removes functions from the diamond.

### Timelock Upgrade System

#### `initializeUpgradeTimelock()`
```solidity
function initializeUpgradeTimelock(uint256 _timelock) internal
```
Initializes the upgrade timelock system with a specified delay period.

#### `proposeDiamondCut()`
```solidity
function proposeDiamondCut(
    IDiamondCut.FacetCut[] memory _diamondCut,
    address _init,
    bytes memory _calldata
) internal
```
Proposes a diamond cut to be executed after the timelock period.

#### `executeDiamondCut()`
```solidity
function executeDiamondCut() internal
```
Executes a previously proposed diamond cut after the timelock period has elapsed.

#### `cancelDiamondCut()`
```solidity
function cancelDiamondCut() internal
```
Cancels a pending diamond cut proposal.

## Security Features

### Access Control
- **Owner-only operations**: Critical functions require contract owner privileges
- **Timelock protection**: Upgrades require a waiting period before execution
- **Proposal validation**: Ensures only valid upgrade proposals can be executed

### Safety Checks
- **Contract code validation**: Ensures facets have actual contract code
- **Function selector conflicts**: Prevents duplicate function selectors
- **Immutable function protection**: Prevents removal of core diamond functions

### Gas Optimization
- **Assembly usage**: Direct storage slot access for gas efficiency
- **Efficient data structures**: Optimized mappings and arrays for minimal gas costs
- **Batch operations**: Process multiple facet cuts in a single transaction

## Constants

```solidity
bytes32 constant DIAMOND_STORAGE_POSITION = keccak256("diamond.standard.diamond.storage");
uint256 constant DEFAULT_UPGRADE_TIMELOCK = 2 days;
```

## Events

The library emits events defined in the IDiamondCut interface:
- `DiamondCut`: Emitted when a diamond cut is executed
- `DiamondCutProposed`: Emitted when an upgrade is proposed
- `DiamondCutCancelled`: Emitted when an upgrade proposal is cancelled

## Usage Examples

### Basic Diamond Cut
```solidity
// Add a new facet
IDiamondCut.FacetCut[] memory cuts = new IDiamondCut.FacetCut[](1);
cuts[0] = IDiamondCut.FacetCut({
    facetAddress: newFacetAddress,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: selectors
});

LibDiamond.diamondCut(cuts, address(0), "");
```

### Timelock Upgrade Process
```solidity
// 1. Initialize timelock (done once)
LibDiamond.initializeUpgradeTimelock(7 days);

// 2. Propose upgrade
LibDiamond.proposeDiamondCut(cuts, initAddress, initCalldata);

// 3. Wait for timelock period...

// 4. Execute upgrade
LibDiamond.executeDiamondCut();
```

## Integration Points

- **Diamond Proxy**: Used by the main Diamond contract for all upgrade operations
- **Diamond Cut Facet**: Provides external interface for diamond cut operations
- **Diamond Loupe Facet**: Uses storage structures for introspection functions
- **Ownership Facet**: Integrates with ownership management functions

## Best Practices

1. **Initialize Timelock**: Always initialize upgrade timelock for production deployments
2. **Validate Facets**: Ensure facet contracts are properly tested before adding
3. **Batch Operations**: Group related function changes into single diamond cuts
4. **Monitor Proposals**: Track upgrade proposals and their execution status
5. **Emergency Procedures**: Have procedures for cancelling problematic upgrades

## Security Considerations

- **Owner Key Security**: Protect the contract owner private key with multi-sig or hardware wallets
- **Timelock Duration**: Choose appropriate timelock periods based on security requirements
- **Facet Validation**: Thoroughly audit facet contracts before deployment
- **Upgrade Testing**: Test all upgrades on testnets before mainnet deployment
- **Emergency Response**: Have procedures for handling upgrade emergencies

## Related Documentation

- [Diamond Standard Overview](../diamond.md)
- [IDiamond Interface](../interfaces/idiamond.md)
- [Diamond Factory Library](diamond-factory-lib.md)
- [EIP-2535 Diamond Standard](../../eips/EIP-DRAFT-Diamond-Enhanced-Marketplace.md)