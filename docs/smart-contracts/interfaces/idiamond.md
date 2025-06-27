# IDiamond Interface

## Overview

The [`IDiamond.sol`](../../smart-contracts/interfaces/idiamond.md) defines the core interface and data structures for the Diamond Standard implementation within the Gemforce platform. This interface establishes the fundamental contract for diamond proxy functionality and provides essential data structures for diamond configuration and storage.

## Interface Details

- **Interface Name**: `IDiamond`
- **License**: MIT
- **Solidity Version**: ^0.8.6

## Key Features

### 🔹 Diamond Standard Compliance
- Core interface for EIP-2535 Diamond Standard
- Standardized diamond proxy functionality
- Consistent diamond address resolution
- Integration with diamond storage patterns

### 🔹 Configuration Management
- Comprehensive diamond settings structure
- Owner and factory address management
- SVG manager integration
- Metadata storage capabilities

### 🔹 Storage Architecture
- Structured diamond storage layout
- Efficient metadata management
- Extensible configuration system
- Type-safe data structures

## Core Data Structures

### DiamondSettings
```solidity
struct DiamondSettings {
    address owner;        // Diamond contract owner
    address factory;      // Factory contract that created this diamond
    address svgManager;   // SVG template manager contract
    string symbol;        // Diamond symbol identifier
    string name;          // Human-readable diamond name
}
```

**Purpose**: Defines the core configuration parameters for a diamond instance.

**Fields**:
- **owner**: Address with administrative control over the diamond
- **factory**: Address of the factory contract that deployed this diamond
- **svgManager**: Address of the SVG template manager for dynamic graphics
- **symbol**: Short identifier for the diamond (typically 3-5 characters)
- **name**: Descriptive name for the diamond instance

**Usage Example**:
```solidity
// Configure diamond settings
DiamondSettings memory settings = DiamondSettings({
    owner: msg.sender,
    factory: factoryAddress,
    svgManager: svgManagerAddress,
    symbol: "GEM1",
    name: "Gemforce Diamond Instance 1"
});
```

---

### DiamondContract
```solidity
struct DiamondContract {
    DiamondSettings settings;           // Core diamond configuration
    mapping(string => string) metadata; // Extensible metadata storage
}
```

**Purpose**: Encapsulates the complete diamond contract state including settings and metadata.

**Fields**:
- **settings**: Core diamond configuration (DiamondSettings struct)
- **metadata**: Key-value mapping for extensible metadata storage

**Metadata Use Cases**:
- **Version Information**: Track diamond version and upgrades
- **Business Logic**: Store business-specific configuration
- **Integration Data**: External service integration parameters
- **Audit Trail**: Track important state changes and events

---

### DiamondStorage
```solidity
struct DiamondStorage {
    DiamondContract diamondContract; // Complete diamond state
}
```

**Purpose**: Root storage structure for diamond state management using diamond storage pattern.

**Fields**:
- **diamondContract**: Complete diamond contract state and configuration

**Storage Pattern**:
```solidity
// Diamond storage access pattern
function diamondStorage() internal pure returns (DiamondStorage storage ds) {
    bytes32 position = keccak256("diamond.standard.diamond.storage");
    assembly {
        ds.slot := position
    }
}
```

## Core Interface Functions

### `getDiamondAddress()`
```solidity
function getDiamondAddress() external view returns (address)
```

**Purpose**: Returns the address of the diamond contract instance.

**Returns**:
- `address`: The address of the current diamond contract

**Usage**: This function provides a standardized way to retrieve the diamond's own address, which is essential for:
- **Self-Reference**: When the diamond needs to reference itself
- **Integration**: External contracts identifying the diamond
- **Logging**: Event emission with diamond address
- **Validation**: Confirming diamond identity in multi-diamond systems

