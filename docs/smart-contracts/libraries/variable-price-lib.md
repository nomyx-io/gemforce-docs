# VariablePriceLib Library

## Overview

The [`VariablePriceLib`](../../../contracts/libraries/VariablePriceLib.sol) library provides core utilities for managing dynamic pricing mechanisms within the Gemforce platform. This library implements flexible pricing strategies that automatically adjust prices based on configurable modifiers, supporting fixed increments, exponential growth, and inverse logarithmic scaling for various tokenomics scenarios.

## Key Features

- **Dynamic Pricing**: Automatic price adjustments after each transaction
- **Multiple Price Modifiers**: Fixed, exponential, and inverse logarithmic pricing strategies
- **Configurable Parameters**: Customizable price modifier factors and maximum price limits
- **Gas Efficient**: Optimized calculations for on-chain price updates
- **Event Tracking**: Price change notifications for external monitoring
- **Integration Ready**: Seamless integration with token sales and marketplace systems

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library VariablePriceLib {
    // Storage management
    function variablePriceStorage() internal pure returns (VariablePriceStorage storage);
    
    // Price operations
    function _updatePrice(VariablePriceContract storage self) internal returns (uint256 _price, uint256 updatedPrice);
    function _currentPrice(VariablePriceContract storage self) internal view returns (uint256 _price);
    function _setPrice(VariablePriceContract storage self, uint256 _price) internal returns (uint256 _newPrice);
    
    // Price modification
    function _increaseClaimPrice(VariablePriceContract storage self) internal;
}
```

## Data Structures

### PriceModifier Enum
```solidity
enum PriceModifier {
    None,           // No price modification
    Fixed,          // Fixed amount increase
    Exponential,    // Exponential price growth
    InverseLog      // Inverse logarithmic scaling
}
```

**Purpose**: Defines the pricing strategy for automatic price adjustments.

**Strategies**:
- **None**: Price remains constant
- **Fixed**: Price increases by a fixed amount each time
- **Exponential**: Price increases by a percentage of current price
- **InverseLog**: Price increases diminish as price grows (dampening effect)

### VariablePriceContract Struct
```solidity
struct VariablePriceContract {
    uint256 price;                      // Current price of the token
    PriceModifier priceModifier;        // Price modification strategy
    uint256 priceModifierFactor;        // Factor for price calculations
    uint256 maxPrice;                   // Maximum price limit
}
```

**Purpose**: Complete configuration for variable pricing behavior.

**Components**:
- **price**: Current token price in wei
- **priceModifier**: Strategy for price increases
- **priceModifierFactor**: Calculation parameter for price adjustments
- **maxPrice**: Upper limit to prevent excessive pricing

### VariablePriceStorage Struct
```solidity
struct VariablePriceStorage {
    VariablePriceContract variablePrices;
}
```

**Purpose**: Diamond storage wrapper for variable pricing data.

**Storage Position**:
```solidity
bytes32 internal constant DIAMOND_STORAGE_POSITION = 
    keccak256("diamond.nextblock.bitgem.app.VariablePriceStorage.storage");
