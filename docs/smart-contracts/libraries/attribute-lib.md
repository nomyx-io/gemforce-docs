# AttributeLib Library

## Overview

The [`AttributeLib`](../../../contracts/libraries/AttributeLib.sol) library provides core utilities for managing token attributes within the Gemforce platform. This library implements a flexible attribute system that allows tokens to have dynamic, typed metadata stored on-chain, supporting various data types and efficient key-value storage.

## Key Features

- **Typed Attributes**: Support for multiple data types (String, Bytes32, Uint256, Arrays)
- **Dynamic Metadata**: Runtime attribute assignment and modification
- **Efficient Storage**: Optimized key-value storage with indexing
- **Burn Protection**: Prevents attribute operations on burned tokens
- **Batch Operations**: Support for setting multiple attributes simultaneously
- **Event Tracking**: Comprehensive event emission for attribute changes

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.6;

library AttributeLib {
    // Storage management
    function attributeStorage() internal pure returns (AttributeStorage storage);
    
    // Attribute retrieval
    function _getAttribute(AttributeContract storage self, uint256 tokenId, string memory key) internal view returns (Attribute memory);
    function _getAttributeValues(uint256 id) internal view returns (string[] memory);
    function _getAttributeKeys(AttributeContract storage self, uint256 tokenId) internal view returns (string[] memory);
    
    // Attribute modification
    function _setAttribute(AttributeContract storage self, uint256 tokenId, Attribute memory attribute) internal;
    function _setAttributes(AttributeContract storage self, uint256 tokenId, Attribute[] memory _attributes) internal;
    function _removeAttribute(AttributeContract storage self, uint256 tokenId, string memory key) internal;
    
    // Token lifecycle
    function _burn(AttributeContract storage self, uint256 tokenId) internal;
}
```

## Data Structures

### AttributeType Enum
```solidity
enum AttributeType {
    Unknown,        // Undefined type
    String,         // String value
    Bytes32,        // 32-byte value
    Uint256,        // 256-bit unsigned integer
    Uint8,          // 8-bit unsigned integer
    Uint256Array,   // Array of 256-bit unsigned integers
    Uint8Array      // Array of 8-bit unsigned integers
}
```

**Purpose**: Defines supported attribute data types for type-safe operations.

### Attribute Struct
```solidity
struct Attribute {
    string key;                     // Attribute identifier
    AttributeType attributeType;    // Type of the attribute value
    string value;                   // String representation of the value
}
```

**Purpose**: Core attribute data structure containing key, type, and value.

**Design Notes**:
- All values stored as strings for simplicity and gas efficiency
- Type information preserved for proper interpretation
- Key serves as unique identifier within token scope

### AttributeContract Struct
```solidity
struct AttributeContract {
    mapping(uint256 => bool) burnedIds;                                     // Burned token tracking
    mapping(uint256 => mapping(string => Attribute)) attributes;            // Token attributes
    mapping(uint256 => string[]) attributeKeys;                            // Token attribute keys
    mapping(uint256 => mapping(string => uint256)) attributeKeysIndexes;   // Key index mapping
}
```

**Purpose**: Complete storage structure for token attribute management.

**Components**:
- **burnedIds**: Tracks burned tokens to prevent attribute operations
- **attributes**: Nested mapping for token-specific attribute storage
- **attributeKeys**: Ordered list of attribute keys per token
- **attributeKeysIndexes**: Efficient key lookup and removal indexing

### AttributeStorage Struct
```solidity
struct AttributeStorage {
    AttributeContract attributes;
}
```

**Purpose**: Diamond storage wrapper for attribute data.

**Storage Position**:
```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nextblock.bitgem.app.AttributeStorage.storage");
```

## Core Functions

### Storage Management

#### `attributeStorage()`
```solidity
function attributeStorage() internal pure returns (AttributeStorage storage ds)
```

**Purpose**: Access Diamond storage for attribute data using assembly.

**Implementation**:
```solidity
function attributeStorage() internal pure returns (AttributeStorage storage ds) {
    bytes32 position = DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
}
```

**Returns**: Storage reference to attribute data

**Usage**: All attribute functions use this to access persistent storage.

### Attribute Retrieval

#### `_getAttribute()`
```solidity
function _getAttribute(
    AttributeContract storage self,
    uint256 tokenId,
    string memory key
) internal view returns (Attribute memory)
```

**Purpose**: Retrieve a specific attribute for a token by key.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier
- `key`: Attribute key to retrieve

**Returns**: Attribute struct containing key, type, and value

**Security**: Prevents access to burned token attributes

**Example Usage**:
```solidity
// Get token rarity attribute
Attribute memory rarityAttr = AttributeLib._getAttribute(
    attributeStorage.attributes,
    tokenId,
    "rarity"
);
console.log("Token rarity:", rarityAttr.value);
```

#### `_getAttributeValues()`
```solidity
function _getAttributeValues(uint256 id) internal view returns (string[] memory)
```

**Purpose**: Retrieve all attribute values for a token in key order.

**Parameters**:
- `id`: Token identifier

**Returns**: Array of attribute values as strings

**Implementation**:
```solidity
function _getAttributeValues(uint256 id) internal view returns (string[] memory) {
    AttributeContract storage ct = AttributeLib.attributeStorage().attributes;
    string[] memory keys = ct.attributeKeys[id];
    string[] memory values = new string[](keys.length);
    uint256 keysLength = keys.length;
    for (uint256 i = 0; i < keysLength; i++) {
        values[i] = ct.attributes[id][keys[i]].value;
    }
    return values;
}
```

**Use Cases**:
- Metadata generation for NFTs
- Bulk attribute export
- Attribute comparison operations

#### `_getAttributeKeys()`
```solidity
function _getAttributeKeys(
    AttributeContract storage self,
    uint256 tokenId
) internal view returns (string[] memory)
```

**Purpose**: Retrieve all attribute keys for a token.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier

**Returns**: Array of attribute keys

**Security**: Prevents access to burned token attributes

**Example Usage**:
```solidity
// Get all attribute keys for a token
string[] memory keys = AttributeLib._getAttributeKeys(
    attributeStorage.attributes,
    tokenId
);