**Example Usage**:
```solidity
// Get diamond address for integration
address diamondAddr = IDiamond(diamond).getDiamondAddress();
console.log("Diamond deployed at:", diamondAddr);

// Use in external contract integration
ExternalContract external = ExternalContract(externalAddress);
external.registerDiamond(diamondAddr);

// Self-reference in diamond functions
function someInternalFunction() internal {
    address self = IDiamond(address(this)).getDiamondAddress();
    emit DiamondOperation(self, "operation_completed");
}
```

## Implementation Examples

### Diamond Configuration Manager
```solidity
// Comprehensive diamond configuration management
contract DiamondConfigManager {
    using LibDiamond for DiamondStorage;
    
    event DiamondConfigured(address indexed diamond, DiamondSettings settings);
    event MetadataUpdated(address indexed diamond, string key, string value);
    event OwnershipTransferred(address indexed diamond, address indexed previousOwner, address indexed newOwner);
    
    function configureDiamond(
        DiamondSettings memory settings,
        string[] memory metadataKeys,
        string[] memory metadataValues
    ) external onlyOwner {
        require(metadataKeys.length == metadataValues.length, "Metadata arrays length mismatch");
        
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        
        // Set core settings
        ds.diamondContract.settings = settings;
        
        // Set metadata
        for (uint256 i = 0; i < metadataKeys.length; i++) {
            ds.diamondContract.metadata[metadataKeys[i]] = metadataValues[i];
            emit MetadataUpdated(address(this), metadataKeys[i], metadataValues[i]);
        }
        
        emit DiamondConfigured(address(this), settings);
    }
    
    function updateDiamondSettings(DiamondSettings memory newSettings) external onlyOwner {
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        
        address previousOwner = ds.diamondContract.settings.owner;
        ds.diamondContract.settings = newSettings;
        
        if (previousOwner != newSettings.owner) {
            emit OwnershipTransferred(address(this), previousOwner, newSettings.owner);
        }
        
        emit DiamondConfigured(address(this), newSettings);
    }
    
    function setMetadata(string memory key, string memory value) external onlyOwner {
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        ds.diamondContract.metadata[key] = value;
        
        emit MetadataUpdated(address(this), key, value);
    }
    
    function getMetadata(string memory key) external view returns (string memory) {
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        return ds.diamondContract.metadata[key];
    }
    
    function getDiamondSettings() external view returns (DiamondSettings memory) {
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        return ds.diamondContract.settings;
    }
    
    function getAllMetadata(string[] memory keys) external view returns (string[] memory values) {
        DiamondStorage storage ds = LibDiamond.diamondStorage();
        values = new string[](keys.length);
        
        for (uint256 i = 0; i < keys.length; i++) {
            values[i] = ds.diamondContract.metadata[keys[i]];
        }
    }
}
```

