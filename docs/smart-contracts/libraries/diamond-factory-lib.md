# DiamondFactoryLib Library

## Overview

The [`DiamondFactoryLib`](../../smart-contracts/libraries/diamond-factory-lib.md) library provides core utilities for creating and managing Diamond proxy contracts using the factory pattern. This library implements the Diamond Factory Standard, enabling deterministic deployment of Diamond contracts with configurable facet sets and initialization parameters.

## Key Features

- **Deterministic Deployment**: Uses CREATE2 for predictable contract addresses
- **Facet Set Management**: Pre-configured facet combinations for different use cases
- **Diamond Creation**: Factory pattern for deploying Diamond proxy contracts
- **Initialization Support**: Automated Diamond initialization with custom parameters
- **Registry Management**: Tracking and management of deployed Diamonds
- **Flexible Configuration**: Support for custom facet combinations and settings

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.6;

library DiamondFactoryLib {
    // Storage management
    function diamondFactoryStorage() internal pure returns (DiamondFactoryStorage storage);
    
    // Facet set management
    function _addFacetSet(DiamondFactoryStorage storage self, string memory setName, IDiamondCut.FacetCut[] memory facets) internal;
    function _getFacets(DiamondFactoryStorage storage self, string memory setName) internal view returns (IDiamondCut.FacetCut[] memory);
    function _setFacet(DiamondFactoryStorage storage self, string memory setName, uint256 idx, IDiamondCut.FacetCut memory facet) internal;
    
    // Diamond deployment
    function _getDiamondAddress(DiamondFactoryStorage storage, address factoryAddress, string memory symbol, bytes memory creationCode) internal view returns (address);
    function create(...) internal returns (address payable);
    function createFromSet(...) internal returns (address payable);
    
    // Registry management
    function add(DiamondFactoryStorage storage self, string memory symbol, address payable diamondAddress) internal;
    function remove(DiamondFactoryStorage storage self, string memory symbol) internal;
}
```

## Data Structures

### ContractData Struct
```solidity
struct ContractData {
    mapping(string => address) diamondAddresses;        // Symbol to Diamond address mapping
    string[] diamondSymbols;                            // Array of all Diamond symbols
    mapping(string => IDiamondCut.FacetCut[]) facetsToAdd;  // Named facet sets
    string[] facetSets;                                 // Array of facet set names
    string defaultFacetSet;                             // Default facet set name
    bytes diamondBytecode;                              // Diamond contract bytecode
}
```

**Purpose**: Core data structure containing all factory state and configuration.

**Components**:
- **diamondAddresses**: Maps Diamond symbols to deployed contract addresses
- **diamondSymbols**: Maintains list of all deployed Diamond symbols
- **facetsToAdd**: Named collections of facets for different Diamond types
- **facetSets**: List of available facet set names
- **defaultFacetSet**: Default facet configuration for new Diamonds
- **diamondBytecode**: Compiled bytecode for Diamond proxy deployment

### DiamondFactoryStorage Struct
```solidity
struct DiamondFactoryStorage {
    ContractData contractData;
}
```

**Purpose**: Diamond storage wrapper for factory data.

**Storage Position**: 
```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nextblock.bitgem.app.DiamondFactoryStorage.storage");
```

### IDiamondElement Interface
```solidity
interface IDiamondElement {
    function initialize(
        address owner,
        DiamondSettings memory settings,
        IDiamondCut.FacetCut[] calldata facets,
        address diamondInit,
        bytes calldata initData
    ) external payable;
}
```

**Purpose**: Interface for Diamond initialization after deployment.

## Core Functions

### Storage Management

#### `diamondFactoryStorage()`
```solidity
function diamondFactoryStorage() internal pure returns (DiamondFactoryStorage storage ds)
```

**Purpose**: Access Diamond storage for factory data using assembly.

**Implementation**:
```solidity
function diamondFactoryStorage() internal pure returns (DiamondFactoryStorage storage ds) {
    bytes32 position = DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
}
```

**Returns**: Storage reference to factory data

**Usage**: All factory functions use this to access persistent storage.

### Facet Set Management

#### `_addFacetSet()`
```solidity
function _addFacetSet(
    DiamondFactoryStorage storage self, 
    string memory setName, 
    IDiamondCut.FacetCut[] memory facets
) internal
```

**Purpose**: Add a named collection of facets for Diamond deployment.

**Parameters**:
- `self`: Storage reference to factory data
- `setName`: Unique name for the facet set
- `facets`: Array of facet cuts to include in the set

**Process**:
1. Add each facet to the named set
2. Register set name if not already present
3. Enable reuse of facet configurations

**Example Usage**:
```solidity
// Create marketplace facet set
IDiamondCut.FacetCut[] memory marketplaceFacets = new IDiamondCut.FacetCut[](3);
marketplaceFacets[0] = IDiamondCut.FacetCut({
    facetAddress: marketplaceFacetAddress,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: marketplaceSelectors
});
marketplaceFacets[1] = IDiamondCut.FacetCut({
    facetAddress: erc721FacetAddress,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: erc721Selectors
});
marketplaceFacets[2] = IDiamondCut.FacetCut({
    facetAddress: ownershipFacetAddress,
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: ownershipSelectors
});

