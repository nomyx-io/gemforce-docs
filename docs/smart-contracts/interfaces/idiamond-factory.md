# IDiamondFactory Interface

The IDiamondFactory interface defines the standard for diamond contract factory implementations. It provides a standardized way to deploy diamond contracts with predefined facet configurations and initialization parameters.

## Overview

IDiamondFactory provides:

- **Standardized Deployment**: Consistent diamond deployment patterns
- **Template Management**: Manage diamond templates and configurations
- **Batch Deployment**: Deploy multiple diamonds efficiently
- **Configuration Validation**: Validate deployment parameters
- **Event Tracking**: Track all diamond deployments

## Key Features

### Diamond Deployment
- **Template-Based**: Deploy from predefined templates
- **Custom Configuration**: Support custom facet combinations
- **Deterministic Addresses**: Predictable diamond addresses using CREATE2
- **Initialization Support**: Handle complex initialization sequences

### Template Management
- **Template Registry**: Store and manage diamond templates
- **Version Control**: Support multiple template versions
- **Access Control**: Manage template creation permissions
- **Validation**: Validate template configurations

### Factory Operations
- **Batch Operations**: Deploy multiple diamonds in single transaction
- **Gas Optimization**: Optimize deployment gas costs
- **Event Logging**: Comprehensive event logging for tracking
- **Error Handling**: Robust error handling and recovery

## Interface Definition

```solidity
interface IDiamondFactory {
    // Events
    event DiamondDeployed(
        address indexed diamond,
        address indexed deployer,
        string indexed templateName,
        bytes32 salt
    );
    
    event TemplateRegistered(
        string indexed name,
        string indexed version,
        address indexed creator
    );
    
    event TemplateUpdated(
        string indexed name,
        string indexed version,
        address indexed updater
    );
    
    event TemplateDeactivated(
        string indexed name,
        address indexed deactivator
    );
    
    // Structs
    struct DiamondTemplate {
        string name;
        string version;
        FacetCut[] facetCuts;
        address initContract;
        bytes initData;
        bool active;
        address creator;
        uint256 createdAt;
    }
    
    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }
    
    enum FacetCutAction {
        Add,
        Replace,
        Remove
    }
    
    struct DeploymentConfig {
        string templateName;
        bytes32 salt;
        bytes initData;
        address owner;
        uint256 gasLimit;
    }
    
    struct DeploymentResult {
        address diamond;
        uint256 gasUsed;
        bytes32 txHash;
    }
    
    // Core Functions
    function deployDiamond(DeploymentConfig calldata config) 
        external 
        returns (address diamond);
    
    function deployDiamonds(DeploymentConfig[] calldata configs) 
        external 
        returns (address[] memory diamonds);
    
    function predictDiamondAddress(DeploymentConfig calldata config) 
        external 
        view 
        returns (address predicted);
    
    // Template Management
    function registerTemplate(DiamondTemplate calldata template) external;
    
    function updateTemplate(
        string calldata name, 
        DiamondTemplate calldata template
    ) external;
    
    function deactivateTemplate(string calldata name) external;
    
    function getTemplate(string calldata name) 
        external 
        view 
        returns (DiamondTemplate memory);
    
    function getTemplateNames() 
        external 
        view 
        returns (string[] memory);
    
    function isTemplateActive(string calldata name) 
        external 
        view 
        returns (bool);
    
    // Query Functions
    function getDeployedDiamonds(address deployer) 
        external 
        view 
        returns (address[] memory);
    
    function getDiamondTemplate(address diamond) 
        external 
        view 
        returns (string memory templateName);
    
    function getDeploymentCount() 
        external 
        view 
        returns (uint256);
    
    function getDeploymentInfo(address diamond) 
        external 
        view 
        returns (
            address deployer,
            string memory templateName,
            uint256 deployedAt,
            bytes32 salt
        );
    
    // Administrative Functions
    function setTemplateCreator(address creator, bool authorized) external;
    
    function isAuthorizedCreator(address creator) 
        external 
        view 
        returns (bool);
    
    function pause() external;
    function unpause() external;
    function paused() external view returns (bool);
}
```

## Core Functions

### deployDiamond()
Deploys a new diamond contract using a registered template.

**Parameters:**
- `config`: Deployment configuration including template name, salt, and initialization data

**Returns:**
- `address`: Address of the deployed diamond

**Usage:**
```solidity
IDiamondFactory.DeploymentConfig memory config = IDiamondFactory.DeploymentConfig({
    templateName: "StandardNFT",
    salt: keccak256("unique-identifier"),
    initData: abi.encode("My Collection", "MYC", "https://api.example.com/"),
    owner: msg.sender,
    gasLimit: 5000000
});

address diamond = factory.deployDiamond(config);
```

### deployDiamonds()
Deploys multiple diamonds in a single transaction for gas efficiency.

**Parameters:**
- `configs`: Array of deployment configurations

