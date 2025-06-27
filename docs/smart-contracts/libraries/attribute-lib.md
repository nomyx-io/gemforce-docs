# AttributeLib Library

## Overview

The [`AttributeLib`](../../smart-contracts/libraries/attribute-lib.md) library provides core utilities for managing token attributes within the Gemforce platform. This library implements a flexible attribute system that allows tokens to have dynamic, typed metadata stored on-chain, supporting various data types and efficient key-value storage.

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
Attribute memory rarityAttr = Attribute({
    key: "rarity",
    attributeType: AttributeType.String,
    value: "Legendary"
});

AttributeLib._getAttribute(
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
            value: Strings.toString(tokenLevels[tokenId])
        });
        
        attributes[initialAttributes.length + 1] = Attribute({
            key: "experience",
            attributeType: AttributeType.Uint256,
            value: Strings.toString(tokenExperience[tokenId])
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
                for (uint256 j = 0; j < evolutions[i].newAttributes.length; j++) {
                    Attribute memory newAttr = Attribute({
                        key: evolutions[i].newAttributes[j],
                        attributeType: AttributeType.String, // Assuming string for simplicity
                        value: evolutions[i].attributeValues[j]
                    });
                    AttributeLib._setAttribute(attrStorage.attributes, tokenId, newAttr);
                    emit AttributeEvolved(tokenId, newAttr.key, newAttr.value);
                }
                break;
            }
        }
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        // Placeholder for actual NFT existence check
        return true; 
    }
}
```

### Gaming Item System
```solidity
// Gaming platform with customizable items
contract GameItems {
    using AttributeLib for AttributeLib.AttributeStorage;
    
    struct ItemStats {
        uint256 attack;
        uint256 defense;
        uint256 manaCost;
        string itemType;
    }
    
    mapping(uint256 => ItemStats) public baseItemStats;
    
    event ItemCreated(uint256 indexed itemId, string name, string itemType);
    event StatsUpdated(uint256 indexed itemId, uint256 attack, uint256 defense);
    
    function createItem(uint256 itemId, string memory name, string memory itemType, uint256 attack, uint256 defense, uint256 manaCost) external {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        // Set basic attributes
        Attribute[] memory attributes = new Attribute[](5);
        attributes[0] = Attribute({key: "name", attributeType: AttributeType.String, value: name});
        attributes[1] = Attribute({key: "itemType", attributeType: AttributeType.String, value: itemType});
        attributes[2] = Attribute({key: "attack", attributeType: AttributeType.Uint256, value: Strings.toString(attack)});
        attributes[3] = Attribute({key: "defense", attributeType: AttributeType.Uint256, value: Strings.toString(defense)});
        attributes[4] = Attribute({key: "manaCost", attributeType: AttributeType.Uint256, value: Strings.toString(manaCost)});
        
        AttributeLib._setAttributes(attrStorage.attributes, itemId, attributes);
        
        baseItemStats[itemId] = ItemStats(attack, defense, manaCost, itemType);
        
        emit ItemCreated(itemId, name, itemType);
    }
    
    function updateItemStats(uint256 itemId, uint256 newAttack, uint256 newDefense) external {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        Attribute memory attackAttr = Attribute({key: "attack", attributeType: AttributeType.Uint256, value: Strings.toString(newAttack)});
        Attribute memory defenseAttr = Attribute({key: "defense", attributeType: AttributeType.Uint256, value: Strings.toString(newDefense)});
        
        AttributeLib._setAttribute(attrStorage.attributes, itemId, attackAttr);
        AttributeLib._setAttribute(attrStorage.attributes, itemId, defenseAttr);
        
        emit StatsUpdated(itemId, newAttack, newDefense);
    }
    
    function getItemAttack(uint256 itemId) external view returns (uint256) {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        Attribute memory attr = AttributeLib._getAttribute(attrStorage.attributes, itemId, "attack");
        return Strings.toUint(attr.value);
    }
}
```

### Certificate System with Verifiable Attributes
```solidity
// Certificate system for verifiable credentials
contract CertificateSystem {
    using AttributeLib for AttributeLib.AttributeStorage;
    
    struct Certificate {
        uint256 certificateId;
        address holder;
        uint256 issueDate;
        uint256 expiryDate;
        bool revoked;
        string certificateType;
    }
    
    mapping(uint256 => Certificate) public certificates;
    
    event CertificateIssued(uint256 indexed certificateId, address indexed holder, string certificateType);
    event AttributeVerified(uint256 indexed certificateId, string indexed key, string value);
    
    function issueCertificate(uint256 certificateId, address holder, string memory certificateType, uint256 expiryDate, Attribute[] memory attributes) external onlyOwner {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        
        certificates[certificateId] = Certificate({
            certificateId: certificateId,
            holder: holder,
            issueDate: block.timestamp,
            expiryDate: expiryDate,
            revoked: false,
            certificateType: certificateType
        });
        
        AttributeLib._setAttributes(attrStorage.attributes, certificateId, attributes);
        
        emit CertificateIssued(certificateId, holder, certificateType);
    }
    
    function updateCertificateAttributes(uint256 certificateId, Attribute[] memory attributes) external onlyOwner {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        AttributeLib._setAttributes(attrStorage.attributes, certificateId, attributes);
    }
    
    function verifyCertificateAttribute(uint256 certificateId, string memory key, string memory expectedValue) external view returns (bool) {
        AttributeLib.AttributeStorage storage attrStorage = AttributeLib.attributeStorage();
        Attribute memory attr = AttributeLib._getAttribute(attrStorage.attributes, certificateId, key);
        
        bool verified = keccak256(abi.encodePacked(attr.value)) == keccak256(abi.encodePacked(expectedValue));
        emit AttributeVerified(certificateId, key, attr.value);
        return verified;
    }
}
```

## Events

### Attribute Events
- `AttributeSet(uint256 indexed tokenId, string indexed key, AttributeType attributeType, string value)`: Emitted when an attribute is set or updated.
- `AttributeRemoved(uint256 indexed tokenId, string indexed key)`: Emitted when an attribute is removed.
- `AttributesSet(uint256 indexed tokenId, uint256 count)`: Emitted when multiple attributes are set for a token.

## Security Considerations

### Burn Protection
- Prevents attribute operations on burned tokens, ensuring data integrity.

### Access Control
- Attributes can be modified only by authorized entities (e.g., contract owner, designated roles).

### Data Integrity
- Type checking for attributes helps ensure data consistency.
- String conversion for storage prevents unexpected behavior.

## Gas Optimization

### Storage Efficiency
- The `AttributeContract` struct and its internal mappings are designed for efficient storage.
- Storage is minimized by packing related data within structs.

### Function Efficiency
- `_setAttributes` allows batch updates, reducing transaction costs for multiple attribute changes.
- Direct storage access in library functions minimizes overhead.

## Error Handling

### Common Errors
- `"Token has been burned"`: Attempting to modify attributes of a burned token.
- `"Attribute not found"`: Attempting to retrieve or remove a non-existent attribute.
- `"Arrays length mismatch"`: Provided arrays for attributes or values have different lengths.

## Best Practices

### Attribute Design
- Define clear attribute schemas and types for your tokens.
- Use meaningful keys for attributes to improve readability and maintainability.
- Consider off-chain storage for large or highly dynamic metadata to save gas.

### Development Guidelines
- Always implement access control for functions that modify attributes.
- Use comprehensive unit and integration tests to ensure attribute logic is correct.
- Monitor attribute-related events for auditing and analytics.

## Related Documentation

- [Metadata Library](../../smart-contracts/libraries/metadata-lib.md)
- [ERC721A Interface](../../smart-contracts/interfaces/ierc721a.md)
- [ERC721A Library](../../smart-contracts/libraries/erc721a-lib.md)
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)
- [Developer Guides: Development Environment Setup](../../developer-guides/development-environment-setup.md)