DiamondFactoryLib._addFacetSet(storage, "marketplace", marketplaceFacets);
```

#### `_getFacets()`
```solidity
function _getFacets(
    DiamondFactoryStorage storage self, 
    string memory setName
) internal view returns (IDiamondCut.FacetCut[] memory)
```

**Purpose**: Retrieve all facets for a named set.

**Parameters**:
- `self`: Storage reference to factory data
- `setName`: Name of the facet set to retrieve

**Returns**: Array of facet cuts for the specified set

**Example Usage**:
```solidity
// Get marketplace facets for deployment
IDiamondCut.FacetCut[] memory facets = DiamondFactoryLib._getFacets(storage, "marketplace");
```

#### `_setFacet()`
```solidity
function _setFacet(
    DiamondFactoryStorage storage self, 
    string memory setName, 
    uint256 idx, 
    IDiamondCut.FacetCut memory facet
) internal
```

**Purpose**: Update a specific facet within a named set.

**Parameters**:
- `self`: Storage reference to attribute contract
- `setName`: Name of the facet set
- `idx`: Index of the facet to update
- `facet`: New facet cut data

**Use Cases**:
- Upgrading facet addresses
- Modifying function selectors
- Changing facet cut actions

**Example Usage**:
```solidity
// Update marketplace facet to new version
IDiamondCut.FacetCut memory newMarketplaceFacet = IDiamondCut.FacetCut({
    facetAddress: newMarketplaceFacetAddress,
    action: IDiamondCut.FacetCutAction.Replace,
    functionSelectors: marketplaceSelectors
});

DiamondFactoryLib._setFacet(storage, "marketplace", 0, newMarketplaceFacet);
```

### Diamond Deployment

#### `_getDiamondAddress()`
```solidity
function _getDiamondAddress(
    DiamondFactoryStorage storage,
    address factoryAddress,
    string memory symbol,
    bytes memory creationCode
) internal view returns (address)
```

**Purpose**: Calculate the deterministic address for a Diamond before deployment.

**Parameters**:
- `factoryAddress`: Address of the factory contract
- `symbol`: Unique symbol for the Diamond
- `creationCode`: Bytecode for Diamond deployment

**Implementation**:
```solidity
return Create2.computeAddress(
    keccak256(abi.encodePacked(factoryAddress, symbol)),
    keccak256(creationCode)
);
```

**Returns**: Predicted Diamond contract address

**Benefits**:
- Enables address prediction before deployment
- Supports cross-chain address consistency
- Allows for address-dependent logic

#### `create()`
```solidity
function create(
    DiamondFactoryStorage storage self,
    address diamondAddress,
    DiamondSettings memory params,
    address diamondInit,
    bytes calldata _calldata,
    bytes memory _creationCode,
    IDiamondCut.FacetCut[] memory facets
) internal returns (address payable _diamondAddress)
```

**Purpose**: Deploy a new Diamond with custom facet configuration.

**Parameters**:
- `self`: Storage reference to factory data
- `diamondAddress`: Predicted Diamond address (for salt generation)
- `params`: Diamond configuration settings
- `diamondInit`: Address of initialization contract
- `_calldata`: Initialization function call data
- `_creationCode`: Diamond contract bytecode
- `facets`: Custom facet configuration

**Process**:
1. Deploy Diamond using CREATE2 with deterministic salt
2. Verify deployment success
3. Register Diamond in factory storage
4. Initialize Diamond with provided parameters and facets

**Example Usage**:
```solidity
// Deploy custom Diamond with specific facets
DiamondSettings memory settings = DiamondSettings({
    name: "Custom NFT Collection",
    symbol: "CUSTOM",
    baseURI: "https://api.example.com/metadata/",
    owner: msg.sender
});

IDiamondCut.FacetCut[] memory customFacets = new IDiamondCut.FacetCut[](2);
customFacets[0] = marketplaceFacet;
customFacets[1] = royaltyFacet;

