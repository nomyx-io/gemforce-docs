# MetadataLib Library

## Overview

[`MetadataLib`](/Users/sschepis/Development/gem-base/contracts/libraries/MetadataLib.sol) is a comprehensive NFT metadata management library that handles the generation, storage, and retrieval of token metadata in compliance with ERC721 standards. This library provides advanced features for dynamic metadata generation, SVG image processing, attribute management, and JSON metadata formatting.

## Key Features

- **Dynamic Metadata Generation**: Generate metadata on-chain with customizable attributes
- **SVG Image Integration**: Advanced SVG template processing and color replacement
- **Attribute Management**: Integration with AttributeLib for token-specific attributes
- **JSON Formatting**: Standards-compliant JSON metadata generation
- **Base64 Encoding**: Automatic encoding for data URIs
- **Contract-Level Metadata**: Support for collection-level metadata
- **Template System**: Flexible template replacement for dynamic content

## Storage Structure

### MetadataStorage

```solidity
struct MetadataStorage {
    MetadataContract metadata;
}
```

### Storage Position

```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nextblock.bitgem.app.MetadataStorage.storage");
```

## Core Functions

### Storage Access

#### `metadataStorage()`
```solidity
function metadataStorage() internal pure returns (MetadataStorage storage ds)
```
Returns the metadata storage struct using assembly for gas efficiency.

#### `setMetadata()`
```solidity
function setMetadata(MetadataContract storage, MetadataContract memory _contract) internal
```
Sets the metadata contract configuration.

### Basic Metadata Properties

#### `name()`
```solidity
function name(MetadataContract storage self) internal view returns (string memory)
```
Returns the collection name.

#### `symbol()`
```solidity
function symbol(MetadataContract storage self) internal view returns (string memory)
```
Returns the collection symbol.

#### `description()`
```solidity
function description(MetadataContract storage self) internal view returns (string memory)
```
Returns the collection description.

#### `image()`
```solidity
function image(MetadataContract storage self) internal view returns (string memory)
```
Returns the collection image as SVG.

### Image Management

#### `getContractImage()`
```solidity
function getContractImage(
    MetadataContract storage self
) internal view returns (address addr, string memory svg)
```

Generates the contract-level image using SVG templates and color replacements.

**Returns:**
- `addr`: Address of the SVG template contract
- `svg`: Generated SVG string

**Features:**
- **Template Resolution**: Resolves SVG templates by name
- **Color Replacement**: Applies color palette to templates
- **Fallback Handling**: Returns image name if template resolution fails

#### `getTokenImage()`
```solidity
function getTokenImage(
    MetadataContract storage,
    DiamondContract storage,
    AttributeContract storage a,
    uint256 tokenId
) internal view returns (string memory svg, Attribute[] memory attributes)
```

Generates token-specific image and retrieves associated attributes.

**Parameters:**
- `tokenId`: ID of the token

**Returns:**
- `svg`: Generated or stored SVG for the token
- `attributes`: Array of token attributes

#### `setTokenImage()`
```solidity
function setTokenImage(
    MetadataContract storage,
    AttributeContract storage a,
    uint256 tokenId,
    string memory svg
) internal
```

Sets a custom image for a specific token.

**Parameters:**
- `tokenId`: ID of the token
- `svg`: SVG content to store

### Configuration Management

#### `_setMetadataSource()`
```solidity
function _setMetadataSource(
    MetadataContract storage self,
    MetadataSource source
) internal
```

Sets the metadata source type (on-chain, IPFS, HTTP, etc.).

#### `_setBaseURI()`
```solidity
function _setBaseURI(MetadataContract storage self, string memory baseURI) internal
```

Sets the base URI for external metadata sources.

### URI Generation

#### `tokenURI()`
```solidity
function tokenURI(
    MetadataContract storage self,
    DiamondContract storage diamond,
    AttributeContract storage attribs,
    uint256 tokenId
) internal view returns (string memory)
```

Generates the complete token URI with metadata JSON.

**Process:**
1. Retrieves token image and attributes
2. Converts attributes to traits
3. Generates JSON metadata
4. Base64 encodes the result
5. Returns data URI