for (uint256 i = 0; i < keys.length; i++) {
    console.log("Attribute key:", keys[i]);
}
```

### Attribute Modification

#### `_setAttribute()`
```solidity
function _setAttribute(
    AttributeContract storage self,
    uint256 tokenId,
    Attribute memory attribute
) internal
```

**Purpose**: Set or update a single attribute for a token.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier
- `attribute`: Attribute data to set

**Process**:
1. Verify token is not burned
2. Check if attribute key is new
3. Add key to keys array if new
4. Update key index mapping
5. Store attribute data

**Security**: Prevents modification of burned token attributes

**Example Usage**:
```solidity
// Set token rarity attribute
Attribute memory rarityAttr = Attribute({
    key: "rarity",
    attributeType: AttributeType.String,
    value: "Legendary"
});

AttributeLib._setAttribute(
    attributeStorage.attributes,
    tokenId,
    rarityAttr
);
```

#### `_setAttributes()`
```solidity
function _setAttributes(
    AttributeContract storage self,
    uint256 tokenId,
    Attribute[] memory _attributes
) internal
```

**Purpose**: Set multiple attributes for a token in a single operation.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier
- `_attributes`: Array of attributes to set

**Implementation**:
```solidity
function _setAttributes(
    AttributeContract storage self,
    uint256 tokenId, 
    Attribute[] memory _attributes
) internal {
    require(self.burnedIds[tokenId] == false, "Token has been burned");
    uint256 attributesLength = _attributes.length;
    for (uint256 i = 0; i < attributesLength; i++) {
        _setAttribute(self, tokenId, _attributes[i]);
    }
}
```

**Benefits**:
- Gas efficient for multiple attributes
- Atomic operation for consistency
- Simplified batch updates

**Example Usage**:
```solidity
// Set multiple attributes at once
Attribute[] memory attributes = new Attribute[](3);
attributes[0] = Attribute("rarity", AttributeType.String, "Epic");
attributes[1] = Attribute("level", AttributeType.Uint256, "25");
attributes[2] = Attribute("element", AttributeType.String, "Fire");

