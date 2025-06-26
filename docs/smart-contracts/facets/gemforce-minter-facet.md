# GemforceMinterFacet

## Overview

The [`GemforceMinterFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/GemforceMinterFacet.sol) provides specialized token minting functionality within the Gemforce diamond system. This facet enables the creation of NFTs with customizable attributes in a single transaction, supporting rich metadata and flexible token properties for various use cases including environmental assets, trade deals, and marketplace items.

## Contract Details

- **Contract Name**: `GemforceMinterFacet`
- **Inheritance**: `Modifiers`, `IGemforceMinterFacet`
- **License**: MIT
- **Solidity Version**: ^0.8.6

## Key Features

### 🔹 Attribute-Rich Token Minting
- Mint tokens with custom attributes in single transaction
- Flexible metadata system using trait types and values
- Support for complex token properties and characteristics
- Efficient batch attribute assignment

### 🔹 Diamond Integration
- Seamless integration with diamond storage pattern
- Access to identity system for token management
- Consistent token ID generation and tracking
- Event-driven architecture for transparency

### 🔹 Owner-Controlled Minting
- Secure owner-only minting functionality
- Controlled token supply management
- Administrative oversight of token creation
- Integration with access control systems

### 🔹 Metadata Management
- Rich attribute system for token metadata
- Support for various data types and formats
- Extensible attribute framework
- On-chain metadata storage

## Core Data Structures

### Attribute Structure
```solidity
struct Attribute {
    string traitType;    // The type/category of the attribute
    string value;        // The value of the attribute
}
```

### Storage Integration
- **Identity System Storage**: Token state and ownership management
- **Attribute Storage**: Token metadata and attribute management
- **Diamond Storage**: Persistent storage across contract upgrades

## Core Functions

### Token Minting Functions

#### `gemforceMint()`
```solidity
function gemforceMint(Attribute[] memory metadata) external override onlyOwner returns (uint256)
```

**Purpose**: Mints a new token with customizable attributes in a single transaction.

**Parameters**:
- `metadata` (Attribute[]): Array of attribute structs to assign to the token

**Returns**:
- `tokenId` (uint256): ID of the newly minted token

**Access Control**: Owner only

**Process**:
1. Gets next available token ID from identity system
2. Emits GemforceMinted event before state changes
3. Mints token to the caller (owner)
4. Assigns all specified attributes to the token
5. Returns the new token ID

**Events**: `GemforceMinted(tokenId, recipient, metadata)`

**Example Usage**:
```solidity
// Mint environmental asset NFT with attributes
Attribute[] memory attributes = new Attribute[](5);
attributes[0] = Attribute("Asset Type", "Carbon Credit");
attributes[1] = Attribute("Project", "Amazon Rainforest Conservation");
attributes[2] = Attribute("Credits", "1000");
attributes[3] = Attribute("Vintage", "2024");
attributes[4] = Attribute("Standard", "VCS");

uint256 tokenId = IGemforceMinterFacet(diamond).gemforceMint(attributes);
console.log("Minted environmental asset with ID:", tokenId);
```

**Gas Optimization**: Single transaction for minting and attribute assignment reduces gas costs compared to separate operations.

**Security Features**:
- Owner-only access prevents unauthorized minting
- Event emission before state changes ensures transparency
- Atomic operation ensures all-or-nothing minting

## Integration Examples

### Environmental Asset Minting
```solidity
// Mint carbon credit NFTs with environmental attributes
contract EnvironmentalAssetMinter {
    function mintCarbonCreditAsset(
        string memory projectName,
        uint256 creditAmount,
        string memory vintage,
        string memory standard,
        string memory location
    ) external onlyAuthorized returns (uint256) {
        Attribute[] memory attributes = new Attribute[](6);
        attributes[0] = Attribute("Asset Type", "Carbon Credit");
        attributes[1] = Attribute("Project", projectName);
        attributes[2] = Attribute("Credits", creditAmount.toString());
        attributes[3] = Attribute("Vintage", vintage);
        attributes[4] = Attribute("Standard", standard);
        attributes[5] = Attribute("Location", location);
        
        uint256 tokenId = IGemforceMinterFacet(diamond).gemforceMint(attributes);
        
        // Initialize carbon credits for the token
        ICarbonCredit(diamond).initializeCarbonCredit(tokenId, creditAmount);
        
        return tokenId;
    }
}
```

### Trade Deal Collateral Minting
```solidity
// Mint collateral tokens for trade deals
contract TradeDealCollateralMinter {
    function mintCollateralToken(
        uint256 tradeDealId,
        string memory collateralType,
        uint256 value,
        string memory description
    ) external onlyTradeDealManager returns (uint256) {
        Attribute[] memory attributes = new Attribute[](5);
        attributes[0] = Attribute("Asset Type", "Trade Deal Collateral");
        attributes[1] = Attribute("Trade Deal ID", tradeDealId.toString());
        attributes[2] = Attribute("Collateral Type", collateralType);
        attributes[3] = Attribute("Value", value.toString());
        attributes[4] = Attribute("Description", description);
        
        uint256 tokenId = IGemforceMinterFacet(diamond).gemforceMint(attributes);
        
        // Link token to trade deal
        ITradeDeal(diamond).addCollateralToken(tradeDealId, tokenId);
        
        return tokenId;
    }
}
```

### Marketplace Item Minting
```solidity
// Mint marketplace items with rich metadata
contract MarketplaceItemMinter {
    function mintMarketplaceItem(
        string memory itemName,
        string memory category,
        string memory description,
        string memory imageURI,
        uint256 price
    ) external onlyMerchant returns (uint256) {
        Attribute[] memory attributes = new Attribute[](6);
        attributes[0] = Attribute("Item Type", "Marketplace Item");
        attributes[1] = Attribute("Name", itemName);
        attributes[2] = Attribute("Category", category);
        attributes[3] = Attribute("Description", description);
        attributes[4] = Attribute("Image", imageURI);
        attributes[5] = Attribute("Price", price.toString());
        
        uint256 tokenId = IGemforceMinterFacet(diamond).gemforceMint(attributes);
        
        // List item on marketplace
        IMarketplace(diamond).listItem(
            address(0),
            payable(msg.sender),
            tokenId,
            price,
            false,
            address(0)
        );
        
        return tokenId;
    }
}
```

### Batch Minting System
```solidity
// Efficient batch minting for multiple tokens
contract BatchMintingSystem {
    function batchMintTokens(
        Attribute[][] memory tokenAttributes
    ) external onlyOwner returns (uint256[] memory) {
        uint256[] memory tokenIds = new uint256[](tokenAttributes.length);
        
        for (uint256 i = 0; i < tokenAttributes.length; i++) {
            tokenIds[i] = IGemforceMinterFacet(diamond).gemforceMint(
                tokenAttributes[i]
            );
        }
        
        return tokenIds;
    }
    
    function mintTokenCollection(
        string memory collectionName,
        string memory baseImageURI,
        uint256 quantity
    ) external onlyOwner returns (uint256[] memory) {
        uint256[] memory tokenIds = new uint256[](quantity);
        
        for (uint256 i = 0; i < quantity; i++) {
            Attribute[] memory attributes = new Attribute[](4);
            attributes[0] = Attribute("Collection", collectionName);
            attributes[1] = Attribute("Edition", (i + 1).toString());
            attributes[2] = Attribute("Total Supply", quantity.toString());
            attributes[3] = Attribute("Image", string(abi.encodePacked(baseImageURI, (i + 1).toString())));
            
            tokenIds[i] = IGemforceMinterFacet(diamond).gemforceMint(attributes);
        }
        
        return tokenIds;
    }
}
```

### Dynamic Attribute Minting
```solidity
// Mint tokens with dynamic attributes based on conditions
contract DynamicAttributeMinter {
    function mintDynamicToken(
        string memory tokenType,
        uint256 rarity,
        bool isSpecial
    ) external onlyOwner returns (uint256) {
        // Base attributes
        Attribute[] memory attributes = new Attribute[](isSpecial ? 6 : 4);
        attributes[0] = Attribute("Type", tokenType);
        attributes[1] = Attribute("Rarity", getRarityString(rarity));
        attributes[2] = Attribute("Minted At", block.timestamp.toString());
        attributes[3] = Attribute("Minted By", "Gemforce System");
        
        // Add special attributes if applicable
        if (isSpecial) {
            attributes[4] = Attribute("Special", "true");
            attributes[5] = Attribute("Special Type", getSpecialType(rarity));
        }
        
        uint256 tokenId = IGemforceMinterFacet(diamond).gemforceMint(attributes);
        
        // Apply special properties if needed
        if (isSpecial) {
            applySpecialProperties(tokenId, rarity);
        }
        
        return tokenId;
    }
    
    function getRarityString(uint256 rarity) internal pure returns (string memory) {
        if (rarity >= 90) return "Legendary";
        if (rarity >= 70) return "Epic";
        if (rarity >= 50) return "Rare";
        if (rarity >= 30) return "Uncommon";
        return "Common";
    }
    
    function getSpecialType(uint256 rarity) internal pure returns (string memory) {
        if (rarity >= 95) return "Mythic";
        if (rarity >= 90) return "Legendary";
        return "Special Edition";
    }
}
```

### Metadata Validation System
```solidity
// Validate and standardize metadata before minting
contract MetadataValidationMinter {
    mapping(string => bool) public validTraitTypes;
    mapping(string => string[]) public validTraitValues;
    
    function setValidTraitTypes(string[] memory traitTypes) external onlyOwner {
        for (uint256 i = 0; i < traitTypes.length; i++) {
            validTraitTypes[traitTypes[i]] = true;
        }
    }
    
    function setValidTraitValues(
        string memory traitType,
        string[] memory values
    ) external onlyOwner {
        validTraitValues[traitType] = values;
    }
    
    function mintValidatedToken(
        Attribute[] memory attributes
    ) external onlyOwner returns (uint256) {
        // Validate all attributes
        for (uint256 i = 0; i < attributes.length; i++) {
            require(
                validTraitTypes[attributes[i].traitType],
                "Invalid trait type"
            );
            
            // Validate trait values if specified
            string[] memory validValues = validTraitValues[attributes[i].traitType];
            if (validValues.length > 0) {
                bool isValid = false;
                for (uint256 j = 0; j < validValues.length; j++) {
                    if (keccak256(bytes(validValues[j])) == keccak256(bytes(attributes[i].value))) {
                        isValid = true;
                        break;
                    }
                }
                require(isValid, "Invalid trait value");
            }
        }
        
        return IGemforceMinterFacet(diamond).gemforceMint(attributes);
    }
}
```

## Events

### Minting Events
```solidity
event GemforceMinted(
    uint256 indexed tokenId,
    address indexed recipient,
    Attribute[] metadata
);
```

**Event Details**:
- `tokenId`: ID of the newly minted token
- `recipient`: Address that received the token (always the owner)
- `metadata`: Array of attributes assigned to the token

## Security Considerations

### Access Control
- Only contract owner can mint tokens
- Prevents unauthorized token creation
- Integration with diamond access control system

### Data Integrity
- Atomic minting and attribute assignment
- Event emission before state changes
- Consistent token ID generation

### Attribute Validation
- Flexible attribute system supports various data types
- On-chain storage ensures data persistence
- Integration with attribute library for consistency

## Gas Optimization

### Single Transaction Minting
- Combines token minting and attribute assignment
- Reduces gas costs compared to separate operations
- Efficient batch attribute processing

### Storage Efficiency
- Uses diamond storage pattern for optimal storage
- Minimal storage footprint per token
- Efficient attribute storage through library

## Error Handling

### Access Control Errors
- Owner-only function restrictions
- Unauthorized access prevention

### Minting Errors
- Token creation failures
- Attribute assignment failures
- Storage operation errors

## Testing Considerations

### Unit Tests
- Token minting functionality
- Attribute assignment accuracy
- Access control validation
- Event emission verification

### Integration Tests
- Diamond storage integration
- Attribute library integration
- Multi-facet workflows
- Gas optimization validation

## Related Documentation

- [IGemforceMinterFacet](../interfaces/igemforce-minter-facet.md) - Minter interface
- [AttributeLib](../libraries/attribute-lib.md) - Attribute management library
- [IdentitySystemStorage](../libraries/identity-system-storage.md) - Storage library
- [CarbonCreditFacet](./carbon-credit-facet.md) - Environmental asset integration
- [MarketplaceFacet](./marketplace-facet.md) - Marketplace integration
- [Token Minting Guide](../../guides/token-minting.md) - Implementation guide

---

*This facet provides the foundation for creating rich, attribute-enabled NFTs within the Gemforce platform, supporting diverse use cases from environmental assets to marketplace items with comprehensive metadata management.*