### Diamond Registry System
```solidity
// Registry for tracking multiple diamond instances
contract DiamondRegistry {
    struct DiamondInfo {
        address diamondAddress;
        DiamondSettings settings;
        uint256 createdAt;
        bool active;
        string category;
    }
    
    mapping(address => DiamondInfo) public diamonds;
    mapping(string => address[]) public diamondsByCategory;
    mapping(address => address[]) public diamondsByOwner;
    address[] public allDiamonds;
    
    event DiamondRegistered(address indexed diamond, address indexed owner, string category);
    event DiamondDeactivated(address indexed diamond);
    event DiamondCategoryUpdated(address indexed diamond, string oldCategory, string newCategory);
    
    function registerDiamond(
        address diamondAddress,
        string memory category
    ) external {
        require(diamondAddress != address(0), "Invalid diamond address");
        require(diamonds[diamondAddress].diamondAddress == address(0), "Diamond already registered");
        
        // Get diamond settings
        DiamondSettings memory settings;
        try IDiamond(diamondAddress).getDiamondAddress() returns (address addr) {
            require(addr == diamondAddress, "Diamond address mismatch");
            // Additional calls to get settings would require extended interface
        } catch {
            revert("Invalid diamond contract");
        }
        
        // Register diamond
        diamonds[diamondAddress] = DiamondInfo({
            diamondAddress: diamondAddress,
            settings: settings,
            createdAt: block.timestamp,
            active: true,
            category: category
        });
        
        allDiamonds.push(diamondAddress);
        diamondsByCategory[category].push(diamondAddress);
        diamondsByOwner[settings.owner].push(diamondAddress);
        
        emit DiamondRegistered(diamondAddress, settings.owner, category);
    }
    
    function deactivateDiamond(address diamondAddress) external {
        DiamondInfo storage info = diamonds[diamondAddress];
        require(info.diamondAddress != address(0), "Diamond not registered");
        require(info.active, "Diamond already deactivated");
        
        info.active = false;
        emit DiamondDeactivated(diamondAddress);
    }
    
    function updateDiamondCategory(
        address diamondAddress,
        string memory newCategory
    ) external {
        DiamondInfo storage info = diamonds[diamondAddress];
        require(info.diamondAddress != address(0), "Diamond not registered");
        
        string memory oldCategory = info.category;
        info.category = newCategory;
        
        // Update category mapping
        _removeFromCategoryArray(oldCategory, diamondAddress);
        diamondsByCategory[newCategory].push(diamondAddress);
        
        emit DiamondCategoryUpdated(diamondAddress, oldCategory, newCategory);
    }
    
    function getDiamondsByCategory(string memory category) external view returns (address[] memory) {
        return diamondsByCategory[category];
    }
    
    function getDiamondsByOwner(address owner) external view returns (address[] memory) {
        return diamondsByOwner[owner];
    }
    
    function getActiveDiamonds() external view returns (address[] memory) {
        uint256 activeCount = 0;
        
        // Count active diamonds
        for (uint256 i = 0; i < allDiamonds.length; i++) {
            if (diamonds[allDiamonds[i]].active) {
                activeCount++;
            }
        }
        
        // Build active diamonds array
        address[] memory activeDiamonds = new address[](activeCount);
        uint256 index = 0;
        
        for (uint256 i = 0; i < allDiamonds.length; i++) {
            if (diamonds[allDiamonds[i]].active) {
                activeDiamonds[index] = allDiamonds[i];
                index++;
            }
        }
        
        return activeDiamonds;
    }
    
    function _removeFromCategoryArray(string memory category, address diamondAddress) internal {
        address[] storage categoryArray = diamondsByCategory[category];
        
        for (uint256 i = 0; i < categoryArray.length; i++) {
            if (categoryArray[i] == diamondAddress) {
                categoryArray[i] = categoryArray[categoryArray.length - 1];
                categoryArray.pop();
                break;
            }
        }
    }
}
```