**Features:**
- **Dynamic Naming**: Adds "Claim" suffix for claim tokens
- **Attribute Integration**: Includes all token attributes as traits
- **Standards Compliance**: ERC721 metadata standard compliant

#### `contractURI()`
```solidity
function contractURI(MetadataContract storage self) internal view returns (string memory)
```

Generates contract-level metadata URI for collection information.

## Metadata Generation

### Trait Creation

#### `createTrait()`
```solidity
function createTrait(
    string memory displayType,
    string memory key,
    string memory value
) internal pure returns (string memory trait)
```

Creates a JSON trait object for metadata attributes.

**Parameters:**
- `displayType`: Display type for the trait (e.g., "number", "percentage")
- `key`: Trait name
- `value`: Trait value

**Features:**
- **Type Handling**: Automatic value formatting based on display type
- **Number Detection**: Unquoted values for numeric traits
- **Validation**: Ensures key is not empty

#### `arrayizeTraits()`
```solidity
function arrayizeTraits(Trait[] memory _traits) internal pure returns (string memory _traitsString)
```

Converts an array of trait structs into a JSON array string.

**Features:**
- **JSON Formatting**: Proper JSON array formatting
- **Comma Handling**: Correct comma placement between traits
- **Empty Array Support**: Handles empty trait arrays

### Complete Metadata Generation

#### `getTokenMetadata()`
```solidity
function getTokenMetadata(
    MetadataContract memory definition,
    Trait[] memory _traits,
    string memory _imageData
) internal pure returns (string memory metadata)
```

Generates complete JSON metadata for a token.

**Parameters:**
- `definition`: Metadata contract definition
- `_traits`: Array of token traits
- `_imageData`: Image data (SVG or URL)

**Generated Fields:**
- `name`: Token name
- `image`: Token image
- `description`: Token description
- `external_url`: External URL (if configured)
- `attributes`: Token attributes array

## Template Processing

### SVG Template Integration

#### `_getReplacements()`
```solidity
function _getReplacements(
    string[] memory colorPalette,
    string[] memory cccc
) internal pure returns (Replacement[] memory)
```

Creates replacement arrays for SVG template processing.

**Parameters:**
- `colorPalette`: Array of color values
- `cccc`: Array of attribute values (CUT, COLOR, CARAT, CLARITY)

**Returns:**
- Array of replacement objects for template processing

**Template Variables:**
- `COLOR 0`, `COLOR 1`, etc.: Color palette values
- `CUT`: Cut attribute value
- `COLOR`: Color attribute value
- `CARAT`: Carat attribute value
- `CLARITY`: Clarity attribute value

## Data Structures

### Trait Structure

```solidity
struct Trait {
    string displayType;
    string key;
    string value;
}
```

### Replacement Structure

```solidity
struct Replacement {
    string matchString;
    string replaceString;
}
```

### Attribute Structure

```solidity
struct Attribute {
    string key;
    AttributeType attributeType;
    string value;
}
```

## Usage Examples

### Basic Token URI Generation

```solidity
function getTokenURI(uint256 tokenId) external view returns (string memory) {
    MetadataStorage storage ms = MetadataLib.metadataStorage();
    DiamondStorage storage ds = DiamondLib.diamondStorage();
    AttributeStorage storage as = AttributeLib.attributeStorage();
    
    return MetadataLib.tokenURI(ms.metadata, ds, as, tokenId);
}
```

### Custom Trait Creation

```solidity
function createCustomTraits(uint256 tokenId) internal view returns (Trait[] memory) {
    Trait[] memory traits = new Trait[](3);
    
    traits[0] = Trait("", "Rarity", "Legendary");
    traits[1] = Trait("number", "Level", "42");
    traits[2] = Trait("percentage", "Completion", "85");
    
    return traits;
}
```

### Dynamic Image Generation

```solidity
function generateTokenImage(uint256 tokenId) internal view returns (string memory) {
    MetadataStorage storage ms = MetadataLib.metadataStorage();
    DiamondStorage storage ds = DiamondLib.diamondStorage();
    AttributeStorage storage as = AttributeLib.attributeStorage();
    
    (string memory svg, ) = MetadataLib.getTokenImage(ms.metadata, ds, as, tokenId);
    return svg;
}
```