```

## Core Functions

### Storage Management

#### `variablePriceStorage()`
```solidity
function variablePriceStorage() internal pure returns (VariablePriceStorage storage ds)
```

**Purpose**: Access Diamond storage for variable pricing data using assembly.

**Implementation**:
```solidity
function variablePriceStorage() internal pure returns (VariablePriceStorage storage ds) {
    bytes32 position = DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
}
```

**Returns**: Storage reference to variable pricing data

**Usage**: All pricing functions use this to access persistent storage.

### Price Operations

#### `_updatePrice()`
```solidity
function _updatePrice(VariablePriceContract storage self) internal returns (uint256 _price, uint256 updatedPrice)
```

**Purpose**: Update the price and return both old and new values.

**Parameters**:
- `self`: Storage reference to variable price contract

**Returns**:
- `_price`: Previous price before update
- `updatedPrice`: New price after increase

**Process**:
1. Capture current price
2. Apply price increase based on modifier
3. Return both old and new prices

**Example Usage**:
```solidity
// Update price after a sale
(uint256 oldPrice, uint256 newPrice) = VariablePriceLib._updatePrice(priceContract);
console.log("Price updated from", oldPrice, "to", newPrice);
```

#### `_currentPrice()`
```solidity
function _currentPrice(VariablePriceContract storage self) internal view returns (uint256 _price)
```

**Purpose**: Get the current price without modification.

**Parameters**:
- `self`: Storage reference to variable price contract

**Returns**: Current price in wei

**Example Usage**:
```solidity
// Check current price before purchase
uint256 currentPrice = VariablePriceLib._currentPrice(priceContract);
require(msg.value >= currentPrice, "Insufficient payment");
```

#### `_setPrice()`
```solidity
function _setPrice(VariablePriceContract storage self, uint256 _price) internal returns (uint256 _newPrice)
```

**Purpose**: Manually set a new price value.

**Parameters**:
- `self`: Storage reference to variable price contract
- `_price`: New price to set

**Returns**: The newly set price

**Use Cases**:
- Initial price configuration
- Administrative price adjustments
- Price resets or corrections

**Example Usage**:
```solidity
// Set initial price for a new token sale
uint256 initialPrice = 0.1 ether;
VariablePriceLib._setPrice(priceContract, initialPrice);
```

### Price Modification

#### `_increaseClaimPrice()`
```solidity
function _increaseClaimPrice(VariablePriceContract storage self) internal
```

**Purpose**: Apply price increase based on the configured price modifier.

**Parameters**:
- `self`: Storage reference to variable price contract

**Price Modification Logic**:

**Fixed Modifier**:
```solidity
if (currentModifier == PriceModifier.Fixed) {
    currentPrice = currentPrice + currentModifierFactor;
}
```
- Increases price by a fixed amount each time
- `priceModifierFactor` = fixed increase amount in wei

**Exponential Modifier**:
```solidity
else if (currentModifier == PriceModifier.Exponential) {
    currentPrice = currentPrice + (currentPrice / currentModifierFactor);
}
```
- Increases price by a percentage of current price
- `priceModifierFactor` = divisor for percentage calculation
- Example: factor of 10 = 10% increase, factor of 5 = 20% increase

**Inverse Logarithmic Modifier**:
```solidity
else if (currentModifier == PriceModifier.InverseLog) {
    currentPrice = currentPrice + (currentPrice / (currentModifierFactor * currentPrice));
}
```
- Diminishing price increases as price grows
- Creates a dampening effect for high-value items
- `priceModifierFactor` affects the rate of dampening

**Example Usage**:
```solidity
// Apply price increase after successful purchase
VariablePriceLib._increaseClaimPrice(priceContract);
emit PriceIncreased(tokenId, newPrice);
```

## Integration Examples

### NFT Collection with Bonding Curve
```solidity
// NFT collection with exponential pricing
contract BondingCurveNFT {
    using VariablePriceLib for VariablePriceLib.VariablePriceStorage;
    
    struct CollectionConfig {
        uint256 startPrice;
        uint256 maxPrice;
        PriceModifier priceStrategy;
        uint256 modifierFactor;
        uint256 maxSupply;
        uint256 currentSupply;
    }
    
    CollectionConfig public config;
    mapping(uint256 => uint256) public tokenPrices;
    
    event TokenMinted(uint256 indexed tokenId, address indexed to, uint256 price);
    event PriceUpdated(uint256 oldPrice, uint256 newPrice);
    
    constructor(
        uint256 _startPrice,
        uint256 _maxPrice,
        PriceModifier _priceStrategy,
        uint256 _modifierFactor,
        uint256 _maxSupply
    ) {
        config = CollectionConfig({
            startPrice: _startPrice,
            maxPrice: _maxPrice,
            priceStrategy: _priceStrategy,
            modifierFactor: _modifierFactor,
            maxSupply: _maxSupply,
            currentSupply: 0
        });
        
        // Initialize pricing
        VariablePriceLib.VariablePriceStorage storage priceStorage = VariablePriceLib.variablePriceStorage();
        priceStorage.variablePrices = VariablePriceContract({
            price: _startPrice,
            priceModifier: _priceStrategy,
            priceModifierFactor: _modifierFactor,
            maxPrice: _maxPrice
        });
    }
    
    function mint(address to) external payable returns (uint256 tokenId) {
        require(config.currentSupply < config.maxSupply, "Max supply reached");
        
        VariablePriceLib.VariablePriceStorage storage priceStorage = VariablePriceLib.variablePriceStorage();
        
        // Get current price
        uint256 currentPrice = VariablePriceLib._currentPrice(priceStorage.variablePrices);
        require(msg.value >= currentPrice, "Insufficient payment");
        
        // Mint token
        tokenId = config.currentSupply + 1;
        config.currentSupply++;
        _mint(to, tokenId);
        
        // Record purchase price
        tokenPrices[tokenId] = currentPrice;
        
        // Update price for next mint
        (uint256 oldPrice, uint256 newPrice) = VariablePriceLib._updatePrice(priceStorage.variablePrices);
        
        // Refund excess payment
        if (msg.value > currentPrice) {
            payable(msg.sender).transfer(msg.value - currentPrice);
        }
        
        emit TokenMinted(tokenId, to, currentPrice);
        emit PriceUpdated(oldPrice, newPrice);
    }
    
    function getCurrentPrice() external view returns (uint256) {
        VariablePriceLib.VariablePriceStorage storage priceStorage = VariablePriceLib.variablePriceStorage();
        return VariablePriceLib._currentPrice(priceStorage.variablePrices);
    }
    
    function getNextPrice() external view returns (uint256) {
        VariablePriceLib.VariablePriceStorage storage priceStorage = VariablePriceLib.variablePriceStorage();
        
        // Simulate price increase
        uint256 currentPrice = priceStorage.variablePrices.price;
        PriceModifier modifier = priceStorage.variablePrices.priceModifier;
        uint256 factor = priceStorage.variablePrices.priceModifierFactor;
        
        if (modifier == PriceModifier.Fixed) {
            return currentPrice + factor;
        } else if (modifier == PriceModifier.Exponential) {
            return currentPrice + (currentPrice / factor);
        } else if (modifier == PriceModifier.InverseLog) {
            return currentPrice + (currentPrice / (factor * currentPrice));
        }
        
        return currentPrice;
    }
    
    function getPriceHistory() external view returns (uint256[] memory prices) {
        prices = new uint256[](config.currentSupply);
        for (uint256 i = 1; i <= config.currentSupply; i++) {
            prices[i - 1] = tokenPrices[i];
        }
    }
    
    function _mint(address to, uint256 tokenId) internal {
        // ERC721 mint implementation
    }
}
```

### Gaming Item Marketplace
```solidity
// Gaming marketplace with dynamic item pricing
contract GameItemMarketplace {
    using VariablePriceLib for VariablePriceLib.VariablePriceStorage;
    
    struct ItemType {
        string name;
        string category;
        VariablePriceContract pricing;
        uint256 totalSold;
        bool active;
    }
    
    struct PurchaseHistory {
        uint256 itemTypeId;
        uint256 price;
        uint256 timestamp;
        address buyer;
    }
    
    mapping(uint256 => ItemType) public itemTypes;
    mapping(uint256 => PurchaseHistory[]) public purchaseHistory;
    uint256 public nextItemTypeId;
    
    event ItemTypeCreated(uint256 indexed itemTypeId, string name, uint256 startPrice);
    event ItemPurchased(uint256 indexed itemTypeId, address indexed buyer, uint256 price);
    event PricingUpdated(uint256 indexed itemTypeId, uint256 oldPrice, uint256 newPrice);
    
    function createItemType(
        string memory name,
        string memory category,
        uint256 startPrice,
        uint256 maxPrice,
        PriceModifier priceStrategy,
        uint256 modifierFactor
    ) external onlyGameMaster returns (uint256 itemTypeId) {
        itemTypeId = nextItemTypeId++;
        
        itemTypes[itemTypeId] = ItemType({
            name: name,
            category: category,
            pricing: VariablePriceContract({
                price: startPrice,
                priceModifier: priceStrategy,
                priceModifierFactor: modifierFactor,
                maxPrice: maxPrice
            }),
            totalSold: 0,
            active: true
        });
        
        emit ItemTypeCreated(itemTypeId, name, startPrice);
    }
    
    function purchaseItem(uint256 itemTypeId) external payable returns (uint256 itemId) {
        ItemType storage itemType = itemTypes[itemTypeId];
        require(itemType.active, "Item type not active");
        
        // Get current price
        uint256 currentPrice = VariablePriceLib._currentPrice(itemType.pricing);
        require(msg.value >= currentPrice, "Insufficient payment");
        
        // Update price
        (uint256 oldPrice, uint256 newPrice) = VariablePriceLib._updatePrice(itemType.pricing);
        
        // Mint item to buyer
        itemId = _mintGameItem(msg.sender, itemTypeId);
        
        // Update statistics
        itemType.totalSold++;
        
        // Record purchase
        purchaseHistory[itemTypeId].push(PurchaseHistory({
            itemTypeId: itemTypeId,
            price: currentPrice,
            timestamp: block.timestamp,
            buyer: msg.sender
        }));
        
        // Refund excess payment
        if (msg.value > currentPrice) {
            payable(msg.sender).transfer(msg.value - currentPrice);
        }
        
        emit ItemPurchased(itemTypeId, msg.sender, currentPrice);
        emit PricingUpdated(itemTypeId, oldPrice, newPrice);
    }
    
    function getItemPrice(uint256 itemTypeId) external view returns (uint256) {
        return VariablePriceLib._currentPrice(itemTypes[itemTypeId].pricing);
    }
    
    function getItemPriceInfo(uint256 itemTypeId) external view returns (
        uint256 currentPrice,
        uint256 nextPrice,
        PriceModifier strategy,
        uint256 factor,
        uint256 maxPrice,
        uint256 totalSold
    ) {
        ItemType memory itemType = itemTypes[itemTypeId];
        currentPrice = itemType.pricing.price;
        strategy = itemType.pricing.priceModifier;
        factor = itemType.pricing.priceModifierFactor;
        maxPrice = itemType.pricing.maxPrice;
        totalSold = itemType.totalSold;
        
        // Calculate next price
        if (strategy == PriceModifier.Fixed) {
            nextPrice = currentPrice + factor;
        } else if (strategy == PriceModifier.Exponential) {
            nextPrice = currentPrice + (currentPrice / factor);
        } else if (strategy == PriceModifier.InverseLog) {
            nextPrice = currentPrice + (currentPrice / (factor * currentPrice));
        } else {
            nextPrice = currentPrice;
        }
        
        // Cap at max price
        if (nextPrice > maxPrice) {
            nextPrice = maxPrice;
        }
    }
    
    function getPurchaseHistory(uint256 itemTypeId) external view returns (PurchaseHistory[] memory) {
        return purchaseHistory[itemTypeId];
    }
    
    function updateItemPricing(
        uint256 itemTypeId,
        uint256 newPrice,
        PriceModifier newStrategy,
        uint256 newFactor,
        uint256 newMaxPrice
    ) external onlyGameMaster {
        ItemType storage itemType = itemTypes[itemTypeId];
        require(itemType.active, "Item type not active");
        
        itemType.pricing.price = newPrice;
        itemType.pricing.priceModifier = newStrategy;
        itemType.pricing.priceModifierFactor = newFactor;
        itemType.pricing.maxPrice = newMaxPrice;
    }
    
    function _mintGameItem(address to, uint256 itemTypeId) internal returns (uint256) {
        // Implementation would mint game item NFT
        return 1;
    }
    
    modifier onlyGameMaster() {
        // Implementation would check game master role
        _;
    }
}
```

### Auction System with Dynamic Reserve Prices
```solidity
// Auction system with variable reserve pricing
contract DynamicAuction {
    using VariablePriceLib for VariablePriceLib.VariablePriceStorage;
    
    struct Auction {
        uint256 tokenId;
        address seller;
        VariablePriceContract reservePricing;
        uint256 startTime;
        uint256 endTime;
        address highestBidder;
        uint256 highestBid;
        bool ended;
        uint256 auctionCount; // Number of times this item has been auctioned
    }
    
    struct Bid {
        address bidder;
        uint256 amount;
        uint256 timestamp;
    }
    
    mapping(uint256 => Auction) public auctions;
    mapping(uint256 => Bid[]) public auctionBids;
    mapping(uint256 => bool) public tokenInAuction;
    uint256 public nextAuctionId;
    
    event AuctionCreated(uint256 indexed auctionId, uint256 indexed tokenId, uint256 reservePrice);
    event BidPlaced(uint256 indexed auctionId, address indexed bidder, uint256 amount);
    event AuctionEnded(uint256 indexed auctionId, address indexed winner, uint256 finalPrice);
    event ReservePriceUpdated(uint256 indexed auctionId, uint256 newReservePrice);
    
    function createAuction(
        uint256 tokenId,
        uint256 duration,
        uint256 initialReservePrice,
        PriceModifier reserveStrategy,
        uint256 reserveFactor,
        uint256 maxReservePrice
    ) external returns (uint256 auctionId) {
        require(!tokenInAuction[tokenId], "Token already in auction");
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        
        auctionId = nextAuctionId++;
        
        auctions[auctionId] = Auction({
            tokenId: tokenId,
            seller: msg.sender,
            reservePricing: VariablePriceContract({
                price: initialReservePrice,
                priceModifier: reserveStrategy,
                priceModifierFactor: reserveFactor,
                maxPrice: maxReservePrice
            }),
            startTime: block.timestamp,
            endTime: block.timestamp + duration,
            highestBidder: address(0),
            highestBid: 0,
            ended: false,
            auctionCount: _getAuctionCount(tokenId) + 1
        });
        
        tokenInAuction[tokenId] = true;
        
        emit AuctionCreated(auctionId, tokenId, initialReservePrice);
    }
    
    function placeBid(uint256 auctionId) external payable {
        Auction storage auction = auctions[auctionId];
        require(block.timestamp < auction.endTime, "Auction ended");
        require(!auction.ended, "Auction already ended");
        
        uint256 currentReserve = VariablePriceLib._currentPrice(auction.reservePricing);
        require(msg.value >= currentReserve, "Bid below reserve price");
        require(msg.value > auction.highestBid, "Bid too low");
        
        // Refund previous highest bidder
        if (auction.highestBidder != address(0)) {
            payable(auction.highestBidder).transfer(auction.highestBid);
        }
        
        auction.highestBidder = msg.sender;
        auction.highestBid = msg.value;
        
        // Record bid
        auctionBids[auctionId].push(Bid({
            bidder: msg.sender,
            amount: msg.value,
            timestamp: block.timestamp
        }));
        
        // Increase reserve price for future auctions of this item
        (uint256 oldReserve, uint256 newReserve) = VariablePriceLib._updatePrice(auction.reservePricing);
        
        emit BidPlaced(auctionId, msg.sender, msg.value);
        emit ReservePriceUpdated(auctionId, newReserve);
    }
    
    function endAuction(uint256 auctionId) external {
        Auction storage auction = auctions[auctionId];
        require(block.timestamp >= auction.endTime || msg.sender == auction.seller, "Auction not ended");
        require(!auction.ended, "Auction already ended");
        
        auction.ended = true;
        tokenInAuction[auction.tokenId] = false;
        
        if (auction.highestBidder != address(0)) {
            // Transfer token to winner
            _transfer(auction.seller, auction.highestBidder, auction.tokenId);
            
            // Transfer payment to seller (minus fees)
            uint256 fee = auction.highestBid * 25 / 1000; // 2.5% fee
            payable(auction.seller).transfer(auction.highestBid - fee);
            
            emit AuctionEnded(auctionId, auction.highestBidder, auction.highestBid);
        } else {
            // No bids, return token to seller
            emit AuctionEnded(auctionId, address(0), 0);
        }
    }
    
    function getCurrentReservePrice(uint256 auctionId) external view returns (uint256) {
        return VariablePriceLib._currentPrice(auctions[auctionId].reservePricing);
    }
    
    function getAuctionInfo(uint256 auctionId) external view returns (
        uint256 tokenId,
        address seller,
        uint256 currentReserve,
        uint256 nextReserve,
        uint256 startTime,
        uint256 endTime,
        address highestBidder,
        uint256 highestBid,
        bool ended,
        uint256 auctionCount
    ) {
        Auction memory auction = auctions[auctionId];
        tokenId = auction.tokenId;
        seller = auction.seller;
        currentReserve = auction.reservePricing.price;
        startTime = auction.startTime;
        endTime = auction.endTime;
        highestBidder = auction.highestBidder;
        highestBid = auction.highestBid;
        ended = auction.ended;
        auctionCount = auction.auctionCount;
        
        // Calculate next reserve price
        PriceModifier strategy = auction.reservePricing.priceModifier;
        uint256 factor = auction.reservePricing.priceModifierFactor;
        
        if (strategy == PriceModifier.Fixed) {
            nextReserve = currentReserve + factor;
        } else if (strategy == PriceModifier.Exponential) {
            nextReserve = currentReserve + (currentReserve / factor);
        } else if (strategy == PriceModifier.InverseLog) {
            nextReserve = currentReserve + (currentReserve / (factor * currentReserve));
        } else {
            nextReserve = currentReserve;
        }
        
        if (nextReserve > auction.reservePricing.maxPrice) {
            nextReserve = auction.reservePricing.maxPrice;
        }
    }
    
    function getBidHistory(uint256 auctionId) external view returns (Bid[] memory) {
        return auctionBids[auctionId];
    }
    
    function _getAuctionCount(uint256 tokenId) internal view returns (uint256) {
        // Implementation would track auction history
        return 0;
    }
    
    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        // Implementation would check ERC721 authorization
        return true;
    }
    
    function _transfer(address from, address to, uint256 tokenId) internal {
        // Implementation would transfer ERC721 token
    }
}
```

## Events

### Variable Price Events
```solidity
event VariablePriceChanged(address eventContract, VariablePriceContract price);
```

## Security Considerations

### Price Manipulation Protection
- Maximum price limits prevent excessive pricing
- Configurable modifier factors allow controlled growth
- Transparent pricing algorithms prevent manipulation
- Event emission for price change monitoring

### Overflow Protection
- Safe arithmetic operations for price calculations
- Bounds checking for modifier factors
- Maximum price enforcement
- Graceful handling of edge cases

### Economic Security
- Predictable pricing algorithms
- Configurable parameters for different use cases
- Protection against price manipulation attacks
- Fair pricing progression for all participants

## Gas Optimization

### Calculation Efficiency
- Optimized arithmetic operations
- Minimal storage reads and writes
- Efficient price update algorithms
- Batch price operations where possible

### Storage Efficiency
- Compact data structures
- Minimal storage footprint
- Efficient parameter encoding
- Optimized Diamond storage access

## Error Handling

### Common Errors
- Division by zero in modifier calculations
- Price overflow in exponential growth
- Invalid modifier factor values
- Maximum price exceeded

### Best Practices
- Validate modifier factors before use
- Implement maximum price limits
- Handle edge cases gracefully
- Provide clear error messages for debugging

## Testing Considerations

### Unit Tests
- Price calculation accuracy for each modifier type
- Boundary condition testing
- Overflow and underflow protection
- Storage operations verification

### Integration Tests
- Multi-transaction pricing scenarios
- Cross-contract price integration
- Economic model validation
- Long-term pricing behavior analysis

### Economic Testing
- Bonding curve validation
- Market dynamics simulation
- Price discovery mechanisms
- Tokenomics model verification

## Related Documentation

- [IVariablePrice Interface](../interfaces/ivariable-price.md) - Variable price interface definition
- [MultiSaleLib](multi-sale-lib.md) - Multi-token sale integration
- [Pricing Strategies Guide](../../guides/pricing-strategies.md) - Economic model implementation
- [Tokenomics Guide](../../guides/tokenomics.md) - Token economics best practices
- [Auction Systems Guide](../../guides/auction-systems.md) - Auction implementation patterns

---

*This library provides comprehensive utilities for dynamic pricing mechanisms within the Gemforce platform, supporting flexible pricing strategies, automatic price adjustments, and configurable economic models for various tokenomics scenarios including bonding curves, auctions, and marketplace systems.*