**Returns:**
- `address[]`: Array of deployed diamond addresses

### predictDiamondAddress()
Predicts the address of a diamond before deployment using CREATE2.

**Parameters:**
- `config`: Deployment configuration

**Returns:**
- `address`: Predicted diamond address

### registerTemplate()
Registers a new diamond template for deployment.

**Parameters:**
- `template`: Template configuration including facets and initialization

**Access Control:**
- Only authorized template creators can register templates

### getTemplate()
Retrieves a registered template by name.

**Parameters:**
- `name`: Template name

**Returns:**
- `DiamondTemplate`: Complete template configuration

## Implementation Example

### Basic Diamond Factory

```solidity
contract DiamondFactory is IDiamondFactory, Ownable, Pausable {
    using Clones for address;
    
    // Storage
    mapping(string => DiamondTemplate) private templates;
    mapping(address => bool) public authorizedCreators;
    mapping(address => string) public diamondTemplates;
    mapping(address => address[]) public deployerDiamonds;
    
    address[] public allDiamonds;
    string[] public templateNames;
    
    address public immutable diamondImplementation;
    
    constructor(address _diamondImplementation) {
        diamondImplementation = _diamondImplementation;
        authorizedCreators[msg.sender] = true;
    }
    
    function deployDiamond(DeploymentConfig calldata config) 
        external 
        override 
        whenNotPaused 
        returns (address diamond) 
    {
        DiamondTemplate memory template = templates[config.templateName];
        require(template.active, "Template not active");
        require(bytes(template.name).length > 0, "Template not found");
        
        // Calculate deterministic address
        bytes32 salt = keccak256(abi.encodePacked(config.salt, msg.sender));
        
        // Deploy diamond using CREATE2
        diamond = Clones.cloneDeterministic(diamondImplementation, salt);
        
        // Initialize diamond with template configuration
        _initializeDiamond(diamond, template, config);
        
        // Record deployment
        _recordDeployment(diamond, config.templateName, msg.sender, salt);
        
        emit DiamondDeployed(diamond, msg.sender, config.templateName, config.salt);
    }
    
    function deployDiamonds(DeploymentConfig[] calldata configs) 
        external 
        override 
        whenNotPaused 
        returns (address[] memory diamonds) 
    {
        diamonds = new address[](configs.length);
        
        for (uint256 i = 0; i < configs.length; i++) {
            diamonds[i] = this.deployDiamond(configs[i]);
        }
    }
    
    function predictDiamondAddress(DeploymentConfig calldata config) 
        external 
        view 
        override 
        returns (address predicted) 
    {
        bytes32 salt = keccak256(abi.encodePacked(config.salt, msg.sender));
        predicted = Clones.predictDeterministicAddress(diamondImplementation, salt);
    }
    
    function registerTemplate(DiamondTemplate calldata template) 
        external 
        override 
    {
        require(authorizedCreators[msg.sender], "Not authorized");
        require(bytes(template.name).length > 0, "Invalid template name");
        require(template.facetCuts.length > 0, "No facets specified");
        
        // Validate facet cuts
        _validateFacetCuts(template.facetCuts);
        
        // Store template
        templates[template.name] = DiamondTemplate({
            name: template.name,
            version: template.version,
            facetCuts: template.facetCuts,
            initContract: template.initContract,
            initData: template.initData,
            active: true,
            creator: msg.sender,
            createdAt: block.timestamp
        });
        
        // Add to template names if new
        if (!_templateExists(template.name)) {
            templateNames.push(template.name);
        }
        
        emit TemplateRegistered(template.name, template.version, msg.sender);
    }
    
    function _initializeDiamond(
        address diamond,
        DiamondTemplate memory template,
        DeploymentConfig calldata config
    ) internal {
        // Perform diamond cut with template facets
        IDiamondCut(diamond).diamondCut(
            template.facetCuts,
            template.initContract,
            abi.encodePacked(template.initData, config.initData)
        );
        
        // Transfer ownership to specified owner
        if (config.owner != address(0)) {
            IOwnership(diamond).transferOwnership(config.owner);
        }
    }
    
    function _recordDeployment(
        address diamond,
        string memory templateName,
        address deployer,
        bytes32 salt
    ) internal {
        diamondTemplates[diamond] = templateName;
        deployerDiamonds[deployer].push(diamond);
        allDiamonds.push(diamond);
    }
}
```

### Advanced Template Management

