# SVGTemplatesFacet

## Overview

The [`SVGTemplatesFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/SVGTemplatesFacet.sol) provides SVG template management functionality within the Gemforce diamond system. This facet enables the creation, storage, and retrieval of SVG templates that can be used for dynamic NFT metadata generation, visual representations, and on-chain graphics rendering.

## Contract Details

- **Contract Name**: `SVGTemplatesFacet`
- **Inheritance**: `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 SVG Template Management
- Create and store SVG templates on-chain
- Retrieve SVG data as strings
- Template address resolution
- Centralized SVG manager integration

### 🔹 Dynamic Graphics Generation
- On-chain SVG rendering capabilities
- Template-based NFT metadata
- Scalable vector graphics support
- Customizable visual elements

### 🔹 Manager-Based Architecture
- Delegated SVG management system
- Upgradeable template storage
- Centralized template registry
- Access control for template creation

## Core Architecture

### SVG Manager Integration
The facet operates through a manager-based architecture where an external SVG manager contract handles the actual template storage and rendering logic. This design provides:

- **Separation of Concerns**: Template logic separated from diamond storage
- **Upgradeability**: SVG manager can be updated without affecting the diamond
- **Scalability**: Dedicated contract for complex SVG operations
- **Modularity**: Clean interface between diamond and SVG functionality

### Template Storage Pattern
```solidity
// SVG templates are managed through external manager
address svgManager = SVGTemplatesLib.svgStorage().svgManager;

// Templates accessed via manager interface
ISVGTemplate(svgManager).svgString(templateName);
```

## Core Functions

### Management Functions

#### `setSVGManager()`
```solidity
function setSVGManager(address _manager) external onlyOwner
```

**Purpose**: Sets the address of the SVG manager contract that handles template operations.

**Parameters**:
- `_manager` (address): Address of the SVG manager contract

**Access Control**: Owner only

**Process**:
1. Updates the SVG manager address in diamond storage
2. All subsequent SVG operations will use the new manager
3. Enables upgrading SVG functionality without diamond changes

**Example Usage**:
```solidity
// Deploy new SVG manager with enhanced features
address newSVGManager = address(new EnhancedSVGManager());

// Update the diamond to use new manager
ISVGTemplates(diamond).setSVGManager(newSVGManager);
console.log("SVG manager updated to:", newSVGManager);
```

**Security Considerations**:
- Only contract owner can change the manager
- Ensure new manager implements ISVGTemplate interface
- Verify manager contract before setting

---

#### `createSVG()`
```solidity
function createSVG(string memory _name) external onlyOwner returns(address _tplAddress)
```

**Purpose**: Creates a new SVG template with the specified name.

**Parameters**:
- `_name` (string): Unique name for the SVG template

**Returns**:
- `_tplAddress` (address): Address of the created template contract

**Access Control**: Owner only

**Process**:
1. Delegates template creation to the SVG manager
2. Manager creates and deploys template contract
3. Returns template address for future reference
4. Emits SVGTemplateCreated event

**Events**: `SVGTemplateCreated(name, template)`

**Example Usage**:
```solidity
// Create template for NFT badges
address badgeTemplate = ISVGTemplates(diamond).createSVG("achievement-badge");
console.log("Badge template created at:", badgeTemplate);

// Create template for dynamic backgrounds
address bgTemplate = ISVGTemplates(diamond).createSVG("gradient-background");
console.log("Background template created at:", bgTemplate);
```

**Error Conditions**:
- Template name already exists
- SVG manager not set
- Template creation failure

### Query Functions

#### `svgs()`
```solidity
function svgs() external view returns (string[] memory)
```

**Purpose**: Returns all SVG template names stored in the system.

**Returns**: Array of template names

**Example Usage**:
```solidity
string[] memory templateNames = ISVGTemplates(diamond).svgs();

console.log("Available templates:");
for (uint256 i = 0; i < templateNames.length; i++) {
    console.log("-", templateNames[i]);
}
```

**Use Cases**:
- UI template selection
- Template enumeration
- System inventory
- Integration discovery

---

#### `svgAddress()`
```solidity
function svgAddress(string memory _name) external view returns (address _svgAddress)
```

**Purpose**: Returns the contract address for a specific SVG template.

**Parameters**:
- `_name` (string): Name of the SVG template

**Returns**:
- `_svgAddress` (address): Contract address of the template

**Example Usage**:
```solidity
address templateAddress = ISVGTemplates(diamond).svgAddress("achievement-badge");

if (templateAddress != address(0)) {
    console.log("Template found at:", templateAddress);
    // Use template for rendering
} else {
    console.log("Template not found");
}
```

**Important Notes**:
- Returns address even if template doesn't exist (may be zero address)
- Check address validity before using
- Address doesn't guarantee template functionality

---

#### `svgString()`
```solidity
function svgString(string memory _name) external view returns (string memory data_)
```

**Purpose**: Returns the SVG data as a string for the specified template.

**Parameters**:
- `_name` (string): Name of the SVG template

**Returns**:
- `data_` (string): Complete SVG markup as string

**Example Usage**:
```solidity
string memory svgData = ISVGTemplates(diamond).svgString("achievement-badge");

// Use in NFT metadata
string memory metadata = string(abi.encodePacked(
    '{"name":"Achievement Badge","image":"data:image/svg+xml;base64,',
    Base64.encode(bytes(svgData)),
    '"}'
));
```

**Use Cases**:
- NFT metadata generation
- On-chain image rendering
- Dynamic visual content
- Template preview

## Integration Examples

### NFT Metadata Generation
```solidity
// Generate dynamic NFT metadata with SVG templates
contract DynamicNFT {
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        // Get achievement level for this token
        uint256 level = getAchievementLevel(tokenId);
        string memory templateName = string(abi.encodePacked("badge-level-", level.toString()));
        
        // Get SVG data from template
        string memory svgData = ISVGTemplates(diamond).svgString(templateName);
        
        // Encode as base64 for data URI
        string memory imageURI = string(abi.encodePacked(
            "data:image/svg+xml;base64,",
            Base64.encode(bytes(svgData))
        ));
        
        // Build complete metadata
        return string(abi.encodePacked(
            '{"name":"Achievement Badge #', tokenId.toString(),
            '","description":"Dynamic achievement badge",',
            '"image":"', imageURI, '",',
            '"attributes":[{"trait_type":"Level","value":', level.toString(), '}]}'
        ));
    }
}
```

### Template-Based Rendering System
```solidity
// System for rendering different visual elements
contract VisualRenderer {
    mapping(string => string) public templateCategories;
    
    function setTemplateCategory(string memory templateName, string memory category) external onlyOwner {
        templateCategories[templateName] = category;
    }
    
    function renderByCategory(string memory category, bytes memory params) external view returns (string memory) {
        string[] memory allTemplates = ISVGTemplates(diamond).svgs();
        
        for (uint256 i = 0; i < allTemplates.length; i++) {
            if (keccak256(bytes(templateCategories[allTemplates[i]])) == keccak256(bytes(category))) {
                return ISVGTemplates(diamond).svgString(allTemplates[i]);
            }
        }
        
        revert("No template found for category");
    }
    
    function getTemplatesByCategory(string memory category) external view returns (string[] memory) {
        string[] memory allTemplates = ISVGTemplates(diamond).svgs();
        string[] memory categoryTemplates = new string[](allTemplates.length);
        uint256 count = 0;
        
        for (uint256 i = 0; i < allTemplates.length; i++) {
            if (keccak256(bytes(templateCategories[allTemplates[i]])) == keccak256(bytes(category))) {
                categoryTemplates[count] = allTemplates[i];
                count++;
            }
        }
        
        // Resize array to actual count
        string[] memory result = new string[](count);
        for (uint256 i = 0; i < count; i++) {
            result[i] = categoryTemplates[i];
        }
        
        return result;
    }
}
```

### Dynamic Badge System
```solidity
// Create achievement badges with different templates
contract AchievementSystem {
    struct Achievement {
        string name;
        string description;
        string svgTemplate;
        uint256 requiredPoints;
    }
    
    mapping(uint256 => Achievement) public achievements;
    mapping(address => mapping(uint256 => bool)) public userAchievements;
    uint256 public achievementCount;
    
    function createAchievement(
        string memory name,
        string memory description,
        string memory svgTemplate,
        uint256 requiredPoints
    ) external onlyOwner {
        // Verify template exists
        address templateAddr = ISVGTemplates(diamond).svgAddress(svgTemplate);
        require(templateAddr != address(0), "Template not found");
        
        achievements[achievementCount] = Achievement({
            name: name,
            description: description,
            svgTemplate: svgTemplate,
            requiredPoints: requiredPoints
        });
        
        achievementCount++;
        emit AchievementCreated(achievementCount - 1, name, svgTemplate);
    }
    
    function awardAchievement(address user, uint256 achievementId) external onlyOwner {
        require(achievementId < achievementCount, "Invalid achievement");
        require(!userAchievements[user][achievementId], "Already awarded");
        
        userAchievements[user][achievementId] = true;
        
        // Generate badge SVG
        string memory badgeSVG = ISVGTemplates(diamond).svgString(achievements[achievementId].svgTemplate);
        
        emit AchievementAwarded(user, achievementId, badgeSVG);
    }
    
    function getUserBadge(address user, uint256 achievementId) external view returns (string memory) {
        require(userAchievements[user][achievementId], "Achievement not earned");
        return ISVGTemplates(diamond).svgString(achievements[achievementId].svgTemplate);
    }
}
```

### Template Management System
```solidity
// Advanced template management with versioning
contract TemplateManager {
    struct TemplateVersion {
        address templateAddress;
        string version;
        uint256 timestamp;
        bool active;
    }
    
    mapping(string => TemplateVersion[]) public templateVersions;
    mapping(string => uint256) public activeVersionIndex;
    
    function createTemplateVersion(
        string memory templateName,
        string memory version
    ) external onlyOwner {
        // Create new template
        address templateAddr = ISVGTemplates(diamond).createSVG(
            string(abi.encodePacked(templateName, "-", version))
        );
        
        // Store version info
        templateVersions[templateName].push(TemplateVersion({
            templateAddress: templateAddr,
            version: version,
            timestamp: block.timestamp,
            active: false
        }));
        
        emit TemplateVersionCreated(templateName, version, templateAddr);
    }
    
    function activateTemplateVersion(string memory templateName, uint256 versionIndex) external onlyOwner {
        require(versionIndex < templateVersions[templateName].length, "Invalid version");
        
        // Deactivate current version
        if (templateVersions[templateName].length > 0) {
            templateVersions[templateName][activeVersionIndex[templateName]].active = false;
        }
        
        // Activate new version
        templateVersions[templateName][versionIndex].active = true;
        activeVersionIndex[templateName] = versionIndex;
        
        emit TemplateVersionActivated(templateName, versionIndex);
    }
    
    function getActiveTemplate(string memory templateName) external view returns (string memory) {
        require(templateVersions[templateName].length > 0, "Template not found");
        
        uint256 activeIndex = activeVersionIndex[templateName];
        TemplateVersion memory activeVersion = templateVersions[templateName][activeIndex];
        require(activeVersion.active, "No active version");
        
        return ISVGTemplates(diamond).svgString(
            string(abi.encodePacked(templateName, "-", activeVersion.version))
        );
    }
}
```

### On-Chain Art Gallery
```solidity
// Gallery system using SVG templates
contract ArtGallery {
    struct Artwork {
        string title;
        string artist;
        string svgTemplate;
        uint256 price;
        bool forSale;
    }
    
    mapping(uint256 => Artwork) public artworks;
    uint256 public artworkCount;
    
    function addArtwork(
        string memory title,
        string memory artist,
        string memory svgTemplate,
        uint256 price
    ) external onlyOwner {
        // Verify template exists
        string memory svgData = ISVGTemplates(diamond).svgString(svgTemplate);
        require(bytes(svgData).length > 0, "Template not found");
        
        artworks[artworkCount] = Artwork({
            title: title,
            artist: artist,
            svgTemplate: svgTemplate,
            price: price,
            forSale: true
        });
        
        artworkCount++;
        emit ArtworkAdded(artworkCount - 1, title, artist);
    }
    
    function previewArtwork(uint256 artworkId) external view returns (string memory) {
        require(artworkId < artworkCount, "Artwork not found");
        return ISVGTemplates(diamond).svgString(artworks[artworkId].svgTemplate);
    }
    
    function purchaseArtwork(uint256 artworkId) external payable {
        require(artworkId < artworkCount, "Artwork not found");
        require(artworks[artworkId].forSale, "Not for sale");
        require(msg.value >= artworks[artworkId].price, "Insufficient payment");
        
        artworks[artworkId].forSale = false;
        
        // Generate NFT with SVG template
        string memory svgData = ISVGTemplates(diamond).svgString(artworks[artworkId].svgTemplate);
        
        emit ArtworkPurchased(artworkId, msg.sender, svgData);
    }
}
```

## Events

### Template Management Events
```solidity
event SVGTemplateCreated(string name, address template);
```

## Security Considerations

### Access Control
- Only contract owner can set SVG manager
- Only contract owner can create templates
- Manager contract should implement proper access controls

### Template Validation
- Verify SVG manager implements required interface
- Check template existence before operations
- Validate template data integrity

### Manager Security
- SVG manager contract should be audited
- Ensure manager cannot be set to malicious contract
- Consider timelock for manager changes

## Gas Optimization

### Efficient Queries
- Batch template operations when possible
- Cache frequently accessed templates
- Use view functions for read operations

### Storage Optimization
- Delegate storage to manager contract
- Minimize on-chain SVG data when possible
- Use IPFS for large templates if needed

## Error Handling

### Common Errors
- SVG manager not set
- Template not found
- Invalid template name
- Manager interface mismatch

### Best Practices
- Always check template existence
- Validate manager address before setting
- Handle empty template responses

## Testing Considerations

### Unit Tests
- Template creation and retrieval
- Manager setting and validation
- Access control enforcement
- Error condition handling

### Integration Tests
- NFT metadata generation
- Template rendering systems
- Manager upgrade scenarios
- Cross-facet interactions

## Related Documentation

- [ISVG](../interfaces/isvg.md) - SVG interface definitions
- [SVGTemplatesLib](../libraries/svg-templates-lib.md) - SVG utilities
- [StringsLib](../libraries/strings-lib.md) - String manipulation
- [GemforceMinterFacet](./gemforce-minter-facet.md) - NFT minting integration
- [NFT Metadata Guide](../../guides/nft-metadata.md) - Implementation guide

---

*This facet provides SVG template management capabilities within the Gemforce platform, enabling dynamic on-chain graphics generation and template-based visual systems.*