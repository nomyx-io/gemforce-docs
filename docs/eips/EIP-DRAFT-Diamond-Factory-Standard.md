# EIP-DRAFT: Diamond Factory Standard for Upgradeable Contract Deployment

## Simple Summary

A standardized factory pattern for deploying and managing Diamond Standard (EIP-2535) contracts with configurable facets, initialization parameters, and upgrade timelock mechanisms.

## Abstract

This EIP proposes a standard factory interface for creating Diamond Standard contracts that enables:
- Standardized Diamond deployment with configurable facet sets
- Template-based Diamond creation with predefined configurations
- Upgrade timelock initialization for security
- Event-driven deployment tracking
- Integration with existing Diamond tooling and infrastructure

## Motivation

While the Diamond Standard (EIP-2535) provides excellent modularity and upgradeability, there is no standardized way to deploy Diamond contracts with consistent configurations. This leads to fragmented deployment patterns and makes it difficult to build tooling that works across different Diamond implementations. A standardized factory pattern addresses these issues by providing a consistent deployment interface.

## Specification

### Core Interface

```solidity
interface IDiamondFactory {
    struct DiamondSettings {
        string name;
        string symbol;
        address owner;
        // Additional settings can be extended
    }
    
    struct FacetDeployment {
        address facetAddress;
        bytes4[] functionSelectors;
        bytes initCalldata;
    }
    
    struct DiamondTemplate {
        string name;
        string description;
        FacetDeployment[] facets;
        DiamondSettings defaultSettings;
        bool active;
    }
    
    // Events
    event DiamondCreated(address indexed diamond, address indexed owner, DiamondSettings settings, uint256 templateId);
    event DiamondTemplateCreated(uint256 indexed templateId, string name, string description);
    event DiamondTemplateUpdated(uint256 indexed templateId, string name, string description, bool active);
    event FacetDeployed(address indexed facet, string name, bytes4[] selectors);
    
    // Core Functions
    function createDiamond(DiamondSettings memory settings, FacetDeployment[] memory facets, address diamondInit, bytes memory initCalldata) external returns (address diamond);
    function createDiamondFromTemplate(uint256 templateId, DiamondSettings memory settings, bytes memory customInitCalldata) external returns (address diamond);
    
    // Template Management
    function createDiamondTemplate(string memory name, string memory description, FacetDeployment[] memory facets, DiamondSettings memory defaultSettings) external returns (uint256 templateId);
    function updateDiamondTemplate(uint256 templateId, string memory name, string memory description, FacetDeployment[] memory facets, DiamondSettings memory defaultSettings, bool active) external;
    function getDiamondTemplate(uint256 templateId) external view returns (DiamondTemplate memory);
    function getAllTemplateIds() external view returns (uint256[] memory);
    
    // Deployment Tracking
    function getDiamondsCreatedBy(address creator) external view returns (address[] memory);
    function getDiamondInfo(address diamond) external view returns (DiamondSettings memory settings, uint256 templateId, address creator, uint256 createdAt);
    function isDiamondFromFactory(address diamond) external view returns (bool);
    
    // Facet Management
    function deployFacet(string memory name, bytes memory bytecode, bytes4[] memory selectors) external returns (address facet);
    function getFacetInfo(address facet) external view returns (string memory name, bytes4[] memory selectors, address deployer, uint256 deployedAt);
    function getAllFacets() external view returns (address[] memory);
}
```

### Key Features

#### 1. Standardized Diamond Deployment
- **Consistent Interface**: Uniform deployment interface across implementations
- **Configurable Facets**: Deploy Diamonds with custom facet configurations
- **Initialization Support**: Handle complex initialization scenarios

#### 2. Template System
- **Predefined Configurations**: Create reusable Diamond templates
- **Template Management**: Add, update, and manage Diamond templates
- **Template-based Deployment**: Deploy Diamonds from predefined templates

#### 3. Deployment Tracking
- **Creator Tracking**: Track which addresses created which Diamonds
- **Deployment History**: Maintain history of all deployments
- **Diamond Verification**: Verify if a Diamond was created by the factory

#### 4. Facet Management
- **Facet Deployment**: Deploy and register facets
- **Facet Registry**: Maintain registry of available facets
- **Selector Tracking**: Track function selectors for each facet

### Template-Based Deployment

```solidity
// Create a template for NFT marketplaces
FacetDeployment[] memory facets = new FacetDeployment[](3);
facets[0] = FacetDeployment(marketplaceFacet, marketplaceSelectors, "");
facets[1] = FacetDeployment(erc721Facet, erc721Selectors, "");
facets[2] = FacetDeployment(ownershipFacet, ownershipSelectors, "");

DiamondSettings memory defaultSettings = DiamondSettings("NFT Marketplace", "NFTM", address(0));

uint256 templateId = factory.createDiamondTemplate("NFT Marketplace", "Standard NFT marketplace with ERC721 support", facets, defaultSettings);

// Deploy from template
DiamondSettings memory customSettings = DiamondSettings("My Marketplace", "MYM", msg.sender);
address diamond = factory.createDiamondFromTemplate(templateId, customSettings, "");
```

