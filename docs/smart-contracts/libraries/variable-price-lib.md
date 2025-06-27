# VariablePriceLib Library

## Overview

The [`VariablePriceLib`](../../smart-contracts/libraries/variable-price-lib.md) library provides core utilities for managing dynamic pricing mechanisms within the Gemforce platform. This library implements flexible pricing strategies that automatically adjust prices based on configurable modifiers, supporting fixed increments, exponential growth, and inverse logarithmic scaling for various tokenomics scenarios.

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
        // Placeholder for access control
        _;
    }
}
```

### Auction System with Dynamic Reserve Prices
```solidity
// Auction contract with reserve price adjusted dynamically
contract DynamicReserveAuction {
    using VariablePriceLib for VariablePriceLib.VariablePriceStorage;
    
    struct AuctionConfig {
        uint256 startingBid;
        uint256 reservePrice;
        uint256 buyNowPrice;
        uint256 auctionEndTime;
        VariablePriceContract reservePricing; // Dynamic reserve price
        bool active;
    }
    
    mapping(uint256 => AuctionConfig) public auctions;
    uint256 public nextAuctionId;
    
    event AuctionCreated(uint256 indexed auctionId, uint256 startingBid, uint256 reservePrice);
    event BidPlaced(uint256 indexed auctionId, address indexed bidder, uint256 bidAmount);
    event AuctionEnded(uint256 indexed auctionId, address winner, uint256 finalPrice);
    event ReservePriceUpdated(uint256 indexed auctionId, uint256 oldPrice, uint256 newPrice);
    
    function createAuction(
        uint256 startingBid,
        uint256 reservePrice,
        uint256 buyNowPrice,
        uint256 auctionDuration,
        PriceModifier priceStrategy,
        uint256 modifierFactor,
        uint256 maxReservePrice
    ) external onlyOwner returns (uint256 auctionId) {
        auctionId = nextAuctionId++;
        
        auctions[auctionId] = AuctionConfig({
            startingBid: startingBid,
            reservePrice: reservePrice,
            buyNowPrice: buyNowPrice,
            auctionEndTime: block.timestamp + auctionDuration,
            reservePricing: VariablePriceContract({
                price: reservePrice,
                priceModifier: priceStrategy,
                priceModifierFactor: modifierFactor,
                maxPrice: maxReservePrice
            }),
            active: true
        });
        
        emit AuctionCreated(auctionId, startingBid, reservePrice);
    }
    
    function placeBid(uint256 auctionId) external payable {
        AuctionConfig storage auction = auctions[auctionId];
        require(auction.active, "Auction not active");
        require(block.timestamp <= auction.auctionEndTime, "Auction ended");
        
        uint256 currentBid = getCurrentBid(auctionId);
        require(msg.value > currentBid, "Bid too low");
        
        // Update reserve price (e.g., after each bid)
        (uint256 oldReserve, uint256 newReserve) = VariablePriceLib._updatePrice(auction.reservePricing);
        emit ReservePriceUpdated(auctionId, oldReserve, newReserve);
        
        _recordBid(auctionId, msg.sender, msg.value);
        emit BidPlaced(auctionId, msg.sender, msg.value);
    }
    
    function endAuction(uint256 auctionId) external returns (address winner, uint256 finalPrice) {
        AuctionConfig storage auction = auctions[auctionId];
        require(auction.active, "Auction not active");
        require(block.timestamp > auction.auctionEndTime, "Auction not ended");
        
        // Determine winner and final price
        (winner, finalPrice) = _determineWinner(auctionId);
        
        auction.active = false;
        
        _transferAssetToWinner(winner, auctionId);
        _distributeFunds(finalPrice, auction.reservePrice, winner);
        
        emit AuctionEnded(auctionId, winner, finalPrice);
    }
    
    function getReservePrice(uint256 auctionId) external view returns (uint256) {
        return VariablePriceLib._currentPrice(auctions[auctionId].reservePricing);
    }
    
    function getAuctionInfo(uint256 auctionId) external view returns (
        uint256 currentBid,
        uint256 reserve,
        uint256 buyNow,
        uint256 endTime,
        bool active
    ) {
        AuctionConfig memory config = auctions[auctionId];
        currentBid = getCurrentBid(auctionId);
        reserve = VariablePriceLib._currentPrice(config.reservePricing);
        buyNow = config.buyNowPrice;
        endTime = config.auctionEndTime;
        active = config.active;
    }
    
    function getCurrentBid(uint256 auctionId) internal view returns (uint256) {
        // Placeholder for internal logic to get highest bid
        return 0;
    }
    
    function _recordBid(uint256 auctionId, address bidder, uint256 bidAmount) internal {
        // Placeholder for bid storage
    }
    
    function _determineWinner(uint256 auctionId) internal view returns (address, uint256) {
        // Placeholder for winner determination
        return (address(0), 0);
    }
    
    function _transferAssetToWinner(address winner, uint256 auctionId) internal {
        // Placeholder for asset transfer
    }
    
    function _distributeFunds(uint256 amount, uint256 reserve, address winner) internal {
        // Placeholder for fund distribution
    }
    
    modifier onlyOwner() {
        // Placeholder for access control
        _;
    }
}
```

## Security Considerations

### Price Manipulation Protection
- **Input Validation**: Validate all price-related input, especially `priceModifierFactor`.
- **Max Price Limits**: Enforce `maxPrice` to prevent runaway prices.
- **Trusted Price Oracles**: If external factors influence price, use secure oracles.

### Floating Point Emulation
- **Integer Math**: All calculations use integer arithmetic to avoid floating point inaccuracies.
- **Precision**: Ensure sufficient precision (e.g., using `1 ether` as base unit for wei).

### Access Control
- **Owner-Only Configuration**: Sensitive pricing parameters should be set by authorized entities.
- **Price Modification**: Only authorized functions should be able to trigger price updates.

## Gas Optimization

### Calculation Efficiency
- Optimized mathematical operations for price modifications.
- Minimal storage reads and writes during price updates.

### Storage Efficiency
- `VariablePriceContract` uses packed storage to minimize gas costs.
- Integrates with Diamond Storage for secure and efficient storage.

## Error Handling

### Common Errors
- `Vpl: Invalid initial price`: Start price is zero or negative.
- `Vpl: Quantity out of bounds`: Purchase quantity exceeds limits.
- `Vpl: Insufficient payment`: Sent value is less than current price.
- `Vpl: Max price exceeded`: Attempting to set price above `maxPrice`.

## Best Practices

### Pricing Strategy Design
- Choose the correct `PriceModifier` based on desired tokenomics (e.g., `Fixed` for flat increases, `InverseLog` for diminishing returns).
- Carefully select `priceModifierFactor` to achieve desired price curve.

### Integration Checklist
- Ensure pricing logic is clearly communicated to users on the frontend.
- Provide real-time price updates and projected future prices.
- Handle excess ETH refunds gracefully on purchases.

### Development Guidelines
- Write comprehensive unit tests for all pricing strategies and edge cases.
- Monitor price changes and ensure they align with expected curves.
- Conduct economic simulations to validate pricing models.

## Related Documentation

- [IVariablePrice Interface](../../smart-contracts/interfaces/ivariable-price.md) - Interface definition.
- [Multi Sale Facet](../../smart-contracts/facets/multi-sale-facet.md) - For multi-token sales integrated with variable pricing.
- [EIP-DRAFT-Multi-Token-Sale-Standard](../../eips/EIP-DRAFT-Multi-Token-Sale-Standard.md) - Contains dynamic pricing use cases.
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)