address diamondAddress = DiamondFactoryLib.create(
    storage,
    predictedAddress,
    settings,
    initContract,
    initCalldata,
    diamondBytecode,
    customFacets
);
```

#### `createFromSet()`
```solidity
function createFromSet(
    DiamondFactoryStorage storage self,
    address diamondAddress,
    DiamondSettings memory params,
    address diamondInit,
    bytes calldata _calldata,
    bytes memory _creationCode,
    string memory facetSet
) internal returns (address payable _diamondAddress)
```

**Purpose**: Deploy a new Diamond using a pre-configured facet set.

**Parameters**:
- `self`: Storage reference to factory data
- `diamondAddress`: Predicted Diamond address
- `params`: Diamond configuration settings
- `diamondInit`: Address of initialization contract
- `_calldata`: Initialization function call data
- `_creationCode`: Diamond contract bytecode
- `facetSet`: Name of pre-configured facet set

**Process**:
1. Deploy Diamond using CREATE2
2. Retrieve facets from named set
3. Initialize Diamond with set facets

**Example Usage**:
```solidity
// Deploy Diamond using marketplace facet set
address diamondAddress = DiamondFactoryLib.createFromSet(
    storage,
    predictedAddress,
    settings,
    initContract,
    initCalldata,
    diamondBytecode,
    "marketplace"
);
```

### Registry Management

#### `add()`
```solidity
function add(
    DiamondFactoryStorage storage self,
    string memory symbol,
    address payable diamondAddress
) internal
```

**Purpose**: Register an existing Diamond with the factory.

**Parameters**:
- `self`: Storage reference to factory data
- `symbol`: Unique symbol for the Diamond
- `diamondAddress`: Address of existing Diamond

**Use Cases**:
- Importing externally deployed Diamonds
- Migrating from other factory contracts
- Manual registration of special Diamonds

#### `remove()`
```solidity
function remove(
    DiamondFactoryStorage storage self,
    string memory symbol
) internal
```

**Purpose**: Unregister a Diamond from the factory.

**Parameters**:
- `self`: Storage reference to factory data
- `symbol`: Symbol of Diamond to remove

**Process**:
1. Clear address mapping
2. Remove symbol from array (swap with last element)
3. Update array length

**Note**: Does not destroy the Diamond contract, only removes factory tracking.

## Integration Examples

### NFT Collection Factory
```solidity
// Factory for creating NFT collection Diamonds
contract NFTCollectionFactory {
    using DiamondFactoryLib for DiamondFactoryLib.DiamondFactoryStorage;
    
    struct CollectionTemplate {
        string name;
        string description;
        string facetSet;
        uint256 deploymentFee;
        bool active;
    }
    
    mapping(string => CollectionTemplate) public templates;
    mapping(address => string[]) public userCollections;
    
    event TemplateCreated(string indexed templateName, string facetSet);
    event CollectionDeployed(address indexed diamond, string symbol, address indexed creator);
    
    function createTemplate(
        string memory templateName,
        string memory description,
        IDiamondCut.FacetCut[] memory facets,
        uint256 deploymentFee
    ) external onlyOwner {
        // Add facet set for template
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        DiamondFactoryLib._addFacetSet(ds, templateName, facets);
        
        templates[templateName] = CollectionTemplate({
            name: templateName,
            description: description,
            facetSet: templateName,
            deploymentFee: deploymentFee,
            active: true
        });
        
        emit TemplateCreated(templateName, templateName);
    }
    
    function deployCollection(
        string memory templateName,
        string memory collectionName,
        string memory symbol,
        string memory baseURI,
        uint256 maxSupply
    ) external payable returns (address diamond) {
        CollectionTemplate memory template = templates[templateName];
        require(template.active, "Template not active");
        require(msg.value >= template.deploymentFee, "Insufficient fee");
        
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        
        // Prepare Diamond settings
        DiamondSettings memory settings = DiamondSettings({
            name: collectionName,
            symbol: symbol,
            baseURI: baseURI,
            maxSupply: maxSupply,
            owner: msg.sender
        });
        
        // Get predicted address
        bytes memory creationCode = getDiamondBytecode();
        address predictedAddress = DiamondFactoryLib._getDiamondAddress(
            ds,
            address(this),
            symbol,
            creationCode
        );
        
        // Deploy Diamond
        diamond = DiamondFactoryLib.createFromSet(
            ds,
            predictedAddress,
            settings,
            getInitContract(),
            getInitCalldata(settings),
            creationCode,
            template.facetSet
        );
        
        // Track user collections
        userCollections[msg.sender].push(symbol);
        
        emit CollectionDeployed(diamond, symbol, msg.sender);
    }
    
    function getUserCollections(address user) external view returns (string[] memory) {
        return userCollections[user];
    }
    
    function getCollectionAddress(string memory symbol) external view returns (address) {
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        return ds.contractData.diamondAddresses[symbol];
    }
    
    function getDiamondBytecode() internal pure returns (bytes memory) {
        // Return bytecode of Diamond contract
        return type(Diamond).creationCode;
    }
    
    function getInitContract() internal pure returns (address) {
        // Return address of initialization contract
        return address(0);
    }
    
    function getInitCalldata(DiamondSettings memory settings) internal pure returns (bytes memory) {
        // Return encoded calldata for initialization
        return abi.encodeCall(IDiamondElement.initialize, (
            settings.owner,
            settings,
            new IDiamondCut.FacetCut[](0), // No initial facets via init
            address(0),
            ""
        ));
    }
    
    modifier onlyOwner() {
        // Placeholder for access control
        _;
    }
}
```

### Gaming Platform Factory
```solidity
// Factory for creating Gaming Platform Diamonds
contract GamingPlatformFactory {
    using DiamondFactoryLib for DiamondFactoryLib.DiamondFactoryStorage;
    
    struct PlatformTemplate {
        string name;
        string description;
        string facetSet;
        uint256 setupFee;
        bool active;
    }
    
    mapping(string => PlatformTemplate) public templates;
    
    event PlatformDeployed(address indexed diamond, string name, address indexed creator);
    
    function createTemplate(
        string memory templateName,
        string memory description,
        IDiamondCut.FacetCut[] memory facets,
        uint256 setupFee
    ) external onlyOwner {
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        DiamondFactoryLib._addFacetSet(ds, templateName, facets);
        
        templates[templateName] = PlatformTemplate({
            name: templateName,
            description: description,
            facetSet: templateName,
            setupFee: setupFee,
            active: true
        });
    }
    
    function deployPlatform(
        string memory templateName,
        string memory platformName,
        string memory symbol,
        address adminAddress
    ) external payable returns (address diamond) {
        PlatformTemplate memory template = templates[templateName];
        require(template.active, "Template not active");
        require(msg.value >= template.setupFee, "Insufficient setup fee");
        
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        
        // Prepare Diamond settings
        DiamondSettings memory settings = DiamondSettings({
            name: platformName,
            symbol: symbol,
            baseURI: "", // Not needed for platforms
            maxSupply: 0, // Not applicable
            owner: adminAddress
        });
        
        // Get predicted address
        bytes memory creationCode = getDiamondBytecode();
        address predictedAddress = DiamondFactoryLib._getDiamondAddress(
            ds,
            address(this),
            symbol,
            creationCode
        );
        
        // Deploy Diamond
        diamond = DiamondFactoryLib.createFromSet(
            ds,
            predictedAddress,
            settings,
            getInitContract(),
            getInitCalldata(settings),
            creationCode,
            template.facetSet
        );
        
        emit PlatformDeployed(diamond, platformName, msg.sender);
    }
    
    function getDiamondBytecode() internal pure returns (bytes memory) {
        // Return bytecode of Diamond contract
        return type(Diamond).creationCode;
    }
    
    function getInitContract() internal pure returns (address) {
        // Return address of initialization contract
        return address(0);
    }
    
    function getInitCalldata(DiamondSettings memory settings) internal pure returns (bytes memory) {
        // Return encoded calldata for initialization
        return abi.encodeCall(IDiamondElement.initialize, (
            settings.owner,
            settings,
            new IDiamondCut.FacetCut[](0),
            address(0),
            ""
        ));
    }
    
    modifier onlyOwner() {
        // Placeholder for access control
        _;
    }
}
```

### DeFi Protocol Factory
```solidity
// Factory for creating DeFi Protocol Diamonds
contract DeFiProtocolFactory {
    using DiamondFactoryLib for DiamondFactoryLib.DiamondFactoryStorage;
    
    struct ProtocolTemplate {
        string name;
        string description;
        string facetSet;
        uint256 deployCost;
        bool active;
    }
    
    mapping(string => ProtocolTemplate) public templates;
    
    event ProtocolDeployed(address indexed diamond, string name, address indexed creator);
    
    function createTemplate(
        string memory templateName,
        string memory description,
        IDiamondCut.FacetCut[] memory facets,
        uint256 deployCost
    ) external onlyOwner {
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        DiamondFactoryLib._addFacetSet(ds, templateName, facets);
        
        templates[templateName] = ProtocolTemplate({
            name: templateName,
            description: description,
            facetSet: templateName,
            deployCost: deployCost,
            active: true
        });
    }
    
    function deployProtocol(
        string memory templateName,
        string memory protocolName,
        string memory symbol,
        address protocolAdmin
    ) external payable returns (address diamond) {
        ProtocolTemplate memory template = templates[templateName];
        require(template.active, "Template not active");
        require(msg.value >= template.deployCost, "Insufficient deployment cost");
        
        DiamondFactoryLib.DiamondFactoryStorage storage ds = DiamondFactoryLib.diamondFactoryStorage();
        
        // Prepare Diamond settings
        DiamondSettings memory settings = DiamondSettings({
            name: protocolName,
            symbol: symbol,
            baseURI: "",
            maxSupply: 0,
            owner: protocolAdmin
        });
        
        // Get predicted address
        bytes memory creationCode = getDiamondBytecode();
        address predictedAddress = DiamondFactoryLib._getDiamondAddress(
            ds,
            address(this),
            symbol,
            creationCode
        );
        
        // Deploy Diamond
        diamond = DiamondFactoryLib.createFromSet(
            ds,
            predictedAddress,
            settings,
            getInitContract(),
            getInitCalldata(settings),
            creationCode,
            template.facetSet
        );
        
        emit ProtocolDeployed(diamond, protocolName, msg.sender);
    }
    
    function getDiamondBytecode() internal pure returns (bytes memory) {
        // Return bytecode of Diamond contract
        return type(Diamond).creationCode;
    }
    
    function getInitContract() internal pure returns (address) {
        // Return address of initialization contract
        return address(0);
    }
    
    function getInitCalldata(DiamondSettings memory settings) internal pure returns (bytes memory) {
        // Return encoded calldata for initialization
        return abi.encodeCall(IDiamondElement.initialize, (
            settings.owner,
            settings,
            new IDiamondCut.FacetCut[](0),
            address(0),
            ""
        ));
    }
    
    modifier onlyOwner() {
        // Placeholder for access control
        _;
    }
}
```

## Security Considerations

### CREATE2 Security
- **Salt Management**: Ensure unique salts for deterministic addresses
- **Collision Prevention**: Avoid salt collisions for predictable addresses
- **Replay Protection**: Implement replay protection for factory calls

### Access Control
- **Template Creation**: Restrict who can create and modify templates
- **Deployment Permissions**: Control who can deploy new Diamonds
- **Owner Assignment**: Validate initial owner assignment

### Deployment Safety
- **Initialization Atomicity**: Ensure initialization is atomic
- **Error Handling**: Robust error handling for deployment failures
- **Gas Limits**: Manage gas limits for complex deployments

## Gas Optimization

### Storage Efficiency
- **Packed Structs**: Use packed structs for `ContractData` and `DiamondSettings`.
- **Minimal Mappings**: Minimize mappings to reduce storage footprint.

### Deployment Efficiency
- **CREATE2**: Leverage CREATE2 for gas-efficient deployments.
- **Batch Deployment**: `deployDiamonds` function for multiple deployments.

## Error Handling

### Common Errors
- `DiamondFactoryLib: `Invalid symbol`: Duplicate symbol in factory.
- `DiamondFactoryLib: `Facet set not found`: Attempting to deploy from non-existent facet set.
- `DiamondFactoryLib: `Initialization failed`: Diamond initialization failed.
- `DiamondFactoryLib: `Deployment failed`: CREATE2 deployment failed.

## Best Practices

### Factory Design
- **Single Responsibility**: Factory focuses solely on Diamond deployment.
- **Modularity**: Use libraries for common functions (e.g., CREATE2).
- **Extensibility**: Design for easy addition of new templates and features.

### Deployment Process
- **Pre-compute Addresses**: Use `predictDiamondAddress` for off-chain calculations.
- **Monitor Deployments**: Track deployments and their status.
- **Post-deployment Validation**: Verify deployed Diamond's configuration and functionality.

## Related Documentation

- [Diamond Standard Overview](../../smart-contracts/diamond.md)
- [Diamond Factory Standard](../../eips/EIP-DRAFT-Diamond-Factory-Standard.md)
- [Diamond Factory](../../smart-contracts/diamond-factory.md)
- [SDK & Libraries: Deploy Utilities](../../sdk-libraries/deploy.md)
- [Deployment Guides: Multi-Network Deployment](../../deployment-guides/multi-network-deployment.md)