### Upgrade Timelock Integration

```solidity
contract DiamondFactory is IDiamondFactory {
    uint256 public constant DEFAULT_UPGRADE_TIMELOCK = 48 hours;
    
    function createDiamond(DiamondSettings memory settings, FacetDeployment[] memory facets, address diamondInit, bytes memory initCalldata) external returns (address diamond) {
        // Deploy Diamond
        diamond = address(new Diamond());
        
        // Initialize with facets
        IDiamondCut.FacetCut[] memory cuts = _prepareFacetCuts(facets);
        IDiamond(diamond).initialize(settings.owner, settings, cuts, diamondInit, initCalldata);
        
        // Set upgrade timelock
        IUpgradeTimelock(diamond).initializeUpgradeTimelock(DEFAULT_UPGRADE_TIMELOCK);
        
        // Track deployment
        _trackDeployment(diamond, settings, 0, msg.sender);
        
        emit DiamondCreated(diamond, settings.owner, settings, 0);
        return diamond;
    }
}
```

## Rationale

### Factory Pattern Benefits
The factory pattern provides several advantages:
- **Consistency**: Ensures all Diamonds are deployed with proper initialization
- **Security**: Enforces security best practices like upgrade timelocks
- **Tracking**: Enables tracking and verification of deployments
- **Tooling**: Enables better tooling and infrastructure development

### Template System
Templates reduce deployment complexity and ensure consistent configurations for common use cases while still allowing customization.

### Upgrade Timelock Integration
Automatic upgrade timelock initialization ensures that deployed Diamonds have proper security measures in place from the start.

## Implementation Details

### Diamond Deployment

```solidity
function _prepareFacetCuts(FacetDeployment[] memory facets) internal pure returns (IDiamondCut.FacetCut[] memory cuts) {
    cuts = new IDiamondCut.FacetCut[](facets.length);
    
    for (uint256 i = 0; i < facets.length; i++) {
        cuts[i] = IDiamondCut.FacetCut({
            facetAddress: facets[i].facetAddress,
            action: IDiamondCut.FacetCutAction.Add,
            functionSelectors: facets[i].functionSelectors
        });
    }
}
```

### Template Storage

```solidity
mapping(uint256 => DiamondTemplate) private templates;
mapping(address => uint256[]) private creatorToTemplates;
uint256 private nextTemplateId = 1;

function createDiamondTemplate(string memory name, string memory description, FacetDeployment[] memory facets, DiamondSettings memory defaultSettings) external returns (uint256 templateId) {
    templateId = nextTemplateId++;
    
    templates[templateId] = DiamondTemplate({
        name: name,
        description: description,
        facets: facets,
        defaultSettings: defaultSettings,
        active: true
    });
    
    creatorToTemplates[msg.sender].push(templateId);
    emit DiamondTemplateCreated(templateId, name, description);
}
```

### Security Considerations

1. **Access Control**: Proper access control for template management and facet deployment
2. **Initialization Security**: Secure handling of initialization parameters and calldata
3. **Upgrade Timelock**: Automatic enforcement of upgrade timelocks
4. **Facet Validation**: Validation of facet addresses and selectors
5. **Template Integrity**: Ensure template configurations are valid and secure

### Gas Optimization

- Efficient storage patterns for templates and deployment tracking
- Batch operations for multiple facet deployments
- Optimized Diamond initialization process
- Minimal redundant storage

## Backwards Compatibility

This standard is designed to work with existing Diamond Standard implementations and does not modify the core Diamond functionality.

## Integration Examples

### With Development Tools

```solidity
// Hardhat deployment script
const factory = await ethers.getContractAt("IDiamondFactory", factoryAddress);
const diamond = await factory.createDiamondFromTemplate(templateId, settings, initData);
```

### With Frontend Applications

```javascript
// Web3 integration
const factory = new web3.eth.Contract(IDiamondFactory.abi, factoryAddress);
const result = await factory.methods.createDiamondFromTemplate(templateId, settings, initData).send({from: account});
const diamondAddress = result.events.DiamondCreated.returnValues.diamond;
```

## Test Cases

Comprehensive test cases should cover:
- Diamond deployment with various facet configurations
- Template creation and management
- Template-based deployment
- Deployment tracking and verification
- Access control mechanisms
- Edge cases and error conditions

## Reference Implementation

The reference implementation includes:
- [`DiamondFactory.sol`](../contracts/DiamondFactory.sol) - Core factory contract
- [`IDiamondFactory.sol`](../contracts/interfaces/IDiamondFactory.sol) - Interface definition
- [`DiamondFactoryLib.sol`](../contracts/libraries/DiamondFactoryLib.sol) - Supporting library functions

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).