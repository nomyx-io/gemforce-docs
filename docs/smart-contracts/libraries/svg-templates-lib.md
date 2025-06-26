# SVGTemplatesLib Library

## Overview

The [`SVGTemplatesLib`](../../../contracts/libraries/SVGTemplatesLib.sol) library provides core utilities for creating and managing on-chain SVG templates within the Gemforce platform. This library enables dynamic generation of SVG graphics for NFT metadata, supporting template-based rendering with variable substitution and multi-part composition.

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
    
    function generateAttributes(uint256 tokenId) internal view returns (string memory) {
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
        require(assetTemplates[templateName].active, "Template not active");
        require(variableNames.length == variableValues.length, "Variable arrays length mismatch");
        
        AssetInstance storage instance = assetInstances[assetId];
        instance.templateName = templateName;
        instance.lastUpdated = block.timestamp;
        
        // Set variables
        for (uint256 i = 0; i < variableNames.length; i++) {
            instance.variables[variableNames[i]] = variableValues[i];
        }
        
        emit AssetRendered(assetId, templateName);
    }
    
    function updateAssetVariable(
        uint256 assetId,
        string memory variableName,
        string memory variableValue
    ) external {
        AssetInstance storage instance = assetInstances[assetId];
        require(bytes(instance.templateName).length > 0, "Asset does not exist");
        
        instance.variables[variableName] = variableValue;
        instance.lastUpdated = block.timestamp;
        
        emit AssetVariableUpdated(assetId, variableName, variableValue);
    }
    
    function renderAsset(uint256 assetId) external view returns (string memory) {
        AssetInstance storage instance = assetInstances[assetId];
        require(bytes(instance.templateName).length > 0, "Asset does not exist");
        
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        AssetTemplate memory template = assetTemplates[instance.templateName];
        
        // Get template SVG
        string memory templateSVG = SVGTemplatesLib._svgString(
            svgStorage.svgTemplates,
            instance.templateName
        );
        
        // Build replacements array
        Replacement[] memory replacements = new Replacement[](template.requiredVariables.length);
        for (uint256 i = 0; i < template.requiredVariables.length; i++) {
            string memory variable = template.requiredVariables[i];
            replacements[i] = Replacement({
                matchString: string(abi.encodePacked("{{", variable, "}}")),
                replaceString: instance.variables[variable]
            });
        }
        
        // Apply replacements using template contract
        address templateAddress = SVGTemplatesLib._svgAddress(svgStorage.svgTemplates, instance.templateName);
        return ISVGTemplate(templateAddress).buildSVG(replacements);
    }
    
    function getAssetMetadata(uint256 assetId) external view returns (
        string memory templateName,
        string memory category,
        uint256 lastUpdated
    ) {
        AssetInstance storage instance = assetInstances[assetId];
        AssetTemplate memory template = assetTemplates[instance.templateName];
        
        return (
            instance.templateName,
            template.category,
            instance.lastUpdated
        );
    }
    
    function getCategoryTemplates(string memory category) external view returns (string[] memory) {
        return categoryTemplates[category];
    }
    
    function getTemplateVariables(string memory templateName) external view returns (string[] memory) {
        return assetTemplates[templateName].requiredVariables;
    }
    
    modifier onlyGameMaster() {
        // Implementation
        _;
    }
}
```

### Certificate Generator
```solidity
// Dynamic certificate generation with SVG templates
contract CertificateGenerator {
    using SVGTemplatesLib for SVGTemplatesLib.SVGStorage;
    
    struct CertificateType {
        string name;
        string templateName;
        string[] requiredFields;
        address issuer;
        bool active;
    }
    
    struct Certificate {
        uint256 certificateId;
        string certificateType;
        mapping(string => string) fields;
        address recipient;
        uint256 issuedAt;
        bool revoked;
    }
    
    mapping(string => CertificateType) public certificateTypes;
    mapping(uint256 => Certificate) public certificates;
    mapping(address => uint256[]) public recipientCertificates;
    uint256 public nextCertificateId;
    
    event CertificateTypeCreated(string indexed typeName, address indexed issuer);
    event CertificateIssued(uint256 indexed certificateId, address indexed recipient, string typeName);
    event CertificateRevoked(uint256 indexed certificateId);
    
    function createCertificateType(
        string memory typeName,
        string[] memory requiredFields,
        string memory svgTemplate
    ) external returns (string memory templateName) {
        templateName = string(abi.encodePacked("cert_", typeName));
        
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        
        // Create SVG template
        address templateAddress = SVGTemplatesLib._createSVG(
            svgStorage.svgTemplates,
            address(this),
            templateName
        );
        
        // Initialize template with certificate SVG
        ISVGTemplate(templateAddress).clear();
        ISVGTemplate(templateAddress).add(svgTemplate);
        
        // Register certificate type
        certificateTypes[typeName] = CertificateType({
            name: typeName,
            templateName: templateName,
            requiredFields: requiredFields,
            issuer: msg.sender,
            active: true
        });
        
        emit CertificateTypeCreated(typeName, msg.sender);
    }
    
    function issueCertificate(
        string memory typeName,
        address recipient,
        string[] memory fieldNames,
        string[] memory fieldValues
    ) external returns (uint256 certificateId) {
        CertificateType memory certType = certificateTypes[typeName];
        require(certType.active, "Certificate type not active");
        require(msg.sender == certType.issuer, "Not authorized issuer");
        require(fieldNames.length == fieldValues.length, "Field arrays length mismatch");
        
        certificateId = nextCertificateId++;
        
        Certificate storage cert = certificates[certificateId];
        cert.certificateId = certificateId;
        cert.certificateType = typeName;
        cert.recipient = recipient;
        cert.issuedAt = block.timestamp;
        cert.revoked = false;
        
        // Set certificate fields
        for (uint256 i = 0; i < fieldNames.length; i++) {
            cert.fields[fieldNames[i]] = fieldValues[i];
        }
        
        recipientCertificates[recipient].push(certificateId);
        
        emit CertificateIssued(certificateId, recipient, typeName);
    }
    
    function revokeCertificate(uint256 certificateId) external {
        Certificate storage cert = certificates[certificateId];
        CertificateType memory certType = certificateTypes[cert.certificateType];
        
        require(msg.sender == certType.issuer, "Not authorized issuer");
        require(!cert.revoked, "Certificate already revoked");
        
        cert.revoked = true;
        emit CertificateRevoked(certificateId);
    }
    
    function generateCertificateSVG(uint256 certificateId) external view returns (string memory) {
        Certificate storage cert = certificates[certificateId];
        require(cert.issuedAt > 0, "Certificate does not exist");
        require(!cert.revoked, "Certificate revoked");
        
        CertificateType memory certType = certificateTypes[cert.certificateType];
        SVGTemplatesLib.SVGStorage storage svgStorage = SVGTemplatesLib.svgStorage();
        
        // Get template SVG
        string memory templateSVG = SVGTemplatesLib._svgString(
            svgStorage.svgTemplates,
            certType.templateName
        );
        
        // Build replacements for certificate fields
        Replacement[] memory replacements = new Replacement[](certType.requiredFields.length + 4);
        
        // Standard certificate fields
        replacements[0] = Replacement("{{CERTIFICATE_ID}}", Strings.toString(certificateId));
        replacements[1] = Replacement("{{RECIPIENT}}", Strings.toHexString(uint160(cert.recipient), 20));
        replacements[2] = Replacement("{{ISSUE_DATE}}", formatTimestamp(cert.issuedAt));
        replacements[3] = Replacement("{{CERTIFICATE_TYPE}}", cert.certificateType);
        
        // Custom fields
        for (uint256 i = 0; i < certType.requiredFields.length; i++) {
            string memory fieldName = certType.requiredFields[i];
            replacements[i + 4] = Replacement({
                matchString: string(abi.encodePacked("{{", fieldName, "}}")),
                replaceString: cert.fields[fieldName]
            });
        }
        
        // Apply replacements
        address templateAddress = SVGTemplatesLib._svgAddress(svgStorage.svgTemplates, certType.templateName);
        return ISVGTemplate(templateAddress).buildSVG(replacements);
    }
    
    function getCertificateMetadata(uint256 certificateId) external view returns (
        string memory certificateType,
        address recipient,
        uint256 issuedAt,
        bool revoked
    ) {
        Certificate storage cert = certificates[certificateId];
        return (
            cert.certificateType,
            cert.recipient,
            cert.issuedAt,
            cert.revoked
        );
    }
    
    function getRecipientCertificates(address recipient) external view returns (uint256[] memory) {
        return recipientCertificates[recipient];
    }
    
    function formatTimestamp(uint256 timestamp) internal pure returns (string memory) {
        // Simple timestamp formatting - in production, use a proper date library
        return Strings.toString(timestamp);
    }
}
```

## Events

### SVG Template Events
```solidity
event SVGTemplateCreated(string name, address template);
```

## Security Considerations

### Template Deployment Security
- CREATE2 ensures deterministic addressing
- Ownership transfer to template creator
- Unique name validation prevents conflicts
- Address verification ensures deployment integrity

### Access Control
- Template ownership controls content updates
- Manager role for SVG system administration
- Template-specific permissions for modifications
- Secure template registry management

### Content Validation
- SVG content validation before storage
- Safe string replacement operations
- Protection against malicious template content
- Graceful error handling for missing templates

## Gas Optimization

### Storage Efficiency
- Efficient template name storage
- Minimal storage writes during creation
- Optimized address calculation
- Batch template operations where possible

### Rendering Efficiency
- Lazy template loading
- Efficient string operations
- Minimal external calls
- Optimized replacement algorithms

## Error Handling

### Common Errors
- "template already deployed" - Duplicate template name
- "template address mismatch" - CREATE2 deployment failure
- Template not found - Graceful empty string return
- Invalid template content - Validation failures

### Best Practices
- Validate template names before creation
- Check template existence before rendering
- Handle missing templates gracefully
- Provide clear error messages for debugging

## Testing Considerations

### Unit Tests
- Template creation and deployment
- Address calculation accuracy
- SVG content retrieval
- Ownership transfer functionality

### Integration Tests
- Multi-template rendering workflows
- Variable substitution accuracy
- Template registry management
- Cross-contract template usage

## Related Documentation

- [ISVG Interface](../interfaces/isvg.md) - SVG interface definitions
- [SVGTemplatesFacet](../facets/svg-templates-facet.md) - SVG templates facet implementation
- [StringsLib](strings-lib.md) - String manipulation utilities
- [NFT Metadata Guide](../../guides/nft-metadata.md) - NFT metadata best practices
- [SVG Generation Guide](../../guides/svg-generation.md) - SVG template creation guide

---

*This library provides comprehensive utilities for on-chain SVG template management within the Gemforce platform, enabling dynamic generation of graphics for NFTs, certificates, gaming assets, and other visual content with variable substitution and multi-part composition capabilities.*