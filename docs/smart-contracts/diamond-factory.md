# Diamond Factory

The Diamond Factory is a contract deployment system that creates new diamond contracts with predefined facet configurations. It provides a standardized way to deploy diamonds with consistent initialization and upgrade patterns.

## Overview

The Diamond Factory provides:

- **Standardized Deployment**: Deploy diamonds with consistent configurations
- **Template Management**: Manage diamond templates and facet combinations
- **Initialization Support**: Handle complex initialization sequences
- **Upgrade Patterns**: Support for upgradeable diamond deployments
- **Access Control**: Manage who can deploy new diamonds

## Key Features

### Diamond Deployment
- **Template-Based**: Deploy from predefined templates
- **Custom Configuration**: Support custom facet combinations
- **Initialization**: Handle complex initialization logic
- **Event Tracking**: Track all deployed diamonds

### Template Management
- **Template Registry**: Store and manage diamond templates
- **Facet Combinations**: Define standard facet sets
- **Version Control**: Support multiple template versions
- **Validation**: Validate template configurations

### Factory Patterns
- **Clone Factory**: Efficient diamond cloning
- **Minimal Proxy**: Gas-efficient deployment patterns
- **Deterministic Addresses**: Predictable diamond addresses
- **Batch Deployment**: Deploy multiple diamonds efficiently

## Core Interface

```solidity
interface IDiamondFactory {
    struct DiamondTemplate {
        string name;
        string version;
        FacetCut[] facetCuts;
        address initContract;
        bytes initData;
        bool active;
    }
    
    struct DeploymentConfig {
        string templateName;
        bytes32 salt;
        bytes initData;
        address owner;
    }
    
    event DiamondDeployed(
        address indexed diamond,
        address indexed owner,
        string templateName,
        bytes32 salt
    );
    
    event TemplateRegistered(
        string indexed name,
        string version,
        address indexed creator
    );
    
    function deployDiamond(DeploymentConfig calldata config) external returns (address diamond);
    function registerTemplate(DiamondTemplate calldata template) external;
    function getTemplate(string calldata name) external view returns (DiamondTemplate memory);
    function predictDiamondAddress(DeploymentConfig calldata config) external view returns (address);
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
    initData: abi.encode(name, symbol, baseURI),
    owner: msg.sender
});

address diamond = IDiamondFactory(factory).deployDiamond(config);
```

### registerTemplate()
Registers a new diamond template for deployment.

**Parameters:**
- `template`: Template configuration including facets and initialization

**Access Control:**
- Restricted to authorized template creators

**Usage:**
```solidity
IDiamondFactory.DiamondTemplate memory template = IDiamondFactory.DiamondTemplate({
    name: "StandardNFT",
    version: "1.0.0",
    facetCuts: facetCuts,
    initContract: initContract,
    initData: initData,
    active: true
});

IDiamondFactory(factory).registerTemplate(template);
```

### predictDiamondAddress()
Predicts the address of a diamond before deployment.

**Parameters:**
- `config`: Deployment configuration

**Returns:**
- `address`: Predicted diamond address

**Usage:**
```solidity
address predictedAddress = IDiamondFactory(factory).predictDiamondAddress(config);
```

## Implementation Example

### Basic Diamond Factory

