# SVGTemplatesLib Library

## Overview

The [`SVGTemplatesLib`](../../smart-contracts/libraries/svg-templates-lib.md) library provides core utilities for creating and managing on-chain SVG templates within the Gemforce platform. This library enables dynamic generation of SVG graphics for NFT metadata, supporting template-based rendering with variable substitution and multi-part composition.

## Key Features

- **On-Chain SVG Storage**: Store SVG templates directly on the blockchain
- **Deterministic Deployment**: CREATE2-based template contract deployment
- **Template Management**: Named template registry with unique addressing
- **Dynamic Rendering**: Variable substitution for customizable graphics
- **Multi-Part Support**: Composition of complex SVG graphics from components
- **Ownership Transfer**: Template ownership management for creators

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.6;

library SVGTemplatesLib {
    // Storage management
    function svgStorage() internal pure returns (SVGStorage storage);
    
    // Template management
    function _svgs(SVGTemplatesContract storage self) internal view returns (string[] memory);
    function _svgAddress(SVGTemplatesContract storage, string memory _name) internal view returns (address);
    function _svgString(SVGTemplatesContract storage self, string memory _name) internal view returns (string memory);
    function _svgData(SVGTemplatesContract storage self, string memory _name) internal view returns (address);
    
    // Template creation
    function _createSVG(SVGTemplatesContract storage self, address sender, string memory _name) internal returns (address);
}
```

## Data Structures

### SVGTemplatesContract Struct
```solidity
struct SVGTemplatesContract {
    mapping(string => address) _templates;      // Template name to contract address mapping
    string[] _templateNames;                    // Array of all template names
}
```

**Purpose**: Core storage structure for managing SVG template contracts.

**Components**:
- **_templates**: Maps template names to deployed contract addresses
- **_templateNames**: Maintains ordered list of all template names

### SaltStorage Struct
```solidity
struct SaltStorage {
    uint256 salt;                              // Salt for CREATE2 operations
}
```

**Purpose**: Storage for CREATE2 salt values to ensure deterministic addressing.

### MultiPartContract Struct
```solidity
struct MultiPartContract {
    string name_;                              // Name of the multi-part template
    bytes[] data_;                             // Array of SVG component data
}
```

**Purpose**: Storage structure for multi-part SVG compositions.

### SVGStorage Struct
```solidity
struct SVGStorage {
    SVGTemplatesContract svgTemplates;         // Main template storage
    SaltStorage salt;                          // CREATE2 salt storage
    address svgManager;                        // SVG manager contract address
    MultiPartContract multiPart;               // Multi-part template storage
}
```

**Purpose**: Complete Diamond storage structure for SVG functionality.

**Storage Position**:
```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nextblock.bitgem.app.SVGStorage.storage");
```

### Replacement Struct
```solidity
struct Replacement {
    string matchString;                        // String to find in template
    string replaceString;                      // String to replace with
}
```

**Purpose**: Variable substitution data for dynamic SVG rendering.

## Core Functions

### Storage Management

#### `svgStorage()`
```solidity
function svgStorage() internal pure returns (SVGStorage storage ds)
```

**Purpose**: Access Diamond storage for SVG template data using assembly.

**Implementation**:
```solidity
function svgStorage() internal pure returns (SVGStorage storage ds) {
    bytes32 position = DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
}
```

**Returns**: Storage reference to SVG template data

**Usage**: All SVG functions use this to access persistent storage.

### Template Management

#### `_svgs()`
```solidity
function _svgs(SVGTemplatesContract storage self) internal view returns (string[] memory)
```

**Purpose**: Retrieve all template names stored in the contract.

**Parameters**:
- `self`: Storage reference to SVG templates contract

**Returns**: Array of all template names

**Example Usage**:
```solidity
// Get all available template names
string[] memory templateNames = SVGTemplatesLib._svgs(svgTemplates);
console.log("Available templates:", templateNames.length);
```

#### `_svgAddress()`
```solidity
function _svgAddress(
    SVGTemplatesContract storage,
    string memory _name
) internal view returns (address)
```

**Purpose**: Calculate the deterministic address for a template contract.

**Parameters**:
- `_name`: Name of the template

**Implementation**:
```solidity
return Create2.computeAddress(
    keccak256(abi.encodePacked(_name)), 
    keccak256(type(SVGTemplate).creationCode)
);
```

**Returns**: Predicted contract address for the named template

**Benefits**:
- Enables address prediction before deployment
- Supports deterministic template addressing
- Allows for efficient template lookup

#### `_svgString()`
```solidity
function _svgString(
    SVGTemplatesContract storage self,
    string memory _name
) internal view returns (string memory data_)
```

**Purpose**: Retrieve the SVG string content from a deployed template.

**Parameters**:
- `self`: Storage reference to SVG templates contract
- `_name`: Name of the template

**Implementation**:
```solidity
try SVGTemplate(_svgAddress(self, _name)).svgString() returns (string memory _data) {
    data_ = _data;
} catch (bytes memory) {}
```

**Returns**: SVG string content or empty string if template doesn't exist

**Error Handling**: Uses try-catch to gracefully handle non-existent templates

**Example Usage**:
```solidity
// Get SVG content for rendering
string memory svgContent = SVGTemplatesLib._svgString(svgTemplates, "avatar_base");
if (bytes(svgContent).length > 0) {
    // Process SVG content
}
```

#### `_svgData()`
```solidity
function _svgData(
    SVGTemplatesContract storage self,
    string memory _name
) internal view returns (address)
```

**Purpose**: Get the stored contract address for a named template.

**Parameters**:
- `self`: Storage reference to SVG templates contract
- `_name`: Name of the template

**Returns**: Contract address of the template or zero address if not found

**Example Usage**:
```solidity
// Check if template exists
address templateAddress = SVGTemplatesLib._svgData(svgTemplates, "background");
require(templateAddress != address(0), "Template not found");
```

### Template Creation

#### `_createSVG()`
```solidity
function _createSVG(
    SVGTemplatesContract storage self, 
    address sender, 
    string memory _name
) internal returns (address _tplAddress)
```

**Purpose**: Deploy a new SVG template contract with deterministic addressing.

**Parameters**:
- `self`: Storage reference to SVG templates contract
- `sender`: Address that will own the template
- `_name`: Unique name for the template

**Process**:
1. Verify template name is unique
2. Calculate target address using CREATE2
3. Deploy template contract with deterministic salt
4. Verify deployment address matches prediction
5. Transfer ownership to sender
6. Update storage with new template

**Security Features**:
- Prevents duplicate template names
- Verifies deployment integrity
- Transfers ownership to creator

**Example Usage**:
```solidity
// Create a new SVG template
address templateAddress = SVGTemplatesLib._createSVG(
    svgTemplates,
    msg.sender,
    "character_base"
);

