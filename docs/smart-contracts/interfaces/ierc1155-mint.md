# IERC1155Mint Interface

## Overview

The [`IERC1155Mint`](../../smart-contracts/interfaces/ierc1155-mint.md) interface defines the standard contract for minting ERC1155 tokens within the Gemforce platform. This interface extends the standard ERC1155 functionality by providing controlled minting capabilities for multi-token contracts, enabling the creation of fungible, non-fungible, and semi-fungible tokens with flexible quantity management.

## Key Features

- **Flexible Token Minting**: Support for minting single and multiple token types
- **Recipient Control**: Mint tokens to specific addresses or caller
- **Batch Operations**: Efficient batch minting for multiple token types
- **Quantity Management**: Precise control over minted token quantities
- **Event Tracking**: Comprehensive event emission for minting operations

## Interface Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

interface IERC1155Mint {
    // Events
    event ERC1155TokenMinted(
        address minter,
        uint256 id,
        uint256 quantity
    );
    
    // Core Minting Functions
    function mint(uint256 id, uint256 quantity, bytes memory data) external;
    function mintTo(address recipient, uint256 id, uint256 quantity, bytes memory data) external;
    function batchMintTo(address recipient, uint256[] memory ids, uint256[] calldata quantities, bytes memory data) external;
}
```

## Core Functions

### Single Token Minting

#### `mint()`
```solidity
function mint(uint256 id, uint256 quantity, bytes memory data) external
```

**Purpose**: Mint tokens of a specified type and quantity to the caller's address.

**Parameters**:
- `id` (uint256): The token type identifier to mint
- `quantity` (uint256): The amount of tokens to mint
- `data` (bytes): Additional data to pass to the recipient (if it's a contract)

**Requirements**:
- Caller must have minting permissions
- Token ID must be valid for minting
- Quantity must be greater than zero
- Contract must not be paused (if pausable)

**Events Emitted**:
- [`ERC1155TokenMinted`](ierc1155-mint.md:8) with minter address, token ID, and quantity

**Example Usage**:
```solidity
// Mint 100 tokens of type 1 to caller
uint256 tokenId = 1;
uint256 quantity = 100;
bytes memory data = "";

IERC1155Mint(tokenContract).mint(tokenId, quantity, data);
console.log("Minted", quantity, "tokens of type", tokenId, "to caller");
```

#### `mintTo()`
```solidity
function mintTo(address recipient, uint256 id, uint256 quantity, bytes memory data) external
```

**Purpose**: Mint tokens of a specified type and quantity to a specific recipient address.

**Parameters**:
- `recipient` (address): The address to receive the minted tokens
- `id` (uint256): The token type identifier to mint
- `quantity` (uint256): The amount of tokens to mint
- `data` (bytes): Additional data to pass to the recipient (if it's a contract)

**Requirements**:
- Caller must have minting permissions
- Recipient address must not be zero address
- Token ID must be valid for minting
- Quantity must be greater than zero
- If recipient is a contract, it must implement ERC1155Receiver

**Events Emitted**:
- [`ERC1155TokenMinted`](ierc1155-mint.md:8) with minter address, token ID, and quantity
- Standard ERC1155 `TransferSingle` event

**Example Usage**:
```solidity
// Mint 50 tokens of type 2 to specific address
address recipient = 0x742d35Cc6634C0532925a3b8D4C9db96590c6C8C;
uint256 tokenId = 2;
uint256 quantity = 50;
bytes memory data = "";