```solidity
contract AdvancedDiamondFactory is DiamondFactory {
    // Template versioning
    mapping(string => mapping(string => DiamondTemplate)) private templateVersions;
    mapping(string => string[]) private templateVersionList;
    mapping(string => string) private latestVersion;
    
    // Template categories
    mapping(string => string) public templateCategories;
    mapping(string => string[]) public categoryTemplates;
    
    // Deployment statistics
    mapping(string => uint256) public templateDeploymentCount;
    mapping(address => uint256) public deployerCount;
    
    function registerTemplateVersion(
        string calldata name,
        string calldata version,
        DiamondTemplate calldata template
    ) external {
        require(authorizedCreators[msg.sender], "Not authorized");
        require(
            templateVersions[name][version].createdAt == 0,
            "Version already exists"
        );
        
        templateVersions[name][version] = template;
        templateVersionList[name].push(version);
        latestVersion[name] = version;
        
        emit TemplateRegistered(name, version, msg.sender);
    }
    
    function deployDiamondVersion(
        string calldata templateName,
        string calldata version,
        DeploymentConfig calldata config
    ) external returns (address diamond) {
        DiamondTemplate memory template = templateVersions[templateName][version];
        require(template.active, "Template version not active");
        
        // Deploy with specific version
        return _deployWithTemplate(template, config);
    }
    
    function getTemplateVersions(string calldata name) 
        external 
        view 
        returns (string[] memory versions) 
    {
        return templateVersionList[name];
    }
    
    function getLatestVersion(string calldata name) 
        external 
        view 
        returns (string memory version) 
    {
        return latestVersion[name];
    }
    
    function categorizeTemplate(
        string calldata templateName,
        string calldata category
    ) external onlyOwner {
        templateCategories[templateName] = category;
        categoryTemplates[category].push(templateName);
    }
    
    function getTemplatesByCategory(string calldata category) 
        external 
        view 
        returns (string[] memory templates) 
    {
        return categoryTemplates[category];
    }
}
```

## Template Examples

### NFT Collection Template

```solidity
function createNFTCollectionTemplate() external pure returns (DiamondTemplate memory) {
    FacetCut[] memory facetCuts = new FacetCut[](4);
    
    facetCuts[0] = FacetCut({
        facetAddress: 0x..., // ERC721Facet
        action: FacetCutAction.Add,
        functionSelectors: getERC721Selectors()
    });
    
    facetCuts[1] = FacetCut({
        facetAddress: 0x..., // MarketplaceFacet
        action: FacetCutAction.Add,
        functionSelectors: getMarketplaceSelectors()
    });
    
    facetCuts[2] = FacetCut({
        facetAddress: 0x..., // MetadataFacet
        action: FacetCutAction.Add,
        functionSelectors: getMetadataSelectors()
    });
    
    facetCuts[3] = FacetCut({
        facetAddress: 0x..., // OwnershipFacet
        action: FacetCutAction.Add,
        functionSelectors: getOwnershipSelectors()
    });
    
    return DiamondTemplate({
        name: "NFTCollection",
        version: "1.0.0",
        facetCuts: facetCuts,
        initContract: 0x..., // NFTCollectionInit
        initData: "",
        active: true,
        creator: msg.sender,
        createdAt: block.timestamp
    });
}
```

### DeFi Protocol Template

```solidity
function createDeFiTemplate() internal pure returns (DiamondTemplate memory) {
    FacetCut[] memory facetCuts = new FacetCut[](5);
    
    facetCuts[0] = FacetCut({
        facetAddress: 0x..., // Token Facet
        action: FacetCutAction.Add,
        functionSelectors: getTokenSelectors()
    });
    
    facetCuts[1] = FacetCut({
        facetAddress: 0x..., // Staking Facet
        action: FacetCutAction.Add,
        functionSelectors: getStakingSelectors()
    });
    
    facetCuts[2] = FacetCut({
        facetAddress: 0x..., // Governance Facet
        action: FacetCutAction.Add,
        functionSelectors: getGovernanceSelectors()
    });
    
    facetCuts[3] = FacetCut({
        facetAddress: 0x..., // Fee Distribution Facet
        action: FacetCutAction.Add,
        functionSelectors: getFeeDistributorSelectors()
    });
    
    facetCuts[4] = FacetCut({
        facetAddress: 0x..., // Treasury Facet
        action: FacetCutAction.Add,
        functionSelectors: getTreasurySelectors()
    });
    
    return DiamondTemplate({
        name: "DeFiProtocol",
        version: "1.0.0",
        facetCuts: facetCuts,
        initContract: 0x..., // DeFiInitContract
        initData: "",
        active: true,
        creator: msg.sender,
        createdAt: block.timestamp
    });
}
```

## Related Documentation

- [Diamond Factory](../../smart-contracts/diamond-factory.md) - Reference for the Diamond Factory implementation.
- [IDiamondFactory Interface](../../smart-contracts/interfaces/idiamond-factory.md) - Interface definition.
- [SDK & Libraries: Deploy Utilities](../../sdk-libraries/deploy.md) - SDK utilities for deployment.
- [Deployment Guides: Multi-Network Deployment](../../deployment-guides/multi-network-deployment.md) - Guides on multi-network deployments.
- [Integrator's Guide: Smart Contracts](../../integrator-guide/smart-contracts.md) - General smart contract integration.