```solidity
contract DiamondFactory is IDiamondFactory {
    using LibDiamondFactory for DiamondTemplate;
    
    mapping(string => DiamondTemplate) private templates;
    mapping(address => bool) public templateCreators;
    address[] public deployedDiamonds;
    
    modifier onlyTemplateCreator() {
        require(templateCreators[msg.sender] || msg.sender == owner(), "Not authorized");
        _;
    }
    
    function deployDiamond(DeploymentConfig calldata config) 
        external 
        override 
        returns (address diamond) 
    {
        DiamondTemplate memory template = templates[config.templateName];
        require(template.active, "Template not active");
        
        // Deploy diamond using CREATE2 for deterministic addresses
        bytes32 salt = keccak256(abi.encodePacked(config.salt, msg.sender));
        diamond = Clones.cloneDeterministic(diamondImplementation, salt);
        
        // Initialize diamond with template configuration
        IDiamondCut(diamond).diamondCut(
            template.facetCuts,
            template.initContract,
            abi.encodePacked(template.initData, config.initData)
        );
        
        // Transfer ownership to specified owner
        IOwnership(diamond).transferOwnership(config.owner);
        
        deployedDiamonds.push(diamond);
        
        emit DiamondDeployed(diamond, config.owner, config.templateName, config.salt);
    }
    
    function registerTemplate(DiamondTemplate calldata template) 
        external 
        override 
        onlyTemplateCreator 
    {
        require(bytes(template.name).length > 0, "Invalid template name");
        require(template.facetCuts.length > 0, "No facets specified");
        
        templates[template.name] = template;
        
        emit TemplateRegistered(template.name, template.version, msg.sender);
    }
}
```

### Advanced Factory with Versioning

```solidity
contract VersionedDiamondFactory {
    struct VersionedTemplate {
        mapping(string => DiamondTemplate) versions;
        string[] versionList;
        string latestVersion;
    }
    
    mapping(string => VersionedTemplate) private templates;
    
    function registerTemplateVersion(
        string calldata name,
        string calldata version,
        DiamondTemplate calldata template
    ) external onlyTemplateCreator {
        VersionedTemplate storage versionedTemplate = templates[name];
        
        // Check if version already exists
        require(
            bytes(versionedTemplate.versions[version].name).length == 0,
            "Version already exists"
        );
        
        versionedTemplate.versions[version] = template;
        versionedTemplate.versionList.push(version);
        versionedTemplate.latestVersion = version;
        
        emit TemplateVersionRegistered(name, version, msg.sender);
    }
    
    function deployDiamondVersion(
        string calldata templateName,
        string calldata version,
        DeploymentConfig calldata config
    ) external returns (address diamond) {
        DiamondTemplate memory template = templates[templateName].versions[version];
        require(template.active, "Template version not active");
        
        // Deploy with version-specific logic
        return _deployDiamondFromTemplate(template, config);
    }
}
```

## Template Patterns

### Standard NFT Template

```solidity
function createNFTTemplate() internal pure returns (DiamondTemplate memory) {
    FacetCut[] memory facetCuts = new FacetCut[](4);
    
    // ERC721 Facet
    facetCuts[0] = FacetCut({
        facetAddress: erc721FacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getERC721Selectors()
    });
    
    // Marketplace Facet
    facetCuts[1] = FacetCut({
        facetAddress: marketplaceFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getMarketplaceSelectors()
    });
    
    // Metadata Facet
    facetCuts[2] = FacetCut({
        facetAddress: metadataFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getMetadataSelectors()
    });
    
    // Ownership Facet
    facetCuts[3] = FacetCut({
        facetAddress: ownershipFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getOwnershipSelectors()
    });
    
    return DiamondTemplate({
        name: "StandardNFT",
        version: "1.0.0",
        facetCuts: facetCuts,
        initContract: nftInitContract,
        initData: "",
        active: true
    });
}
```

### DeFi Protocol Template

```solidity
function createDeFiTemplate() internal pure returns (DiamondTemplate memory) {
    FacetCut[] memory facetCuts = new FacetCut[](5);
    
    // Token Facet
    facetCuts[0] = FacetCut({
        facetAddress: tokenFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getTokenSelectors()
    });
    
    // Staking Facet
    facetCuts[1] = FacetCut({
        facetAddress: stakingFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getStakingSelectors()
    });
    
    // Governance Facet
    facetCuts[2] = FacetCut({
        facetAddress: governanceFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getGovernanceSelectors()
    });
    
    // Fee Distribution Facet
    facetCuts[3] = FacetCut({
        facetAddress: feeDistributorFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getFeeDistributorSelectors()
    });
    
    // Treasury Facet
    facetCuts[4] = FacetCut({
        facetAddress: treasuryFacetAddress,
        action: FacetCutAction.Add,
        functionSelectors: getTreasurySelectors()
    });
    
    return DiamondTemplate({
        name: "DeFiProtocol",
        version: "1.0.0",
        facetCuts: facetCuts,
        initContract: defiInitContract,
        initData: "",
        active: true
    });
}
```