### Diamond Metadata Manager
```solidity
// Advanced metadata management for diamonds
contract DiamondMetadataManager {
    struct MetadataSchema {
        string[] requiredKeys;
        string[] optionalKeys;
        mapping(string => string) defaultValues;
        mapping(string => bool) isRequired;
    }
    
    mapping(string => MetadataSchema) public schemas;
    mapping(address => string) public diamondSchemas;
    
    event SchemaCreated(string indexed schemaName, string[] requiredKeys, string[] optionalKeys);
    event DiamondSchemaAssigned(address indexed diamond, string schemaName);
    event MetadataValidated(address indexed diamond, bool isValid);
    
    function createMetadataSchema(
        string memory schemaName,
        string[] memory requiredKeys,
        string[] memory optionalKeys,
        string[] memory defaultKeys,
        string[] memory defaultValues
    ) external onlyOwner {
        require(defaultKeys.length == defaultValues.length, "Default arrays length mismatch");
        
        MetadataSchema storage schema = schemas[schemaName];
        schema.requiredKeys = requiredKeys;
        schema.optionalKeys = optionalKeys;
        
        // Set required flags
        for (uint256 i = 0; i < requiredKeys.length; i++) {
            schema.isRequired[requiredKeys[i]] = true;
        }
        
        // Set default values
        for (uint256 i = 0; i < defaultKeys.length; i++) {
            schema.defaultValues[defaultKeys[i]] = defaultValues[i];
        }
        
        emit SchemaCreated(schemaName, requiredKeys, optionalKeys);
    }
    
    function assignSchemaToD diamond(address diamondAddress, string memory schemaName) external onlyOwner {
        require(schemas[schemaName].requiredKeys.length > 0, "Schema does not exist");
        
        diamondSchemas[diamondAddress] = schemaName;
        emit DiamondSchemaAssigned(diamondAddress, schemaName);
    }
    
    function validateDiamondMetadata(address diamondAddress) external view returns (
        bool isValid,
        string[] memory missingKeys,
        string[] memory suggestions
    ) {
        string memory schemaName = diamondSchemas[diamondAddress];
        require(bytes(schemaName).length > 0, "No schema assigned to diamond");
        
        MetadataSchema storage schema = schemas[schemaName];
        string[] memory missing = new string[](schema.requiredKeys.length);
        string[] memory suggest = new string[](schema.requiredKeys.length);
        uint256 missingCount = 0;
        uint256 suggestCount = 0;
        
        // Check required keys
        for (uint256 i = 0; i < schema.requiredKeys.length; i++) {
            string memory key = schema.requiredKeys[i];
            
            // This would require extended interface to get metadata
            // For now, we'll simulate the check
            bool hasKey = _checkDiamondHasMetadata(diamondAddress, key);
            
            if (!hasKey) {
                missing[missingCount] = key;
                missingCount++;
                
                // Suggest default value if available
                if (bytes(schema.defaultValues[key]).length > 0) {
                    suggest[suggestCount] = string(abi.encodePacked(key, "=", schema.defaultValues[key]));
                    suggestCount++;
                }
            }
        }
        
        // Resize arrays
        missingKeys = new string[](missingCount);
        suggestions = new string[](suggestCount);
        
        for (uint256 i = 0; i < missingCount; i++) {
            missingKeys[i] = missing[i];
        }
        
        for (uint256 i = 0; i < suggestCount; i++) {
            suggestions[i] = suggest[i];
        }
        
        isValid = missingCount == 0;
    }
    
    function getSchemaInfo(string memory schemaName) external view returns (
        string[] memory requiredKeys,
        string[] memory optionalKeys
    ) {
        MetadataSchema storage schema = schemas[schemaName];
        return (schema.requiredKeys, schema.optionalKeys);
    }
    
    function _checkDiamondHasMetadata(address diamondAddress, string memory key) internal view returns (bool) {
        // This would require extended interface or direct storage access
        // Placeholder implementation
        return true;
    }
}
```

### Diamond Factory Integration
```solidity
// Integration with diamond factory for standardized deployment
contract DiamondFactoryIntegration {
    struct DeploymentConfig {
        DiamondSettings settings;
        string[] metadataKeys;
        string[] metadataValues;
        address[] initialFacets;
        bytes4[][] facetSelectors;
    }
    
    mapping(address => DeploymentConfig) public deploymentConfigs;
    mapping(address => address) public factoryToDiamond;
    
    event DiamondDeploymentConfigured(address indexed factory, DeploymentConfig config);
    event DiamondDeployed(address indexed factory, address indexed diamond, DiamondSettings settings);
    
    function configureDeployment(
        address factory,
        DeploymentConfig memory config
    ) external onlyOwner {
        deploymentConfigs[factory] = config;
        
        emit DiamondDeploymentConfigured(factory, config);
    }
    
    function deployDiamond(address factory) external {
        DeploymentConfig memory config = deploymentConfigs[factory];
        require(config.settings.owner != address(0), "Deployment config not set");
        
        // Deploy diamond via factory
        address diamond = IDiamondFactory(factory).deployDiamond({
            templateName: "Custom",
            salt: keccak256(abi.encodePacked(block.timestamp, msg.sender)),
            initData: "",
            owner: config.settings.owner,
            gasLimit: 0
        });
        
        factoryToDiamond[factory] = diamond;
        
        // Set initial facets and metadata
        IDiamondCut(diamond).diamondCut(
            _buildFacetCuts(config.initialFacets, config.facetSelectors),
            address(0),
            ""
        );
        
        IDiamond(diamond).configureDiamond(config.settings, config.metadataKeys, config.metadataValues);
        
        emit DiamondDeployed(factory, diamond, config.settings);
    }
    
    function _buildFacetCuts(address[] memory facets, bytes4[][] memory selectors) internal pure returns (IDiamondCut.FacetCut[] memory) {
        IDiamondCut.FacetCut[] memory cuts = new IDiamondCut.FacetCut[](facets.length);
        for (uint i = 0; i < facets.length; i++) {
            cuts[i] = IDiamondCut.FacetCut({
                facetAddress: facets[i],
                action: IDiamondCut.FacetCutAction.Add,
                functionSelectors: selectors[i]
            });
        }
        return cuts;
    }
}
```