// Template is now deployed and owned by msg.sender
console.log("Template deployed at:", templateAddress);
```

## Integration Examples

### NFT Avatar System
```solidity
// NFT collection with dynamic SVG avatars
contract AvatarNFT {
    using SVGTemplatesLib for SVGTemplatesLib.SVGStorage;
    
    struct AvatarTraits {
        string background;
        string body;
        string eyes;
        string mouth;
        string accessory;
        uint256 colorScheme;
    }
    
    mapping(uint256 => AvatarTraits) public avatarTraits;
    mapping(string => string[]) public traitOptions;
    
    event TemplateCreated(string indexed templateName, address templateAddress);
    event AvatarGenerated(uint256 indexed tokenId, AvatarTraits traits);
    
    function initializeTemplates() external onlyOwner {
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        
        // Create base templates for avatar components
        string[] memory templateNames = new string[](5);
        templateNames[0] = "background_template";
        templateNames[1] = "body_template";
        templateNames[2] = "eyes_template";
        templateNames[3] = "mouth_template";
        templateNames[4] = "accessory_template";
        
        for (uint256 i = 0; i < templateNames.length; i++) {
            address templateAddress = SVGTemplatesLib._createSVG(
                svgStorage.svgTemplates,
                address(this),
                templateNames[i]
            );
            emit TemplateCreated(templateNames[i], templateAddress);
        }
        
        // Initialize trait options
        traitOptions["background"] = ["forest", "ocean", "space", "city"];
        traitOptions["body"] = ["human", "robot", "alien", "animal"];
        traitOptions["eyes"] = ["normal", "glowing", "mechanical", "large"];
        traitOptions["mouth"] = ["smile", "frown", "neutral", "fangs"];
        traitOptions["accessory"] = ["hat", "glasses", "necklace", "none"];
    }
    
    function generateAvatar(uint256 tokenId, uint256 seed) external {
        require(_exists(tokenId), "Token does not exist");
        
        // Generate random traits based on seed
        AvatarTraits memory traits = AvatarTraits({
            background: traitOptions["background"][seed % traitOptions["background"].length],
            body: traitOptions["body"][(seed / 10) % traitOptions["body"].length],
            eyes: traitOptions["eyes"][(seed / 100) % traitOptions["eyes"].length],
            mouth: traitOptions["mouth"][(seed / 1000) % traitOptions["mouth"].length],
            accessory: traitOptions["accessory"][(seed / 10000) % traitOptions["accessory"].length],
            colorScheme: seed % 10
        });
        
        avatarTraits[tokenId] = traits;
        emit AvatarGenerated(tokenId, traits);
    }
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "Token does not exist");
        
        string memory svgContent = generateSVG(tokenId);
        string memory json = Base64.encode(
            bytes(
                string(
                    abi.encodePacked(
                        '{"name": "Avatar #',
                        Strings.toString(tokenId),
                        '", "description": "Dynamic SVG Avatar", "image": "data:image/svg+xml;base64,',
                        Base64.encode(bytes(svgContent)),
                        '", "attributes": ',
                        generateAttributes(tokenId),
                        '}'
                    )
                )
            )
        );
        
        return string(abi.encodePacked("data:application/json;base64,", json));
    }
    
    function generateSVG(uint256 tokenId) public view returns (string memory) {
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        AvatarTraits memory traits = avatarTraits[tokenId];
        
        // Get base templates
        string memory backgroundSVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, "background_template");
        string memory bodySVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, "body_template");
        string memory eyesSVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, "eyes_template");
        string memory mouthSVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, "mouth_template");
        string memory accessorySVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, "accessory_template");
        
        // Create replacements for dynamic content
        Replacement[] memory replacements = new Replacement[](6);
        replacements[0] = Replacement("{{BACKGROUND_TYPE}}", traits.background);
        replacements[1] = Replacement("{{BODY_TYPE}}", traits.body);
        replacements[2] = Replacement("{{EYES_TYPE}}", traits.eyes);
        replacements[3] = Replacement("{{MOUTH_TYPE}}", traits.mouth);
        replacements[4] = Replacement("{{ACCESSORY_TYPE}}", traits.accessory);
        replacements[5] = Replacement("{{COLOR_SCHEME}}", Strings.toString(traits.colorScheme));
        
        // Build complete SVG
        return string(
            abi.encodePacked(
                '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 400">',
                applyReplacements(backgroundSVG, replacements),
                applyReplacements(bodySVG, replacements),
                applyReplacements(eyesSVG, replacements),
                applyReplacements(mouthSVG, replacements),
                applyReplacements(accessorySVG, replacements),
                '</svg>'
            )
        );
    }
    
    function generateAttributes(uint256 tokenId) public view returns (string memory) {
        AvatarTraits memory traits = avatarTraits[tokenId];
        
        return string(
            abi.encodePacked(
                '[',
                '{"trait_type": "Background", "value": "', traits.background, '"},',
                '{"trait_type": "Body", "value": "', traits.body, '"},',
                '{"trait_type": "Eyes", "value": "', traits.eyes, '"},',
                '{"trait_type": "Mouth", "value": "', traits.mouth, '"},',
                '{"trait_type": "Accessory", "value": "', traits.accessory, '"},',
                '{"trait_type": "Color Scheme", "value": ', Strings.toString(traits.colorScheme), '}',
                ']'
            )
        );
    }
    
    function applyReplacements(string memory template, Replacement[] memory replacements) internal pure returns (string memory) {
        string memory result = template;
        for (uint256 i = 0; i < replacements.length; i++) {
            result = StringsLib.replace(result, replacements[i].matchString, replacements[i].replaceString);
        }
        return result;
    }
    
    modifier onlyOwner() {
        // Implementation
        _;
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        // Implementation
        return true;
    }
}
```

### Gaming Asset Renderer
```solidity
// Dynamic SVG rendering for gaming assets
contract GameAssetRenderer {
    using SVGTemplatesLib for SVGTemplatesLib.SVGStorage;
    
    struct AssetTemplate {
        string name;
        string category;
        string[] requiredVariables;
        bool active;
    }
    
    struct AssetInstance {
        string templateName;
        mapping(string => string) variables;
        uint256 lastUpdated;
    }
    
    mapping(string => AssetTemplate) public assetTemplates;
    mapping(uint256 => AssetInstance) public assetInstances;
    mapping(string => string[]) public categoryTemplates;
    
    event AssetTemplateRegistered(string indexed templateName, string category);
    event AssetRendered(uint256 indexed assetId, string templateName);
    event AssetVariableUpdated(uint256 indexed assetId, string variable, string value);
    
    function registerAssetTemplate(
        string memory templateName,
        string memory category,
        string[] memory requiredVariables,
        string memory svgTemplate
    ) external onlyGameMaster {
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        
        // Create SVG template contract
        address templateAddress = SVGTemplatesLib._createSVG(
            svgStorage.svgTemplates,
            address(this),
            templateName
        );
        
        // Initialize template with SVG content
        ISVGTemplate(templateAddress).clear();
        ISVGTemplate(templateAddress).add(svgTemplate);
        
        // Register template metadata
        assetTemplates[templateName] = AssetTemplate({
            name: templateName,
            category: category,
            requiredVariables: requiredVariables,
            active: true
        });
        
        categoryTemplates[category].push(templateName);
        
        emit AssetTemplateRegistered(templateName, category);
    }
    
    function createAsset(
        uint256 assetId,
        string memory templateName,
        string[] memory variableNames,
        string[] memory variableValues
    ) external {
        AssetTemplate storage template = assetTemplates[templateName];
        require(bytes(template.name).length > 0, "Template not found");
        require(template.active, "Template not active");
        require(variableNames.length == variableValues.length, "Variable arrays length mismatch");
        
        AssetInstance storage instance = assetInstances[assetId];
        instance.templateName = templateName;
        instance.lastUpdated = block.timestamp;
        
        for (uint256 i = 0; i < variableNames.length; i++) {
            instance.variables[variableNames[i]] = variableValues[i];
        }
        emit AssetRendered(assetId, templateName);
    }
    
    function updateAssetVariable(uint256 assetId, string memory variableName, string memory value) external {
        AssetInstance storage instance = assetInstances[assetId];
        require(bytes(instance.templateName).length > 0, "Asset not found"); // Check if asset exists
        
        instance.variables[variableName] = value;
        instance.lastUpdated = block.timestamp;
        
        emit AssetVariableUpdated(assetId, variableName, value);
    }
    
    function getAssetSVG(uint256 assetId) external view returns (string memory) {
        AssetInstance memory instance = assetInstances[assetId];
        require(bytes(instance.templateName).length > 0, "Asset not found");
        
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        string memory templateSVG = SVGTemplatesLib._svgString(svgStorage.svgTemplates, instance.templateName);
        
        Replacement[] memory replacements = new Replacement[](instance.variables.length); // Dynamic array
        uint256 i = 0;
        // Iterate through mapping (not directly possible, would require storing keys)
        // For demonstration, assume keys are known or passed
        
        // Placeholder for variable substitution:
        string memory result = templateSVG;
        // In reality, iterate over stored instance.variables and replace
        
        return result;
    }
    
    function getTemplateListByCategory(string memory category) external view returns (string[] memory) {
        return categoryTemplates[category];
    }
    
    modifier onlyGameMaster() {
        // Placeholder for access control
        _;
    }
}
```

### Certificate Generator
```solidity
// On-chain certificate generation with dynamic SVG and data
contract CertificateGenerator {
    using SVGTemplatesLib for SVGTemplatesLib.SVGStorage;
    
    struct CertificateConfig {
        string templateName;
        string[] requiredFields;
        bool active;
    }
    
    mapping(string => CertificateConfig) public certificateConfigs;
    
    event CertificateConfigured(string indexed configName, string templateName);
    event CertificateGenerated(uint256 indexed certificateId, string configName, address indexed recipient);
    
    function configureCertificate(
        string memory configName,
        string memory templateName,
        string[] memory requiredFields,
        string memory svgTemplate
    ) external onlyOwner {
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        
        // Create or update SVG template
        address templateAddress = SVGTemplatesLib._createSVG(
            svgStorage.svgTemplates,
            address(this),
            templateName
        );
        ISVGTemplate(templateAddress).clear();
        ISVGTemplate(templateAddress).add(svgTemplate);
        
        certificateConfigs[configName] = CertificateConfig({
            templateName: templateName,
            requiredFields: requiredFields,
            active: true
        });
        
        emit CertificateConfigured(configName, templateName);
    }
    
    function generateCertificate(
        string memory configName,
        uint256 certificateId,
        address recipient,
        string[] memory fieldNames,
        string[] memory fieldValues
    ) external returns (string memory) {
        CertificateConfig storage config = certificateConfigs[configName];
        require(config.active, "Certificate config not active");
        require(fieldNames.length == fieldValues.length, "Field arrays mismatch");
        
        // Validate required fields
        for (uint256 i = 0; i < config.requiredFields.length; i++) {
            bool found = false;
            for (uint256 j = 0; j < fieldNames.length; j++) {
                if (keccak256(abi.encodePacked(config.requiredFields[i])) == keccak256(abi.encodePacked(fieldNames[j]))) {
                    found = true;
                    break;
                }
            }
            require(found, "Missing required field");
        }
        
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        string memory svgTemplate = SVGTemplatesLib._svgString(svgStorage.svgTemplates, config.templateName);
        
        Replacement[] memory replacements = new Replacement[](fieldNames.length);
        for (uint256 i = 0; i < fieldNames.length; i++) {
            replacements[i] = Replacement(
                string(abi.encodePacked("{{", fieldNames[i], "}}")), // e.g., {{RECIPIENT_NAME}}
                fieldValues[i]
            );
        }
        
        string memory finalSVG = applyReplacements(svgTemplate, replacements);
        
        // Mint NFT representing the certificate if applicable
        // (ERC721 or ERC1155 minting logic here)
        
        emit CertificateGenerated(certificateId, configName, recipient);
        
        return finalSVG;
    }
    
    function applyReplacements(string memory template, Replacement[] memory replacements) internal pure returns (string memory) {
        string memory result = template;
        for (uint256 i = 0; i < replacements.length; i++) {
            result = StringsLib.replace(result, replacements[i].matchString, replacements[i].replaceString);
        }
        return result;
    }
    
    modifier onlyOwner() {
        // Implementation
        _;
    }
}
```

## Security Considerations

### Template Deployment Security
- **Access Control**: Only authorized parties should be able to create new SVG templates.
- **Input Validation**: Validate template names and SVG content for malicious code.
- **Deterministic Addresses**: Reliance on CREATE2 ensures predictable addresses, which can be vulnerable if salt generation is not unique/random enough.

### Access Control
- **Owner-Only Functions**: Sensitive manager functions (e.g., setting `svgManager` address) should be restricted to the contract owner.
- **Template Ownership**: Templates should be owned by the facet or a trusted entity, not external users.

### Content Validation
- **SVG Sanitization**: Ensure only safe SVG elements and attributes are allowed.
- **XSS Prevention**: Prevent Cross-Site Scripting (XSS) if SVG is rendered in a web context.

## Gas Optimization

### Storage Efficiency
- The `SVGStorage` struct uses a custom storage slot pattern to avoid storage collisions.
- Templates are stored as separate contracts, minimizing the main diamond's storage footprint.

### Rendering Efficiency
- Dynamic SVG generation on-chain minimizes data stored directly in NFT metadata.
- `StringsLib.replace` for string manipulation is comparatively expensive; ensure replacements are minimal for gas.

## Error Handling

### Common Errors
- `SVSTLib: Invalid name`: Template name is empty or already exists.
- `SVSTLib: Template not found`: Attempting to use a non-existent template.
- `SVSTLib: Deployment failed`: CREATE2 deployment failed.
- `SVSTLib: Not template owner`: Unauthorized attempt to modify template.

## Best Practices

### Template Design
- Design SVG templates to be modular and reusable.
- Use placeholders (`{{VARIABLE_NAME}}`) for dynamic content.
- Keep templates as small as possible to minimize gas costs.

### Development Guidelines
- Thoroughly test SVG rendering logic off-chain before deployment.
- Implement robust access control for all template management functions.
- Monitor events for template creation and updates.

### Integration Checklist
- Ensure off-chain rendering applications properly handle SVG rendering from `data:image/svg+xml;base64` URIs.
- Provide clear documentation for template variable names and expected types.

## Related Documentation

- [SVGTemplatesFacet](../../smart-contracts/facets/svg-templates-facet.md) - Reference for the SVGTemplates Facet implementation.
- [ISVG Interface](../../smart-contracts/interfaces/isvg.md) - Interface definition.
- [NFT Metadata Standards](../../developer-guides/development-environment-setup.md)
- [SVG Generation Tools & Libraries](https://github.com/darenmurph/awesome-svg) (External)