IERC1155Mint(tokenContract).mintTo(recipient, tokenId, quantity, data);
console.log("Minted", quantity, "tokens of type", tokenId, "to", recipient);
```

### Batch Minting

#### `batchMintTo()`
```solidity
function batchMintTo(address recipient, uint256[] memory ids, uint256[] calldata quantities, bytes memory data) external
```

**Purpose**: Efficiently mint multiple token types with specified quantities to a recipient in a single transaction.

**Parameters**:
- `recipient` (address): The address to receive all minted tokens
- `ids` (uint256[]): Array of token type identifiers to mint
- `quantities` (uint256[]): Array of quantities corresponding to each token ID
- `data` (bytes): Additional data to pass to the recipient (if it's a contract)

**Requirements**:
- Caller must have minting permissions
- Recipient address must not be zero address
- Arrays must have the same length and not be empty
- All token IDs must be valid for minting
- All quantities must be greater than zero
- If recipient is a contract, it must implement ERC1155Receiver

**Events Emitted**:
- [`ERC1155TokenMinted`](ierc1155-mint.md:8) for each token type minted
- Standard ERC1155 `TransferBatch` event

**Gas Optimization**: More efficient than multiple individual mint calls

**Example Usage**:
```solidity
// Batch mint multiple token types
address recipient = 0x742d35Cc6634C0532925a3b8D4C9db96590c6C8C;

uint256[] memory tokenIds = new uint256[](3);
tokenIds[0] = 1; // Common item
tokenIds[1] = 2; // Rare item
tokenIds[2] = 3; // Epic item

uint256[] memory quantities = new uint256[](3);
quantities[0] = 100; // 100 common items
quantities[1] = 10;  // 10 rare items
quantities[2] = 1;   // 1 epic item

bytes memory data = "";