### Standard Diamond Implementation
```solidity
// Basic Diamond contract implementation adhering to IDiamond interface
contract MyDiamond is IDiamond, Diamond, DiamondCutFacet, DiamondLoupeFacet, OwnershipFacet {
    // Constructor initializes DiamondStorage
    constructor(DiamondSettings memory initialSettings, address diamondInit, bytes memory initCalldata) {
        // Perform initialization
        LibDiamond.diamondStorage().diamondContract.settings = initialSettings;
        
        // Optional initialization call
        if (diamondInit != address(0)) {
            (bool success,) = diamondInit.delegatecall(initCalldata);
            require(success, "Initialization failed");
        }
    }
}
```

### Extended Diamond Interface
```solidity
interface IExtendedDiamond is IDiamond {
    // New functions can be added here
    function version() external view returns (string memory);
    function upgradeTo(address newImplementation) external;
}
```

## Security Considerations

### Access Control
- **Owner-Only Functions**: Sensitive operations restricted to the diamond owner
- **Factory Restrictions**: Only trusted factories can create diamonds
- **Delegated Calls**: Careful handling of delegatecall to prevent vulnerabilities

### Data Integrity
- **Storage Collisions**: Prevention through Diamond Storage pattern
- **Immutable Data**: Critical configuration once set is immutable
- **Type Safety**: Enforcement of data types in structures
- **Event Logging**: Comprehensive event logs for auditability

### Interface Compliance
- **EIP-2535 Adherence**: Full compliance with Diamond Standard
- **EIP-165 Support**: Standard interface detection
- **Consistent APIs**: Predictable and reliable contract interactions

## Gas Optimization

### Storage Efficiency
- **Minimal Storage**: Only essential data stored on-chain
- **Packed Structs**: Optimize storage layout for structs
- **Lazy Initialization**: Initialize only when necessary
- **View Functions**: No gas cost for read-only queries

### Function Optimization
- **Internal Helper Functions**: Reduce external calls
- **Delegatecall**: Minimize gas cost for cross-facet calls
- **Batch Operations**: Efficient processing of multiple items

## Testing

### Unit Tests
- Test each core function in isolation
- Verify data structure integrity
- Simulate various configuration scenarios

### Integration Tests
- Test deployment through Diamond Factory
- Verify cross-facet and cross-contract interactions
- Simulate upgrade scenarios and state preservation

## Related Documentation

- [Diamond Standard Overview](../../smart-contracts/diamond.md)
- [Diamond Factory](../../smart-contracts/diamond-factory.md)
- [Diamond Cut Facet](../../smart-contracts/facets/diamond-cut-facet.md)
- [Diamond Loupe Facet](../../smart-contracts/facets/diamond-loupe-facet.md)
- [Owner Facet](../../smart-contracts/facets/ownership-facet.md)
- [Diamond Architecture](../../developer-guides/development-environment-setup.md)
- [Integrator's Guide: Smart Contracts](../../integrator-guide/smart-contracts.md) - General smart contract integration guidance.