### Contract Metadata Setup

```solidity
function setupContractMetadata() internal {
    MetadataContract memory metadata = MetadataContract({
        _name: "Gemforce Collection",
        _symbol: "GEM",
        _description: "A collection of unique digital gems",
        _imageName: "gem_template",
        _imageColors: ["#FF0000", "#00FF00", "#0000FF"],
        _externalUri: "https://gemforce.io",
        _baseUri: "https://api.gemforce.io/metadata/",
        _metadataSource: MetadataSource.OnChain
    });
    
    MetadataLib.setMetadata(metadataStorage().metadata, metadata);
}
```

## Integration Points

### With AttributeLib

```solidity
// Get token attributes for metadata
string[] memory keys = AttributeLib._getAttributeKeys(attributeStorage, tokenId);
string[] memory values = AttributeLib._getAttributeValues(tokenId);
```

### With SVGTemplatesLib

```solidity
// Generate SVG with template replacement
SVGManager mgr = SVGManager(SVGTemplatesLib.svgStorage().svgManager);
address templateAddr = mgr.svgAddress(imageName);
string memory svg = ISVGTemplate(templateAddr).buildSVG(replacements);
```

### With StringsLib

```solidity
// String manipulation for metadata processing
bool isHex = StringsLib.startsWith(imageData, "0x");
string memory processed = StringsLib.replace(template, "{{value}}", replacement);
```

## Special Features

### Claim Token Detection

The library automatically detects claim tokens and modifies the name:

```solidity
if (
    TYPE_HASH == keccak256(bytes(attributes[i].key)) &&
    CLAIM_HASH == keccak256(bytes(attributes[i].value))
) {
    _name = string(abi.encodePacked(_name, " Claim"));
}
```

### Image Format Detection

Supports both SVG content and hex-encoded data:

```solidity
if(!_imageData.startsWith("0x")) {
    _imageData = string(abi.encodePacked('"', _imageData, '"'));
}
```

### Dynamic Color Replacement

Supports dynamic color palette application to SVG templates:

```solidity
string[] memory colors = ["#FF0000", "#00FF00", "#0000FF"];
Replacement[] memory replacements = _getReplacements(colors, attributes);
```

## Performance Considerations

### Gas Optimization
- **String Operations**: Efficient string concatenation using `abi.encodePacked`
- **Memory Management**: Careful memory allocation for large metadata
- **Template Caching**: SVG templates resolved once per call

### Storage Efficiency
- **Attribute Integration**: Leverages existing attribute storage
- **Template Reuse**: Shared SVG templates across tokens
- **Lazy Loading**: Images generated on-demand

## Security Considerations

### Input Validation
- **Key Validation**: Ensures trait keys are not empty
- **Address Validation**: Validates SVG template addresses
- **Data Sanitization**: Proper JSON escaping for metadata values

### Access Control
- **Internal Functions**: Most functions are internal for controlled access
- **Storage Protection**: Diamond storage pattern prevents conflicts
- **Template Security**: SVG template resolution with error handling

## Best Practices

1. **Metadata Standards**: Follow ERC721 metadata standards
2. **Gas Efficiency**: Minimize string operations in loops
3. **Template Organization**: Organize SVG templates logically
4. **Attribute Consistency**: Maintain consistent attribute naming
5. **Error Handling**: Implement proper fallbacks for template failures

## Related Documentation

- [Attribute Library](attribute-lib.md) - Token attribute management
- [SVG Templates Library](svg-templates-lib.md) - SVG template processing
- [Strings Library](strings-lib.md) - String manipulation utilities
- [Diamond Library](diamond-lib.md) - Diamond pattern integration
- [ERC721 Standard](https://eips.ethereum.org/EIPS/eip-721) - NFT metadata standard

## Migration Notes

When upgrading metadata systems:
- Ensure attribute compatibility with existing tokens
- Test SVG template resolution thoroughly
- Validate JSON metadata format compliance
- Consider gas costs for large collections