IERC1155Mint(tokenContract).batchMintTo(recipient, tokenIds, quantities, data);
console.log("Batch minted", tokenIds.length, "token types to", recipient);
```

## Integration Examples

### Gaming Item System
```solidity
// Gaming platform with different item types
contract GameItemSystem {
    IERC1155Mint public gameItems;
    
    // Item type definitions
    uint256 public constant COMMON_SWORD = 1;
    uint256 public constant RARE_SHIELD = 2;
    uint256 public constant EPIC_ARMOR = 3;
    uint256 public constant LEGENDARY_WEAPON = 4;
    
    // Item rarity and quantities
    mapping(uint256 => uint256) public itemRarity;
    mapping(uint256 => uint256) public maxSupply;
    mapping(uint256 => uint256) public currentSupply;
    
    event PlayerRewardMinted(address indexed player, uint256[] itemIds, uint256[] quantities);
    event QuestRewardDistributed(uint256 indexed questId, address[] players, uint256 rewardItemId);
    
    constructor(address _gameItems) {
        gameItems = IERC1155Mint(_gameItems);
        
        // Set up item rarities and max supplies
        itemRarity[COMMON_SWORD] = 1;
        itemRarity[RARE_SHIELD] = 2;
        itemRarity[EPIC_ARMOR] = 3;
        itemRarity[LEGENDARY_WEAPON] = 4;
        
        maxSupply[COMMON_SWORD] = 10000;
        maxSupply[RARE_SHIELD] = 1000;
        maxSupply[EPIC_ARMOR] = 100;
        maxSupply[LEGENDARY_WEAPON] = 10;
    }
    
    function mintStarterPack(address player) external onlyGameMaster {
        uint256[] memory itemIds = new uint256[](2);
        itemIds[0] = COMMON_SWORD;
        itemIds[1] = RARE_SHIELD;
        
        uint256[] memory quantities = new uint256[](2);
        quantities[0] = 1; // 1 common sword
        quantities[1] = 1; // 1 rare shield
        
        // Check supply limits
        for (uint256 i = 0; i < itemIds.length; i++) {
            require(
                currentSupply[itemIds[i]] + quantities[i] <= maxSupply[itemIds[i]],
                "Would exceed max supply"
            );
            currentSupply[itemIds[i]] += quantities[i];
        }
        
        gameItems.batchMintTo(player, itemIds, quantities, "");
        emit PlayerRewardMinted(player, itemIds, quantities);
    }
    
    function mintQuestReward(
        address[] memory players,
        uint256 rewardItemId,
        uint256 quantityPerPlayer
    ) external onlyGameMaster {
        require(players.length > 0, "No players specified");
        
        uint256 totalQuantity = players.length * quantityPerPlayer;
        require(
            currentSupply[rewardItemId] + totalQuantity <= maxSupply[rewardItemId],
            "Would exceed max supply"
        );
        
        currentSupply[rewardItemId] += totalQuantity;
        
        for (uint256 i = 0; i < players.length; i++) {
            gameItems.mintTo(players[i], rewardItemId, quantityPerPlayer, "");
        }
        
        emit QuestRewardDistributed(block.number, players, rewardItemId);
    }
    
    function mintSpecialEvent(
        address player,
        uint256 eventItemId,
        uint256 quantity,
        bytes memory eventData
    ) external onlyGameMaster {
        require(
            currentSupply[eventItemId] + quantity <= maxSupply[eventItemId],
            "Would exceed max supply"
        );
        
        currentSupply[eventItemId] += quantity;
        gameItems.mintTo(player, eventItemId, quantity, eventData);
    }
    
    function getItemInfo(uint256 itemId) external view returns (
        uint256 rarity,
        uint256 maxSupplyAmount,
        uint256 currentSupplyAmount,
        uint256 remainingSupply
    ) {
        rarity = itemRarity[itemId];
        maxSupplyAmount = maxSupply[itemId];
        currentSupplyAmount = currentSupply[itemId];
        remainingSupply = maxSupplyAmount - currentSupplyAmount;
    }
    
    modifier onlyGameMaster() {
        // Implementation would check for game master role
        _;
    }
}
```

### Digital Asset Marketplace
```solidity
// Marketplace for digital assets with minting capabilities
contract DigitalAssetMarketplace {
    IERC1155Mint public digitalAssets;
    
    struct AssetTemplate {
        string name;
        string description;
        uint256 basePrice;
        uint256 maxSupply;
        uint256 currentSupply;
        address creator;
        uint256 royaltyPercentage;
        bool active;
    }
    
    mapping(uint256 => AssetTemplate) public assetTemplates;
    mapping(address => bool) public authorizedCreators;
    uint256 public nextAssetId = 1;
    
    event AssetTemplateCreated(uint256 indexed assetId, string name, address indexed creator);
    event AssetMinted(uint256 indexed assetId, address indexed recipient, uint256 quantity);
    event CreatorAuthorized(address indexed creator, bool authorized);
    
    constructor(address _digitalAssets) {
        digitalAssets = IERC1155Mint(_digitalAssets);
    }
    
    function createAssetTemplate(
        string memory name,
        string memory description,
        uint256 basePrice,
        uint256 maxSupply,
        uint256 royaltyPercentage
    ) external returns (uint256 assetId) {
        require(authorizedCreators[msg.sender], "Not authorized creator");
        require(maxSupply > 0, "Max supply must be greater than 0");
        require(royaltyPercentage <= 1000, "Royalty too high"); // Max 10%
        
        assetId = nextAssetId++;
        
        assetTemplates[assetId] = AssetTemplate({
            name: name,
            description: description,
            basePrice: basePrice,
            maxSupply: maxSupply,
            currentSupply: 0,
            creator: msg.sender,
            royaltyPercentage: royaltyPercentage,
            active: true
        });
        
        emit AssetTemplateCreated(assetId, name, msg.sender);
    }
    
    function mintAsset(
        uint256 assetId,
        address recipient,
        uint256 quantity
    ) external payable {
        AssetTemplate storage template = assetTemplates[assetId];
        require(template.active, "Asset template not active");
        require(template.currentSupply + quantity <= template.maxSupply, "Would exceed max supply");
        
        uint256 totalCost = template.basePrice * quantity;
        require(msg.value >= totalCost, "Insufficient payment");
        
        // Update supply
        template.currentSupply += quantity;
        
        // Mint the assets
        digitalAssets.mintTo(recipient, assetId, quantity, "");
        
        // Handle payments
        uint256 royalty = (totalCost * template.royaltyPercentage) / 10000;
        uint256 platformFee = totalCost - royalty;
        
        if (royalty > 0) {
            payable(template.creator).transfer(royalty);
        }
        
        // Platform keeps the rest (simplified - would have more complex fee structure)
        
        emit AssetMinted(assetId, recipient, quantity);
    }
    
    function batchMintAssets(
        uint256[] memory assetIds,
        uint256[] memory quantities,
        address recipient
    ) external payable {
        require(assetIds.length == quantities.length, "Array length mismatch");
        require(assetIds.length > 0, "No assets specified");
        
        uint256 totalCost = 0;
        
        // Validate and calculate total cost
        for (uint256 i = 0; i < assetIds.length; i++) {
            AssetTemplate storage template = assetTemplates[assetIds[i]];
            require(template.active, "Asset template not active");
            require(
                template.currentSupply + quantities[i] <= template.maxSupply,
                "Would exceed max supply"
            );
            
            totalCost += template.basePrice * quantities[i];
            template.currentSupply += quantities[i];
        }
        
        require(msg.value >= totalCost, "Insufficient payment");
        
        // Mint all assets
        digitalAssets.batchMintTo(recipient, assetIds, quantities, "");
        
        // Handle payments (simplified)
        // In practice, would calculate individual royalties per creator
        
        for (uint256 i = 0; i < assetIds.length; i++) {
            emit AssetMinted(assetIds[i], recipient, quantities[i]);
        }
    }
    
    function setCreatorAuthorization(address creator, bool authorized) external onlyOwner {
        authorizedCreators[creator] = authorized;
        emit CreatorAuthorized(creator, authorized);
    }
    
    function getAssetInfo(uint256 assetId) external view returns (
        AssetTemplate memory template,
        uint256 remainingSupply
    ) {
        template = assetTemplates[assetId];
        remainingSupply = template.maxSupply - template.currentSupply;
    }
    
    modifier onlyOwner() {
        // Implementation would check for owner role
        _;
    }
}
```

### Collectible Card System
```solidity
// Trading card game with different card rarities
contract CollectibleCardSystem {
    IERC1155Mint public gameCards;
    
    enum CardRarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY }
    
    struct CardTemplate {
        string name;
        CardRarity rarity;
        uint256 attack;
        uint256 defense;
        uint256 cost;
        string artwork;
        bool active;
    }
    
    mapping(uint256 => CardTemplate) public cardTemplates;
    mapping(CardRarity => uint256) public packProbabilities;
    mapping(address => uint256) public playerPacksOpened;
    
    uint256 public nextCardId = 1;
    uint256 public packPrice = 0.01 ether;
    
    event CardTemplateCreated(uint256 indexed cardId, string name, CardRarity rarity);
    event PackOpened(address indexed player, uint256[] cardIds, uint256[] quantities);
    event CardMinted(uint256 indexed cardId, address indexed recipient, uint256 quantity);
    
    constructor(address _gameCards) {
        gameCards = IERC1155Mint(_gameCards);
        
        // Set pack probabilities (out of 10000)
        packProbabilities[CardRarity.COMMON] = 5000;    // 50%
        packProbabilities[CardRarity.UNCOMMON] = 3000;  // 30%
        packProbabilities[CardRarity.RARE] = 1500;      // 15%
        packProbabilities[CardRarity.EPIC] = 450;       // 4.5%
        packProbabilities[CardRarity.LEGENDARY] = 50;   // 0.5%
    }
    
    function createCardTemplate(
        string memory name,
        CardRarity rarity,
        uint256 attack,
        uint256 defense,
        uint256 cost,
        string memory artwork
    ) external onlyGameDesigner returns (uint256 cardId) {
        cardId = nextCardId++;
        
        cardTemplates[cardId] = CardTemplate({
            name: name,
            rarity: rarity,
            attack: attack,
            defense: defense,
            cost: cost,
            artwork: artwork,
            active: true
        });
        
        emit CardTemplateCreated(cardId, name, rarity);
    }
    
    function openBoosterPack(address player) external payable {
        require(msg.value >= packPrice, "Insufficient payment for pack");
        
        uint256[] memory cardIds = new uint256[](5); // 5 cards per pack
        uint256[] memory quantities = new uint256[](5);
        
        // Generate 5 random cards based on rarity probabilities
        for (uint256 i = 0; i < 5; i++) {
            cardIds[i] = _generateRandomCard();
            quantities[i] = 1;
        }
        
        // Mint the cards
        gameCards.batchMintTo(player, cardIds, quantities, "");
        
        playerPacksOpened[player]++;
        emit PackOpened(player, cardIds, quantities);
    }
    
    function mintSpecialCard(
        address recipient,
        uint256 cardId,
        uint256 quantity,
        bytes memory eventData
    ) external onlyGameDesigner {
        require(
            currentSupply[cardId] + quantity <= maxSupply[cardId],
            "Would exceed max supply"
        );
        
        currentSupply[cardId] += quantity;
        gameCards.mintTo(recipient, cardId, quantity, eventData);
    }
    
    function getItemInfo(uint256 itemId) external view returns (
        uint256 rarity,
        uint256 maxSupplyAmount,
        uint256 currentSupplyAmount,
        uint256 remainingSupply
    ) {
        rarity = itemRarity[itemId];
        maxSupplyAmount = maxSupply[itemId];
        currentSupplyAmount = currentSupply[itemId];
        remainingSupply = maxSupplyAmount - currentSupplyAmount;
    }
    
    modifier onlyGameDesigner() {
        // Implementation would check for game master role
        _;
    }
}
```

## Security Considerations

### Access Control
- **Authorized Minting**: Only authorized entities can mint tokens
- **Recipient Validation**: Validate recipient addresses
- **Supply Limits**: Enforce max supply per token type

### Supply Management
- **Controlled Minting**: Prevent over-minting
- **Supply Tracking**: Track current supply and remaining supply
- **Burn Functions**: Provide burn functionality for tokens (if needed)

### Recipient Validation
- **ERC1155Receiver**: Ensure contract recipients implement ERC1155Receiver
- **Zero Address Check**: Prevent minting to zero address

## Gas Optimization

### Efficient Operations
- **Batch Minting**: `batchMintTo` for efficient multi-token minting
- **Single Transactions**: Minimize multiple transaction calls
- **Gas-Efficient Logic**: Optimize internal minting logic

### Storage Optimization
- **Minimal Storage**: Store only essential token data
- **Packed Structs**: Use packed structs for data structures
- **Mapping Usage**: Efficient use of mappings for fast lookups

## Error Handling

### Common Errors
- `IERC1155Mint: Unauthorized Minter`: Caller does not have minting permissions
- `IERC1155Mint: Invalid Recipient`: Recipient address is zero or invalid
- `IERC1155Mint: Invalid Token ID`: Token ID is invalid or not configured
- `IERC1155Mint: Quantity Exceeds Supply`: Minting would exceed max supply
- `IERC1155Mint: Zero Quantity`: Attempting to mint zero tokens

## Best Practices

### Integration Checklist
- Implement robust access control for minting functions
- Integrate with off-chain systems for supply management and analytics
- Validate all input parameters before minting
- Handle ERC1155Receiver callbacks for contract recipients

### Development Guidelines
- Write comprehensive unit tests for all minting scenarios
- Monitor events for real-time tracking of token supply
- Use secure random number generation for unique token IDs (if applicable)
- Consider upgradability for future token features

## Related Documentation

- [Multi Sale Facet](../../smart-contracts/facets/multi-sale-facet.md) - For multi-token sales using ERC1155.
- [ERC721A Interface](ierc721a.md) - For comparison with ERC721 minting.
- [SDK & Libraries: Blockchain Utilities](../../sdk-libraries/blockchain.md) - General blockchain interaction utilities.
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)
- [Developer Guides: Development Environment Setup](../../developer-guides/development-environment-setup.md)