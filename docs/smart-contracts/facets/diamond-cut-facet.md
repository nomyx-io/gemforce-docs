# Diamond Cut Facet

The Diamond Cut Facet implements the core diamond upgrade functionality as defined in EIP-2535. This facet enables adding, replacing, and removing functions from a diamond contract, making it upgradeable while maintaining state.

## Overview

The Diamond Cut Facet is a fundamental component of the Diamond Standard that provides:

- **Function Management**: Add, replace, or remove functions from the diamond
- **Facet Management**: Manage which facets are part of the diamond
- **Upgrade Safety**: Ensures upgrades maintain contract integrity
- **Access Control**: Restricts upgrade operations to authorized users

## Key Features

### Function Operations
- **Add Functions**: Add new functions from facets to the diamond
- **Replace Functions**: Update existing function implementations
- **Remove Functions**: Remove functions that are no longer needed

### Safety Mechanisms
- **Selector Validation**: Ensures function selectors don't conflict
- **Facet Address Validation**: Validates facet contract addresses
- **Initialization Support**: Supports initialization calls during upgrades

## Interface

```solidity
interface IDiamondCut {
    enum FacetCutAction {Add, Replace, Remove}
    
    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }
    
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}
```

## Core Functions

### diamondCut()
Performs diamond upgrade operations by adding, replacing, or removing functions.

**Parameters:**
- `_diamondCut`: Array of FacetCut structs defining the changes
- `_init`: Address of contract to call for initialization (optional)
- `_calldata`: Data to pass to initialization contract (optional)

**Access Control:**
- Only contract owner can perform diamond cuts
- Validates all operations before execution

**Events:**
- Emits `DiamondCut` event for each successful upgrade

## Usage Examples

### Adding a New Facet

```solidity
// Define the facet cut
IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
cut[0] = IDiamondCut.FacetCut({
    facetAddress: newFacetAddress,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: selectors
});

// Perform the diamond cut
IDiamondCut(diamond).diamondCut(cut, address(0), "");
```

### Replacing Function Implementation

```solidity
// Replace existing functions with new implementation
IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
cut[0] = IDiamondCut.FacetCut({
    facetAddress: updatedFacetAddress,
    action: IDiamondCut.FacetCutAction.Replace,
    functionSelectors: selectorsToReplace
});

IDiamondCut(diamond).diamondCut(cut, address(0), "");
```

### Removing Functions

```solidity
// Remove functions from diamond
IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
cut[0] = IDiamondCut.FacetCut({
    facetAddress: address(0), // Address 0 for removal
    action: IDiamondCut.FacetCutAction.Remove,
    functionSelectors: selectorsToRemove
});

IDiamondCut(diamond).diamondCut(cut, address(0), "");
```

## Security Considerations

### Access Control
- **Owner Only**: Only the diamond owner can perform cuts
- **Multi-sig Recommended**: Use multi-signature wallets for production
- **Timelock Integration**: Consider timelock contracts for upgrade delays

### Validation Checks
- **Selector Conflicts**: Prevents duplicate function selectors
- **Address Validation**: Ensures facet addresses are valid contracts
- **Function Existence**: Validates functions exist before replacement/removal

### Upgrade Safety
- **State Preservation**: Upgrades preserve existing contract state
- **Initialization Calls**: Supports safe state migration during upgrades
- **Rollback Planning**: Plan for potential rollback scenarios

## Integration with Other Facets

### Diamond Loupe Facet
Works with Diamond Loupe Facet to provide introspection capabilities:
- Query current facets and functions
- Validate upgrade operations
- Monitor diamond composition

### Ownership Facet
Integrates with ownership controls:
- Restricts cuts to authorized users
- Supports ownership transfer
- Enables governance integration

## Best Practices

### Upgrade Planning
1. **Test Thoroughly**: Test all upgrades on testnets first
2. **Incremental Changes**: Make small, incremental upgrades
3. **Documentation**: Document all changes and their impacts
4. **Monitoring**: Monitor contract behavior after upgrades

### Development Workflow
1. **Facet Development**: Develop and test new facets independently
2. **Integration Testing**: Test facet integration with existing diamond
3. **Upgrade Execution**: Execute upgrades during low-activity periods
4. **Post-Upgrade Validation**: Validate functionality after upgrades

## Error Handling

### Common Errors
- `DiamondCut: No selectors in facet to cut`: Empty selector array
- `DiamondCut: Add facet can't be address(0)`: Invalid facet address for add
- `DiamondCut: Can't replace function that doesn't exist`: Function not found
- `DiamondCut: Can't remove function that doesn't exist`: Function not found

### Troubleshooting
1. **Validate Selectors**: Ensure function selectors are correct
2. **Check Addresses**: Verify facet contract addresses
3. **Review Permissions**: Confirm caller has upgrade permissions
4. **Test Operations**: Test upgrade operations on development networks

## Related Documentation

- [Diamond Standard Overview](../diamond.md)
- [Diamond Loupe Facet](diamond-loupe-facet.md)
- [Ownership Facet](ownership-facet.md)
- [Developer Guides: Development Environment Setup](../../developer-guides/development-environment-setup.md)
- [Deployment guides: Multi-Network Deployment](../../deployment-guides/multi-network-deployment.md)

## Standards Compliance

- **EIP-2535**: Diamond Standard implementation
- **EIP-165**: Interface detection support
- **Access Control**: OpenZeppelin-compatible ownership patterns