# Diamond Contract

## Overview

The [`Diamond.sol`](/Users/sschepis/Development/gem-base/contracts/Diamond.sol) contract is the core proxy contract implementing the EIP-2535 Diamond Standard. It serves as the main entry point for all diamond-based contracts in the Gemforce system, providing upgradeable functionality through modular facets.

## Contract Details

- **Contract Name**: `Diamond`
- **Inheritance**: `Initializable`, `IERC173`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 EIP-2535 Diamond Standard Implementation
- Modular architecture through facets
- Upgradeable functionality without changing contract address
- Gas-efficient proxy pattern with delegatecall

### 🔹 Multi-Interface Support
- ERC165 (Interface Detection)
- ERC173 (Contract Ownership)
- ERC721 (Non-Fungible Token)
- ERC721Metadata (Token Metadata)
- ERC721Enumerable (Token Enumeration)

### 🔹 Upgrade Timelock Security
- Built-in upgrade timelock mechanism
- Prevents immediate malicious upgrades
- Configurable timelock duration

## Core Functions

### Initialization

#### `initialize()`
```solidity
function initialize(
    address _owner, 
    DiamondSettings memory params,
    IDiamondCut.FacetCut[] memory _facets,
    address diamondInit,
    bytes calldata _calldata
) public initializer
```

**Purpose**: Initializes the Diamond contract with owner, settings, and initial facets.

**Parameters**:
- `_owner`: The initial owner of the contract
- `params`: Diamond settings including name and symbol
- `_facets`: Array of initial facets to add during deployment
- `diamondInit`: Address of the initialization contract
- `_calldata`: Initialization calldata to execute

**Key Operations**:
1. Sets up supported interfaces (ERC165, ERC173, ERC721, etc.)
2. Performs diamond cut to add initial facets
3. Sets contract owner
4. Configures diamond metadata (name, symbol)
5. Initializes upgrade timelock with default duration

### Ownership Management

#### `transferOwnership()`
```solidity
function transferOwnership(address _newOwner) external override
```

**Purpose**: Transfers contract ownership to a new address.

**Access Control**: Only current owner

**Parameters**:
- `_newOwner`: Address of the new owner

#### `owner()`
```solidity
function owner() external override view returns (address owner_)
```

**Purpose**: Returns the current contract owner.

**Returns**: Current owner address

### Utility Functions

#### `diamondAddress()`
```solidity
function diamondAddress() external view returns (address)
```

**Purpose**: Returns this contract's address.

**Returns**: The diamond contract address

### Proxy Functionality

#### `fallback()`
```solidity
fallback() external payable
```

**Purpose**: Core proxy functionality that routes function calls to appropriate facets.

**Process**:
1. Retrieves diamond storage
2. Looks up facet address for the called function selector
3. Executes function using `delegatecall`
4. Returns result or reverts with error

#### `receive()`
```solidity
receive() external payable
```

**Purpose**: Allows the contract to receive ETH directly.

## Architecture Integration

### Diamond Storage Pattern

The Diamond contract uses the diamond storage pattern to avoid storage collisions:

```solidity
LibDiamond.DiamondStorage storage ds;
bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
assembly {
    ds.slot := position
}
```

### Facet Routing

Function calls are routed through the fallback function:

1. **Function Selector Lookup**: `msg.sig` is used to find the corresponding facet
2. **Delegatecall Execution**: Function is executed in the context of the diamond
3. **Return Handling**: Results are returned to the caller

## Security Considerations

### Access Control
- Owner-only functions protected by `LibDiamond.enforceIsContractOwner()`
- Upgrade timelock prevents immediate malicious upgrades
- Interface validation ensures proper ERC compliance

### Upgrade Safety
- Timelock mechanism for upgrades
- Facet validation during diamond cuts
- Storage collision prevention through diamond storage pattern

### Reentrancy Protection
- Inherits reentrancy protection from facets
- Delegatecall pattern maintains security context

## Gas Optimization

### Efficient Proxy Pattern
- Single storage slot lookup for facet routing
- Assembly-optimized delegatecall implementation
- Minimal overhead for function calls

### Storage Efficiency
- Diamond storage pattern prevents storage collisions
- Packed structs in diamond settings
- Efficient interface registration

## Integration Examples

### Basic Diamond Deployment

```solidity
// Deploy diamond with initial facets
IDiamondCut.FacetCut[] memory cuts = new IDiamondCut.FacetCut[](3);

// Add DiamondCutFacet
cuts[0] = IDiamondCut.FacetCut({
    facetAddress: diamondCutFacet,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: diamondCutSelectors
});

// Add DiamondLoupeFacet
cuts[1] = IDiamondCut.FacetCut({
    facetAddress: diamondLoupeFacet,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: diamondLoupeSelectors
});

// Add OwnershipFacet
cuts[2] = IDiamondCut.FacetCut({
    facetAddress: ownershipFacet,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: ownershipSelectors
});

// Initialize diamond
diamond.initialize(
    owner,
    DiamondSettings({name: "MyDiamond", symbol: "MD"}),
    cuts,
    diamondInit,
    initCalldata
);
```

### Adding New Facets

```solidity
// Add marketplace functionality
IDiamondCut.FacetCut[] memory cuts = new IDiamondCut.FacetCut[](1);
cuts[0] = IDiamondCut.FacetCut({
    facetAddress: marketplaceFacet,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: marketplaceSelectors
});

IDiamondCut(diamond).diamondCut(cuts, address(0), "");
```

## Events

The Diamond contract itself doesn't emit events directly, but inherits events from:
- `LibDiamond` for diamond cuts and ownership changes
- Individual facets for their specific functionality

## Error Handling

### Common Errors

- `"Diamond: Function does not exist"`: Called function selector not found in any facet
- `"LibDiamond: Must be contract owner"`: Unauthorized access to owner-only functions
- `"LibDiamond: Upgrade timelock not expired"`: Attempted upgrade before timelock expiration

## Testing Considerations

### Unit Tests
- Test initialization with various facet combinations
- Verify ownership transfer functionality
- Test fallback function routing
- Validate interface support

### Integration Tests
- Test with real facet implementations
- Verify upgrade procedures
- Test complex multi-facet interactions

## Related Documentation

- [DiamondFactory](diamond-factory.md) - Factory for diamond deployment
- [DiamondCutFacet](facets/diamond-cut-facet.md) - Diamond upgrade functionality
- [DiamondLoupeFacet](facets/diamond-loupe-facet.md) - Diamond introspection
- [LibDiamond](libraries/lib-diamond.md) - Diamond utility library

## External References

- [EIP-2535: Diamonds, Multi-Facet Proxy](https://eips.ethereum.org/EIPS/eip-2535)
- [Diamond Standard Documentation](https://eip2535diamonds.substack.com/)
- [OpenZeppelin Initializable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable)

---

*This documentation covers the core Diamond contract. For specific functionality, refer to the individual facet documentation.*