## Security Considerations

### Access Control
- **Template Creation**: Restrict who can register templates
- **Deployment Permissions**: Control diamond deployment access
- **Owner Assignment**: Validate diamond ownership assignment
- **Template Validation**: Validate template configurations

### Template Security
- **Facet Validation**: Ensure facet contracts are secure
- **Initialization Safety**: Validate initialization logic
- **Upgrade Restrictions**: Consider upgrade limitations
- **Version Control**: Manage template versions securely

### Deployment Safety
- **Address Prediction**: Secure address prediction mechanisms
- **Salt Management**: Prevent salt collision attacks
- **Initialization Atomicity**: Ensure atomic initialization
- **Ownership Transfer**: Secure ownership transfer process

## Best Practices

### Template Design
1. **Modular Facets**: Design reusable, modular facets
2. **Standard Interfaces**: Use standard interfaces for compatibility
3. **Initialization Patterns**: Follow consistent initialization patterns
4. **Documentation**: Document template capabilities and limitations

### Factory Management
1. **Access Control**: Implement proper access controls
2. **Template Validation**: Validate all template configurations
3. **Event Logging**: Log all deployments and template changes
4. **Upgrade Planning**: Plan for factory upgrades

### Deployment Patterns
1. **Deterministic Addresses**: Use CREATE2 for predictable addresses
2. **Batch Operations**: Support batch deployments for efficiency
3. **Gas Optimization**: Optimize deployment gas costs
4. **Error Handling**: Implement comprehensive error handling

## Integration Examples

### Frontend Integration

```typescript
class DiamondFactory {
    private contract: Contract;
    
    constructor(address: string, provider: Provider) {
        this.contract = new Contract(address, DIAMOND_FACTORY_ABI, provider);
    }
    
    async deployDiamond(config: DeploymentConfig): Promise<string> {
        const tx = await this.contract.deployDiamond(config);
        const receipt = await tx.wait();
        
        const event = receipt.events?.find(e => e.event === 'DiamondDeployed');
        return event?.args?.diamond;
    }
    
    async predictAddress(config: DeploymentConfig): Promise<string> {
        return await this.contract.predictDiamondAddress(config);
    }
    
    async getTemplate(name: string): Promise<DiamondTemplate> {
        return await this.contract.getTemplate(name);
    }
}
```

### CLI Integration

```bash
# Deploy a diamond using CLI
gemforce deploy-diamond \
  --template "StandardNFT" \
  --name "My NFT Collection" \
  --symbol "MNC" \
  --owner "0x..." \
  --network "sepolia"

# Register a new template
gemforce register-template \
  --name "CustomNFT" \
  --version "1.0.0" \
  --config "./templates/custom-nft.json" \
  --network "sepolia"
```

## Related Documentation

- [Diamond Standard Overview](diamond.md)
- [Diamond Cut Facet](facets/diamond-cut-facet.md)
- [Diamond Loupe Facet](facets/diamond-loupe-facet.md)
- [Diamond Factory Library](libraries/diamond-factory-lib.md)
- [Deployment Guides: Multi-Network Deployment](../deployment-guides/multi-network-deployment.md)
- [Developer Guides: Development Environment Setup](../developer-guides/development-environment-setup.md)

## Standards Compliance

- **EIP-2535**: Diamond Standard implementation
- **EIP-1167**: Minimal Proxy Standard for cloning
- **EIP-1014**: CREATE2 for deterministic addresses
- **Factory Pattern**: Standard factory design patterns