# Diamond Loupe Facet

The Diamond Loupe Facet provides introspection capabilities for diamond contracts as defined in EIP-2535. It allows querying the current state of a diamond, including which facets are installed and what functions they provide.

## Overview

The Diamond Loupe Facet is an essential component of the Diamond Standard that provides:

- **Facet Discovery**: Query all facets currently part of the diamond
- **Function Mapping**: Discover which facet implements each function
- **Interface Detection**: Support for EIP-165 interface detection
- **Diamond Inspection**: Complete visibility into diamond composition

## Key Features

### Introspection Capabilities
- **List All Facets**: Get addresses of all facets in the diamond
- **Function Selectors**: Query function selectors for each facet
- **Facet Mapping**: Find which facet implements a specific function
- **Interface Support**: Check if diamond supports specific interfaces

### Standards Compliance
- **EIP-2535**: Full Diamond Standard compliance
- **EIP-165**: Interface detection support
- **Read-Only Operations**: All functions are view/pure (no state changes)

## Interface

```solidity
interface IDiamondLoupe {
    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }
    
    function facets() external view returns (Facet[] memory facets_);
    function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory facetFunctionSelectors_);
    function facetAddresses() external view returns (address[] memory facetAddresses_);
    function facetAddress(bytes4 _functionSelector) external view returns (address facetAddress_);
    function supportsInterface(bytes4 _interfaceId) external view returns (bool);
}
```

## Core Functions

### facets()
Returns all facets and their function selectors.

**Returns:**
- `Facet[]`: Array of Facet structs containing address and selectors

**Usage:**
```solidity
IDiamondLoupe.Facet[] memory allFacets = IDiamondLoupe(diamond).facets();
for (uint i = 0; i < allFacets.length; i++) {
    console.log("Facet:", allFacets[i].facetAddress);
    console.log("Functions:", allFacets[i].functionSelectors.length);
}
```

### facetFunctionSelectors()
Returns function selectors for a specific facet.

**Parameters:**
- `_facet`: Address of the facet to query

**Returns:**
- `bytes4[]`: Array of function selectors implemented by the facet

### facetAddresses()
Returns addresses of all facets in the diamond.

**Returns:**
- `address[]`: Array of facet addresses

### facetAddress()
Returns the facet address that implements a specific function.

**Parameters:**
- `_functionSelector`: Function selector to query

**Returns:**
- `address`: Address of facet implementing the function

### supportsInterface()
Checks if the diamond supports a specific interface (EIP-165).

**Parameters:**
- `_interfaceId`: Interface identifier to check

**Returns:**
- `bool`: True if interface is supported

## Usage Examples

### Complete Diamond Inspection

```solidity
contract DiamondInspector {
    function inspectDiamond(address diamond) external view returns (
        uint256 facetCount,
        uint256 totalFunctions,
        address[] memory facetAddresses
    ) {
        IDiamondLoupe loupe = IDiamondLoupe(diamond);
        
        // Get all facets
        IDiamondLoupe.Facet[] memory facets = loupe.facets();
        facetCount = facets.length;
        facetAddresses = new address[](facetCount);
        
        // Count total functions and collect addresses
        for (uint i = 0; i < facets.length; i++) {
            totalFunctions += facets[i].functionSelectors.length;
            facetAddresses[i] = facets[i].facetAddress;
        }
    }
}
```

### Function Discovery

```solidity
contract FunctionDiscovery {
    function findFunctionImplementation(
        address diamond,
        string memory functionSignature
    ) external view returns (address facetAddress, bool exists) {
        bytes4 selector = bytes4(keccak256(bytes(functionSignature)));
        facetAddress = IDiamondLoupe(diamond).facetAddress(selector);
        exists = facetAddress != address(0);
    }
}
```

## Security Considerations

### Read-Only Nature
- **No State Changes**: All functions are view/pure
- **Gas Efficient**: Minimal gas cost for queries
- **Safe to Call**: No risk of state modification

### Information Disclosure
- **Public Information**: All diamond structure is publicly visible
- **Design Consideration**: Consider privacy implications
- **Transparency**: Enables full transparency of contract capabilities

## Best Practices

### Integration Guidelines
1. **Cache Results**: Cache facet information when possible
2. **Batch Queries**: Use `facets()` for complete information
3. **Interface Checks**: Always verify interface support
4. **Error Handling**: Handle cases where functions don't exist

### Development Workflow
1. **Testing**: Use loupe for testing diamond composition
2. **Debugging**: Inspect diamond state during development
3. **Documentation**: Generate documentation from loupe data
4. **Monitoring**: Monitor diamond changes in production

## Related Documentation

- [Diamond Standard Overview](../diamond.md)
- [Diamond Cut Facet](diamond-cut-facet.md)
- [IDiamond Interface](../interfaces/idiamond.md)
- [Developer Guides: Development Environment Setup](../../developer-guides/development-environment-setup.md)

## Standards Compliance

- **EIP-2535**: Diamond Standard implementation
- **EIP-165**: Interface detection support
- **Gas Optimization**: Efficient storage and retrieval patterns