AttributeLib._setAttributes(
    attributeStorage.attributes,
    tokenId,
    attributes
);
```

#### `_removeAttribute()`
```solidity
function _removeAttribute(
    AttributeContract storage self,
    uint256 tokenId,
    string memory key
) internal
```

**Purpose**: Remove a specific attribute from a token.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier
- `key`: Attribute key to remove

**Process**:
1. Verify token is not burned
2. Delete attribute data
3. Find key index in keys array
4. Shift remaining keys to fill gap
5. Update key indexes
6. Remove last element from array
7. Emit removal event

**Security**: Prevents modification of burned token attributes

**Example Usage**:
```solidity
// Remove temporary attribute
AttributeLib._removeAttribute(
    attributeStorage.attributes,
    tokenId,
    "temporary_boost"
);
```

### Token Lifecycle

#### `_burn()`
```solidity
function _burn(
    AttributeContract storage self,
    uint256 tokenId
) internal
```

**Purpose**: Mark a token as burned to prevent future attribute operations.

**Parameters**:
- `self`: Storage reference to attribute contract
- `tokenId`: Token identifier to burn

**Implementation**:
```solidity
function _burn(
    AttributeContract storage self,
    uint256 tokenId
) internal {
    self.burnedIds[tokenId] = true;
}
```

**Effects**:
- Prevents all future attribute operations on the token
- Preserves existing attribute data for historical purposes
- Enables gas-efficient burn protection checks

## Integration Examples

### NFT Collection with Dynamic Attributes
```solidity
// NFT collection with evolving attributes
contract EvolvingNFT {
    using AttributeLib for AttributeLib.AttributeStorage;
    
    struct Evolution {
        uint256 requiredLevel;
        string[] newAttributes;
        string[] attributeValues;
    }
    
    mapping(uint256 => uint256) public tokenLevels;
    mapping(uint256 => uint256) public tokenExperience;
    mapping(uint256 => Evolution[]) public evolutionPaths;
    
    event TokenEvolved(uint256 indexed tokenId, uint256 newLevel);
    event ExperienceGained(uint256 indexed tokenId, uint256 experience);
    event AttributeEvolved(uint256 indexed tokenId, string attribute, string newValue);
    
    function initializeToken(uint256 tokenId, string[] memory initialAttributes, string[] memory initialValues) external {
        require(_exists(tokenId), "Token does not exist");
        require(initialAttributes.length == initialValues.length, "Arrays length mismatch");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Set initial attributes
        Attribute[] memory attributes = new Attribute[](initialAttributes.length + 2);
        
        for (uint256 i = 0; i < initialAttributes.length; i++) {
            attributes[i] = Attribute({
                key: initialAttributes[i],
                attributeType: AttributeType.String,
                value: initialValues[i]
            });
        }
        
        // Add level and experience
        attributes[initialAttributes.length] = Attribute({
            key: "level",
            attributeType: AttributeType.Uint256,
            value: "1"
        });
        
        attributes[initialAttributes.length + 1] = Attribute({
            key: "experience",
            attributeType: AttributeType.Uint256,
            value: "0"
        });
        
        AttributeLib._setAttributes(attrStorage.attributes, tokenId, attributes);
        
        tokenLevels[tokenId] = 1;
        tokenExperience[tokenId] = 0;
    }
    
    function gainExperience(uint256 tokenId, uint256 experience) external {
        require(_exists(tokenId), "Token does not exist");
        require(experience > 0, "Experience must be positive");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Update experience
        tokenExperience[tokenId] += experience;
        
        // Update experience attribute
        Attribute memory expAttr = Attribute({
            key: "experience",
            attributeType: AttributeType.Uint256,
            value: Strings.toString(tokenExperience[tokenId])
        });
        
        AttributeLib._setAttribute(attrStorage.attributes, tokenId, expAttr);
        
        emit ExperienceGained(tokenId, experience);
        
        // Check for level up
        _checkLevelUp(tokenId);
    }
    
    function _checkLevelUp(uint256 tokenId) internal {
        uint256 currentLevel = tokenLevels[tokenId];
        uint256 currentExp = tokenExperience[tokenId];
        uint256 requiredExp = currentLevel * 100; // Simple formula
        
        if (currentExp >= requiredExp) {
            uint256 newLevel = currentLevel + 1;
            tokenLevels[tokenId] = newLevel;
            
            AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
            
            // Update level attribute
            Attribute memory levelAttr = Attribute({
                key: "level",
                attributeType: AttributeType.Uint256,
                value: Strings.toString(newLevel)
            });
            
            AttributeLib._setAttribute(attrStorage.attributes, tokenId, levelAttr);
            
            emit TokenEvolved(tokenId, newLevel);
            
            // Apply evolution changes
            _applyEvolution(tokenId, newLevel);
        }
    }
    
    function _applyEvolution(uint256 tokenId, uint256 newLevel) internal {
        Evolution[] memory evolutions = evolutionPaths[tokenId];
        
        for (uint256 i = 0; i < evolutions.length; i++) {
            if (evolutions[i].requiredLevel == newLevel) {
                AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
                
                // Apply evolution attributes
                for (uint256 j = 0; j < evolutions[i].newAttributes.length; j++) {
                    Attribute memory attr = Attribute({
                        key: evolutions[i].newAttributes[j],
                        attributeType: AttributeType.String,
                        value: evolutions[i].attributeValues[j]
                    });
                    
                    AttributeLib._setAttribute(attrStorage.attributes, tokenId, attr);
                    
                    emit AttributeEvolved(tokenId, evolutions[i].newAttributes[j], evolutions[i].attributeValues[j]);
                }
                break;
            }
        }
    }
    
    function getTokenAttributes(uint256 tokenId) external view returns (string[] memory keys, string[] memory values) {
        require(_exists(tokenId), "Token does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        keys = AttributeLib._getAttributeKeys(attrStorage.attributes, tokenId);
        values = AttributeLib._getAttributeValues(tokenId);
    }
    
    function getAttribute(uint256 tokenId, string memory key) external view returns (Attribute memory) {
        require(_exists(tokenId), "Token does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        return AttributeLib._getAttribute(attrStorage.attributes, tokenId, key);
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        // Implementation would check token existence
        return true;
    }
}
```

### Gaming Item System
```solidity
// Gaming items with dynamic stats and upgrades
contract GameItems {
    using AttributeLib for AttributeLib.AttributeStorage;
    
    struct ItemType {
        string name;
        string category;
        string[] baseAttributes;
        string[] baseValues;
        uint256 maxLevel;
    }
    
    struct UpgradePath {
        uint256 level;
        string attribute;
        string valueModifier;
        uint256 cost;
    }
    
    mapping(uint256 => ItemType) public itemTypes;
    mapping(uint256 => UpgradePath[]) public upgradePaths;
    mapping(uint256 => uint256) public itemTypeIds;
    IERC20 public upgradeToken;
    
    event ItemCreated(uint256 indexed tokenId, uint256 itemTypeId);
    event ItemUpgraded(uint256 indexed tokenId, string attribute, string newValue);
    event ItemEnchanted(uint256 indexed tokenId, string enchantment);
    
    constructor(address _upgradeToken) {
        upgradeToken = IERC20(_upgradeToken);
    }
    
    function createItemType(
        uint256 itemTypeId,
        string memory name,
        string memory category,
        string[] memory baseAttributes,
        string[] memory baseValues,
        uint256 maxLevel
    ) external onlyGameMaster {
        require(baseAttributes.length == baseValues.length, "Arrays length mismatch");
        
        itemTypes[itemTypeId] = ItemType({
            name: name,
            category: category,
            baseAttributes: baseAttributes,
            baseValues: baseValues,
            maxLevel: maxLevel
        });
    }
    
    function mintItem(address to, uint256 tokenId, uint256 itemTypeId) external onlyGameMaster {
        ItemType memory itemType = itemTypes[itemTypeId];
        require(bytes(itemType.name).length > 0, "Item type does not exist");
        
        // Mint token (implementation would call ERC721 mint)
        _mint(to, tokenId);
        
        itemTypeIds[tokenId] = itemTypeId;
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Set base attributes
        Attribute[] memory attributes = new Attribute[](itemType.baseAttributes.length + 3);
        
        for (uint256 i = 0; i < itemType.baseAttributes.length; i++) {
            attributes[i] = Attribute({
                key: itemType.baseAttributes[i],
                attributeType: AttributeType.String,
                value: itemType.baseValues[i]
            });
        }
        
        // Add metadata attributes
        attributes[itemType.baseAttributes.length] = Attribute({
            key: "item_type",
            attributeType: AttributeType.String,
            value: itemType.name
        });
        
        attributes[itemType.baseAttributes.length + 1] = Attribute({
            key: "category",
            attributeType: AttributeType.String,
            value: itemType.category
        });
        
        attributes[itemType.baseAttributes.length + 2] = Attribute({
            key: "level",
            attributeType: AttributeType.Uint256,
            value: "1"
        });
        
        AttributeLib._setAttributes(attrStorage.attributes, tokenId, attributes);
        
        emit ItemCreated(tokenId, itemTypeId);
    }
    
    function upgradeItem(uint256 tokenId, string memory attribute) external {
        require(_exists(tokenId), "Token does not exist");
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Get current level
        Attribute memory levelAttr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, "level");
        uint256 currentLevel = StringsLib.parseUint(levelAttr.value);
        
        ItemType memory itemType = itemTypes[itemTypeIds[tokenId]];
        require(currentLevel < itemType.maxLevel, "Item at max level");
        
        // Find upgrade path
        UpgradePath[] memory paths = upgradePaths[itemTypeIds[tokenId]];
        UpgradePath memory upgradePath;
        bool found = false;
        
        for (uint256 i = 0; i < paths.length; i++) {
            if (paths[i].level == currentLevel + 1 && 
                keccak256(bytes(paths[i].attribute)) == keccak256(bytes(attribute))) {
                upgradePath = paths[i];
                found = true;
                break;
            }
        }
        
        require(found, "Upgrade path not found");
        
        // Pay upgrade cost
        upgradeToken.transferFrom(msg.sender, address(this), upgradePath.cost);
        
        // Apply upgrade
        Attribute memory currentAttr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, attribute);
        uint256 currentValue = StringsLib.parseUint(currentAttr.value);
        uint256 modifier = StringsLib.parseUint(upgradePath.valueModifier);
        uint256 newValue = currentValue + modifier;
        
        Attribute memory newAttr = Attribute({
            key: attribute,
            attributeType: currentAttr.attributeType,
            value: Strings.toString(newValue)
        });
        
        AttributeLib._setAttribute(attrStorage.attributes, tokenId, newAttr);
        
        // Update level
        Attribute memory newLevelAttr = Attribute({
            key: "level",
            attributeType: AttributeType.Uint256,
            value: Strings.toString(currentLevel + 1)
        });
        
        AttributeLib._setAttribute(attrStorage.attributes, tokenId, newLevelAttr);
        
        emit ItemUpgraded(tokenId, attribute, Strings.toString(newValue));
    }
    
    function enchantItem(uint256 tokenId, string memory enchantment, string memory value) external onlyGameMaster {
        require(_exists(tokenId), "Token does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        Attribute memory enchantAttr = Attribute({
            key: enchantment,
            attributeType: AttributeType.String,
            value: value
        });
        
        AttributeLib._setAttribute(attrStorage.attributes, tokenId, enchantAttr);
        
        emit ItemEnchanted(tokenId, enchantment);
    }
    
    function getItemStats(uint256 tokenId) external view returns (
        string[] memory attributes,
        string[] memory values,
        string memory itemType,
        uint256 level
    ) {
        require(_exists(tokenId), "Token does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        attributes = AttributeLib._getAttributeKeys(attrStorage.attributes, tokenId);
        values = AttributeLib._getAttributeValues(tokenId);
        
        Attribute memory typeAttr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, "item_type");
        Attribute memory levelAttr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, "level");
        
        itemType = typeAttr.value;
        level = StringsLib.parseUint(levelAttr.value);
    }
    
    function burnItem(uint256 tokenId) external {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        AttributeLib._burn(attrStorage.attributes, tokenId);
        
        // Burn token (implementation would call ERC721 burn)
        _burn(tokenId);
    }
    
    function _mint(address to, uint256 tokenId) internal {
        // Implementation would mint ERC721 token
    }
    
    function _burn(uint256 tokenId) internal {
        // Implementation would burn ERC721 token
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        // Implementation would check token existence
        return true;
    }
    
    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        // Implementation would check authorization
        return true;
    }
    
    modifier onlyGameMaster() {
        // Implementation would check game master role
        _;
    }
}
```

### Certificate System with Verifiable Attributes
```solidity
// Certificate system with verifiable attributes
contract VerifiableCertificates {
    using AttributeLib for AttributeLib.AttributeStorage;
    
    struct CertificateTemplate {
        string name;
        string[] requiredAttributes;
        address issuer;
        bool active;
    }
    
    struct Verification {
        address verifier;
        uint256 timestamp;
        bool verified;
        string notes;
    }
    
    mapping(string => CertificateTemplate) public certificateTemplates;
    mapping(uint256 => Verification[]) public verifications;
    mapping(address => bool) public authorizedVerifiers;
    
    event CertificateIssued(uint256 indexed tokenId, string template, address indexed recipient);
    event AttributeVerified(uint256 indexed tokenId, string attribute, address indexed verifier);
    event VerificationAdded(uint256 indexed tokenId, address indexed verifier, bool verified);
    
    function createCertificateTemplate(
        string memory templateName,
        string[] memory requiredAttributes
    ) external {
        certificateTemplates[templateName] = CertificateTemplate({
            name: templateName,
            requiredAttributes: requiredAttributes,
            issuer: msg.sender,
            active: true
        });
    }
    
    function issueCertificate(
        uint256 tokenId,
        address recipient,
        string memory templateName,
        string[] memory attributeValues
    ) external {
        CertificateTemplate memory template = certificateTemplates[templateName];
        require(template.active, "Template not active");
        require(msg.sender == template.issuer, "Not authorized issuer");
        require(template.requiredAttributes.length == attributeValues.length, "Attributes length mismatch");
        
        // Mint certificate token
        _mint(recipient, tokenId);
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Set certificate attributes
        Attribute[] memory attributes = new Attribute[](template.requiredAttributes.length + 3);
        
        for (uint256 i = 0; i < template.requiredAttributes.length; i++) {
            attributes[i] = Attribute({
                key: template.requiredAttributes[i],
                attributeType: AttributeType.String,
                value: attributeValues[i]
            });
        }
        
        // Add metadata
        attributes[template.requiredAttributes.length] = Attribute({
            key: "template",
            attributeType: AttributeType.String,
            value: templateName
        });
        
        attributes[template.requiredAttributes.length + 1] = Attribute({
            key: "issuer",
            attributeType: AttributeType.String,
            value: Strings.toHexString(uint160(template.issuer), 20)
        });
        
        attributes[template.requiredAttributes.length + 2] = Attribute({
            key: "issue_date",
            attributeType: AttributeType.Uint256,
            value: Strings.toString(block.timestamp)
        });
        
        AttributeLib._setAttributes(attrStorage.attributes, tokenId, attributes);
        
        emit CertificateIssued(tokenId, templateName, recipient);
    }
    
    function addVerification(
        uint256 tokenId,
        bool verified,
        string memory notes
    ) external {
        require(authorizedVerifiers[msg.sender], "Not authorized verifier");
        require(_exists(tokenId), "Certificate does not exist");
        
        verifications[tokenId].push(Verification({
            verifier: msg.sender,
            timestamp: block.timestamp,
            verified: verified,
            notes: notes
        }));
        
        emit VerificationAdded(tokenId, msg.sender, verified);
    }
    
    function verifyAttribute(
        uint256 tokenId,
        string memory attribute,
        string memory expectedValue
    ) external view returns (bool) {
        require(_exists(tokenId), "Certificate does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        Attribute memory attr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, attribute);
        
        return keccak256(bytes(attr.value)) == keccak256(bytes(expectedValue));
    }
    
    function getCertificateData(uint256 tokenId) external view returns (
        string[] memory attributes,
        string[] memory values,
        Verification[] memory certificateVerifications
    ) {
        require(_exists(tokenId), "Certificate does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        attributes = AttributeLib._getAttributeKeys(attrStorage.attributes, tokenId);
        values = AttributeLib._getAttributeValues(tokenId);
        certificateVerifications = verifications[tokenId];
    }
    
    function revokeCertificate(uint256 tokenId) external {
        require(_exists(tokenId), "Certificate does not exist");
        
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        Attribute memory issuerAttr = AttributeLib._getAttribute(attrStorage.attributes, tokenId, "issuer");
        
        address issuer = parseAddress(issuerAttr.value);
        require(msg.sender == issuer, "Not authorized");
        
        // Add revocation attribute
        Attribute memory revokedAttr = Attribute({
            key: "revoked",
            attributeType: AttributeType.Uint256,
            value: Strings.toString(block.timestamp)
        });
        
        AttributeLib._setAttribute(attrStorage.attributes, tokenId, revokedAttr);
    }
    
    function parseAddress(string memory addressString) internal pure returns (address) {
        // Implementation would parse hex string to address
        return address(0);
    }
    
    function _mint(address to, uint256 tokenId) internal {
        // Implementation would mint ERC721 token
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        // Implementation would check token existence
        return true;
    }
}
```

## Events

### Attribute Events
```solidity
event AttributeSet(address indexed tokenAddress, uint256 tokenId, Attribute attribute);
event AttributeRemoved(address indexed tokenAddress, uint256 tokenId, string attributeKey);
```

## Security Considerations

### Burn Protection
- All attribute operations check burn status
- Prevents modification of burned token attributes
- Preserves historical data for burned tokens
- Efficient burn status checking

### Access Control
- No built-in access control (delegated to calling contracts)
- Calling contracts must implement proper authorization
- Event emission for audit trails

### Access Control
- No built-in access control (delegated to calling contracts)
- Calling contracts must implement proper authorization
- Event emission for audit trails
- Attribute modification requires proper token ownership validation

### Data Integrity
- Type information preserved for proper value interpretation
- String storage for all values ensures consistency
- Key uniqueness enforced within token scope
- Efficient indexing for key management

## Gas Optimization

### Storage Efficiency
- Optimized key-value storage with indexing
- Minimal storage writes during attribute operations
- Efficient key removal with array compaction
- Batch operations for multiple attributes

### Function Efficiency
- Direct storage access without external calls
- Efficient key lookup using index mapping
- Minimal gas overhead for attribute operations
- Optimized loops for batch operations

## Error Handling

### Common Errors
- "Token has been burned" - Attempt to modify burned token attributes
- Array length mismatches in batch operations
- Invalid token IDs or non-existent tokens
- Unauthorized attribute modifications

### Best Practices
- Always validate token existence before operations
- Check burn status before attribute modifications
- Implement proper access control in calling contracts
- Use batch operations for multiple attributes
- Handle missing attributes gracefully

## Testing Considerations

### Unit Tests
- Attribute setting and retrieval
- Batch attribute operations
- Key management and indexing
- Burn protection functionality
- Event emission verification

### Integration Tests
- Cross-contract attribute usage
- NFT metadata integration
- Gaming system attribute evolution
- Certificate verification workflows
- Attribute-based access control

### Edge Cases
- Empty attribute values
- Maximum key length handling
- Large attribute arrays
- Concurrent attribute modifications
- Burn state transitions

## Related Documentation

- [IAttribute Interface](../interfaces/iattribute.md) - Attribute interface definition
- [AttributeFacet](../facets/attribute-facet.md) - Attribute facet implementation
- [ERC721 Integration](../../guides/erc721-attributes.md) - NFT attribute integration
- [Gaming Attributes Guide](../../guides/gaming-attributes.md) - Gaming system attribute patterns
- [Metadata Standards](../../guides/metadata-standards.md) - Attribute metadata best practices

---

*This library provides comprehensive utilities for managing dynamic token attributes within the Gemforce platform, supporting typed metadata, efficient storage, and flexible attribute systems for NFTs, gaming items, certificates, and other tokenized assets.*