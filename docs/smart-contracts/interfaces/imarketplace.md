# IMarketplace Interface

## Overview

The `IMarketplace.sol` interface defines the core interface for NFT marketplace functionality within the Gemforce platform. This interface establishes the standard contract for marketplace operations including listing, purchasing, delisting, and fee management for NFT trading.

## Interface Details

- **Interface Name**: `IMarketplace`
- **License**: MIT
- **Solidity Version**: ^0.8.4

## Key Features

### 🔹 NFT Trading Operations
- List NFTs for sale with flexible pricing
- Purchase NFTs with multiple payment methods
- Delist items from the marketplace
- Transfer management for NFT custody

### 🔹 Fee Management System
- Configurable fee receivers and percentages
- Parts-per-million fee calculation
- Multiple fee recipient support
- Transparent fee distribution

### 🔹 Multi-Token Payment Support
- ETH payments for NFT purchases
- ERC20 token payment options
- Flexible payment token configuration
- Secure payment processing

### 🔹 Comprehensive Item Management
- Detailed item metadata tracking
- Seller and owner management
- Sale status tracking
- Item discovery and querying

## Core Data Structures

### FeeReceiver
```solidity
struct FeeReceiver {
    address payable receiver;    // Address that will receive the fee
    uint256 sharePerMillion;     // Fee share in parts per million
}
```

**Purpose**: Defines fee distribution configuration for marketplace transactions.

**Fields**:
- **receiver**: Address that will receive the fee payment
- **sharePerMillion**: Fee percentage in parts per million (e.g., 10,000 = 1%)

**Fee Calculation**:
- **1% fee**: `sharePerMillion = 10,000`
- **0.5% fee**: `sharePerMillion = 5,000`
- **2.5% fee**: `sharePerMillion = 25,000`
- **Maximum precision**: Up to 0.0001% (1 part per million)

**Usage Example**:
```solidity
// Configure marketplace fees
FeeReceiver[] memory feeReceivers = new FeeReceiver[](2);

// Platform fee: 2.5%
feeReceivers[0] = FeeReceiver({
    receiver: payable(platformAddress),
    sharePerMillion: 25000
});

// Creator royalty: 5%
feeReceivers[1] = FeeReceiver({
    receiver: payable(creatorAddress),
    sharePerMillion: 50000
});
```

---

### MarketItem
```solidity
struct MarketItem {
    address nftContract;    // NFT contract address
    uint256 tokenId;        // Token ID within the NFT contract
    address seller;         // Address of the seller
    address owner;          // Current owner of the NFT
    uint256 price;          // Sale price
    bool sold;              // Whether the item has been sold
    address receiver;       // Address to receive sale proceeds
    address paymentToken;   // Token contract for payment (address(0) for ETH)
}
```

**Purpose**: Comprehensive data structure representing an NFT listing in the marketplace.

**Fields**:
- **nftContract**: Address of the NFT contract containing the token
- **tokenId**: Unique identifier of the NFT within its contract
- **seller**: Address of the account that listed the item for sale
- **owner**: Current owner of the NFT (may differ from seller)
- **price**: Sale price in the specified payment token
- **sold**: Boolean indicating if the item has been purchased
- **receiver**: Address that will receive the sale proceeds
- **paymentToken**: Address of the ERC20 token for payment (zero address for ETH)

**Usage Example**:
```solidity
// Create marketplace listing
MarketItem memory item = MarketItem({
    nftContract: nftContractAddress,
    tokenId: 123,
    seller: msg.sender,
    owner: msg.sender,
    price: 1 ether,
    sold: false,
    receiver: msg.sender,
    paymentToken: address(0) // ETH payment
});
```

## Core Interface Functions

### Listing Functions

#### `listItem()`
```solidity
function listItem(
    address nftContract,
    address payable receiver,
    uint256 tokenId,
    uint256 price,
    bool transferNFT,
    address paymentToken
) external payable
```

**Purpose**: Lists an NFT for sale on the marketplace with specified parameters.

**Parameters**:
- `nftContract` (address): Address of the NFT contract
- `receiver` (address payable): Address to receive sale proceeds
- `tokenId` (uint256): ID of the token to list
- `price` (uint256): Sale price in the specified payment token
- `transferNFT` (bool): Whether to transfer NFT to marketplace custody
- `paymentToken` (address): Token for payment (zero address for ETH)

**Access Control**: Public (with ownership validation)

**Process**:
1. Validates caller owns the NFT or has approval
2. Optionally transfers NFT to marketplace custody
3. Creates marketplace listing with specified parameters
4. Emits Listings event with complete item details
5. Returns item ID for future reference

**Events**: `Listings(nftContract, tokenId, seller, receiver, owner, price, sold, paymentToken)`

**Example Usage**:
```solidity
// List NFT for sale with ETH payment
address nftContract = 0x1234567890123456789012345678901234567890;
uint256 tokenId = 123;
uint256 price = 1 ether;

// First approve marketplace to transfer NFT
IERC721(nftContract).approve(marketplaceAddress, tokenId);

// List the item
IMarketplace(marketplace).listItem(
    nftContract,
    payable(msg.sender),  // Receive proceeds
    tokenId,
    price,
    true,                 // Transfer NFT to marketplace
    address(0)            // ETH payment
);

console.log("NFT listed for sale at", price, "ETH");
```

**Payment Token Options**:
```solidity
// List with USDC payment
address usdcToken = 0xA0b86a33E6441e6e80A7181a02d0d25e8E687770;
uint256 usdcPrice = 1000 * 10**6; // 1000 USDC

IMarketplace(marketplace).listItem(
    nftContract,
    payable(msg.sender),
    tokenId,
    usdcPrice,
    true,
    usdcToken
);
```

---

#### `delistItem()`
```solidity
function delistItem(address nftContract, uint256 itemId) external
```

**Purpose**: Removes an NFT listing from the marketplace.

**Parameters**:
- `nftContract` (address): Address of the NFT contract
- `itemId` (uint256): ID of the marketplace item to delist

**Access Control**: Seller or authorized party only

**Process**:
1. Validates caller has permission to delist
2. Removes item from active marketplace listings
3. Returns NFT to seller if in marketplace custody
4. Updates item status to delisted
5. Emits Delisted event

**Events**: `Delisted(itemId)`

**Example Usage**:
```solidity
// Delist an NFT from the marketplace
uint256 itemId = 456;
address nftContract = 0x1234567890123456789012345678901234567890;

IMarketplace(marketplace).delistItem(nftContract, itemId);
console.log("Item", itemId, "delisted from marketplace");
```

### Purchase Functions

#### `purchaseItem()`
```solidity
function purchaseItem(address nftContract, uint256 itemId) external payable
```

**Purpose**: Purchases an NFT from the marketplace.

**Parameters**:
- `nftContract` (address): Address of the NFT contract
- `itemId` (uint256): ID of the marketplace item to purchase

**Access Control**: Public (with payment validation)

**Process**:
1. Validates item is available for purchase
2. Processes payment according to item's payment token
3. Calculates and distributes fees to configured receivers
4. Transfers remaining proceeds to seller/receiver
5. Transfers NFT to buyer
6. Updates item status to sold
7. Emits Sales event

**Events**: `Sales(tokenAddress, tokenId, owner)`

**Example Usage**:
```solidity
// Purchase NFT with ETH
uint256 itemId = 456;
address nftContract = 0x1234567890123456789012345678901234567890;

// Get item details first
MarketItem memory item = IMarketplace(marketplace).fetchItem(nftContract, itemId);
require(!item.sold, "Item already sold");

// Purchase with ETH
IMarketplace(marketplace).purchaseItem{value: item.price}(nftContract, itemId);
console.log("NFT purchased for", item.price, "ETH");
```

**ERC20 Token Purchase**:
```solidity
// Purchase NFT with ERC20 token
MarketItem memory item = IMarketplace(marketplace).fetchItem(nftContract, itemId);

if (item.paymentToken != address(0)) {
    // Approve token transfer first
    IERC20(item.paymentToken).approve(marketplaceAddress, item.price);
    
    // Purchase (no ETH value needed)
    IMarketplace(marketplace).purchaseItem(nftContract, itemId);
}
```

### Query Functions

#### `fetchItems()`
```solidity
function fetchItems() external view returns (MarketItem[] memory)
```

**Purpose**: Retrieves all active marketplace listings.

**Returns**:
- `MarketItem[]`: Array of all marketplace items

**Example Usage**:
```solidity
// Get all marketplace listings
MarketItem[] memory allItems = IMarketplace(marketplace).fetchItems();

console.log("Total marketplace items:", allItems.length);

for (uint256 i = 0; i < allItems.length; i++) {
    MarketItem memory item = allItems[i];
    if (!item.sold) {
        console.log("Available NFT:", item.nftContract, "Token ID:", item.tokenId, "Price:", item.price);
    }
}
```

---

#### `fetchItem()`
```solidity
function fetchItem(address nftContract, uint256 tokenId) external view returns (MarketItem memory)
```

**Purpose**: Retrieves details for a specific NFT listing.

**Parameters**:
- `nftContract` (address): Address of the NFT contract
- `tokenId` (uint256): ID of the token

**Returns**:
- `MarketItem`: Complete item details

**Example Usage**:
```solidity
// Get specific item details
address nftContract = 0x1234567890123456789012345678901234567890;
uint256 tokenId = 123;

MarketItem memory item = IMarketplace(marketplace).fetchItem(nftContract, tokenId);

if (item.nftContract != address(0)) {
    console.log("Item found:");
    console.log("- Seller:", item.seller);
    console.log("- Price:", item.price);
    console.log("- Sold:", item.sold);
    console.log("- Payment Token:", item.paymentToken);
} else {
    console.log("Item not found in marketplace");
}
```

---

#### `getMarketplaceFeeReceivers()`
```solidity
function getMarketplaceFeeReceivers() external view returns (FeeReceiver[] memory)
```

**Purpose**: Returns the current fee configuration for marketplace transactions.

**Returns**:
- `FeeReceiver[]`: Array of fee receiver configurations

**Example Usage**:
```solidity
// Check marketplace fee structure
FeeReceiver[] memory feeReceivers = IMarketplace(marketplace).getMarketplaceFeeReceivers();

console.log("Marketplace fee structure:");
uint256 totalFeePerMillion = 0;

for (uint256 i = 0; i < feeReceivers.length; i++) {
    FeeReceiver memory fee = feeReceivers[i];
    uint256 feePercentage = fee.sharePerMillion / 10000; // Convert to basis points
    
    console.log("- Receiver:", fee.receiver);
    console.log("- Fee:", feePercentage / 100, ".", feePercentage % 100, "%");
    
    totalFeePerMillion += fee.sharePerMillion;
}

console.log("Total marketplace fee:", totalFeePerMillion / 10000 / 100, ".", (totalFeePerMillion / 10000) % 100, "%");
```

## Integration Examples

### Complete Marketplace Integration
```solidity
// Comprehensive marketplace integration example
contract NFTMarketplaceIntegration {
    IMarketplace public marketplace;
    
    struct ListingInfo {
        uint256 itemId;
        uint256 listingTime;
        uint256 views;
        bool featured;
    }
    
    mapping(address => mapping(uint256 => ListingInfo)) public listingInfo;
    mapping(address => uint256[]) public userListings;
    
    event ItemListed(address indexed nftContract, uint256 indexed tokenId, address indexed seller, uint256 price);
    event ItemPurchased(address indexed nftContract, uint256 indexed tokenId, address indexed buyer, uint256 price);
    event ItemDelisted(address indexed nftContract, uint256 indexed tokenId, address indexed seller);
    
    constructor(address _marketplace) {
        marketplace = IMarketplace(_marketplace);
    }
    
    function listNFTForSale(
        address nftContract,
        uint256 tokenId,
        uint256 price,
        address paymentToken,
        bool featured
    ) external {
        // Verify ownership
        require(IERC721(nftContract).ownerOf(tokenId) == msg.sender, "Not token owner");
        
        // List on marketplace
        marketplace.listItem(
            nftContract,
            payable(msg.sender),
            tokenId,
            price,
            true, // Transfer to marketplace
            paymentToken
        );
        
        // Track listing info
        listingInfo[nftContract][tokenId] = ListingInfo({
            itemId: tokenId, // Simplified - would get actual item ID
            listingTime: block.timestamp,
            views: 0,
            featured: featured
        });
        
        userListings[msg.sender].push(tokenId);
        
        emit ItemListed(nftContract, tokenId, msg.sender, price);
    }
    
    function purchaseNFT(address nftContract, uint256 tokenId) external payable {
        // Get item details
        MarketItem memory item = marketplace.fetchItem(nftContract, tokenId);
        require(!item.sold, "Item already sold");
        require(item.nftContract != address(0), "Item not found");
        
        // Handle payment
        if (item.paymentToken == address(0)) {
            // ETH payment
            require(msg.value >= item.price, "Insufficient ETH");
            marketplace.purchaseItem{value: item.price}(nftContract, tokenId);
            
            // Refund excess ETH
            if (msg.value > item.price) {
                payable(msg.sender).transfer(msg.value - item.price);
            }
        } else {
            // ERC20 payment
            require(msg.value == 0, "Do not send ETH for token payments");
            
            // Transfer tokens to this contract first, then to marketplace
            IERC20(item.paymentToken).transferFrom(msg.sender, address(this), item.price);
            IERC20(item.paymentToken).approve(address(marketplace), item.price);
            
            marketplace.purchaseItem(nftContract, tokenId);
        }
        
        emit ItemPurchased(nftContract, tokenId, msg.sender, item.price);
    }
    
    function delistNFT(address nftContract, uint256 tokenId) external {
        // Verify seller
        MarketItem memory item = marketplace.fetchItem(nftContract, tokenId);
        require(item.seller == msg.sender, "Not the seller");
        require(!item.sold, "Item already sold");
        
        // Delist from marketplace
        marketplace.delistItem(nftContract, tokenId);
        
        // Clean up tracking
        delete listingInfo[nftContract][tokenId];
        _removeFromUserListings(msg.sender, tokenId);
        
        emit ItemDelisted(nftContract, tokenId, msg.sender);
    }
    
    function getMarketplaceOverview() external view returns (
        uint256 totalListings,
        uint256 totalSold,
        uint256 averagePrice,
        FeeReceiver[] memory feeStructure
    ) {
        MarketItem[] memory allItems = marketplace.fetchItems();
        
        totalListings = allItems.length;
        uint256 totalValue = 0;
        
        for (uint256 i = 0; i < allItems.length; i++) {
            if (allItems[i].sold) {
                totalSold++;
            }
            totalValue += allItems[i].price;
        }
        
        if (totalListings > 0) {
            averagePrice = totalValue / totalListings;
        }
        
        feeStructure = marketplace.getMarketplaceFeeReceivers();
    }
    
    function getUserListings(address user) external view returns (
        uint256[] memory tokenIds,
        MarketItem[] memory items
    ) {
        uint256[] memory userTokenIds = userListings[user];
        tokenIds = new uint256[](userTokenIds.length);
        items = new MarketItem[](userTokenIds.length);
        
        for (uint256 i = 0; i < userTokenIds.length; i++) {
            tokenIds[i] = userTokenIds[i];
            // Would need to track NFT contract addresses for complete implementation
        }
    }
    
    function _removeFromUserListings(address user, uint256 tokenId) internal {
        uint256[] storage listings = userListings[user];
        
        for (uint256 i = 0; i < listings.length; i++) {
            if (listings[i] == tokenId) {
                listings[i] = listings[listings.length - 1];
                listings.pop();
                break;
            }
        }
    }
}
```

### Marketplace Analytics System
```solidity
// Analytics and reporting for marketplace activity
contract MarketplaceAnalytics {
    IMarketplace public marketplace;
    
    struct SalesData {
        uint256 totalSales;
        uint256 totalVolume;
        uint256 averagePrice;
        uint256 uniqueBuyers;
        uint256 uniqueSellers;
    }
    
    struct CollectionStats {
        address nftContract;
        uint256 totalListings;
        uint256 totalSales;
        uint256 totalVolume;
        uint256 floorPrice;
        uint256 averagePrice;
    }
    
    mapping(address => SalesData) public collectionSales;
    mapping(address => mapping(uint256 => uint256)) public dailyVolume;
    mapping(address => bool) public trackedBuyers;
    mapping(address => bool) public trackedSellers;
    
    event SaleRecorded(address indexed nftContract, uint256 indexed tokenId, uint256 price, address buyer, address seller);
    event NewCollection(address indexed nftContract);
    
    function recordSale(
        address nftContract,
        uint256 tokenId,
        uint256 price,
        address buyer,
        address seller
    ) external onlyAuthorized {
        SalesData storage data = collectionSales[nftContract];
        
        // Update sales data
        data.totalSales++;
        data.totalVolume += price;
        data.averagePrice = data.totalVolume / data.totalSales;
        
        // Track unique participants
        if (!trackedBuyers[buyer]) {
            trackedBuyers[buyer] = true;
            data.uniqueBuyers++;
        }
        
        if (!trackedSellers[seller]) {
            trackedSellers[seller] = true;
            data.uniqueSellers++;
        }
        
        // Update daily volume
        uint256 today = block.timestamp / 1 days;
        dailyVolume[nftContract][today] += price;
        
        emit SaleRecorded(nftContract, tokenId, price, buyer, seller);
    }
    
    function getCollectionStats(address nftContract) external view returns (CollectionStats memory) {
        MarketItem[] memory allItems = marketplace.fetchItems();
        SalesData memory salesData = collectionSales[nftContract];
        
        uint256 totalListings = 0;
        uint256 floorPrice = type(uint256).max;
        uint256 totalListingValue = 0;
        
        // Analyze current listings
        for (uint256 i = 0; i < allItems.length; i++) {
            if (allItems[i].nftContract == nftContract && !allItems[i].sold) {
                totalListings++;
                totalListingValue += allItems[i].price;
                
                if (allItems[i].price < floorPrice) {
                    floorPrice = allItems[i].price;
                }
            }
        }
        
        if (floorPrice == type(uint256).max) {
            floorPrice = 0;
        }
        
        return CollectionStats({
            nftContract: nftContract,
            totalListings: totalListings,
            totalSales: salesData.totalSales,
            totalVolume: salesData.totalVolume,
            floorPrice: floorPrice,
            averagePrice: salesData.averagePrice
        });
    }
    
    function getDailyVolume(address nftContract, uint256 daysBack) external view returns (
        uint256[] memory dates,
        uint256[] memory volumes
    ) {
        dates = new uint256[](daysBack);
        volumes = new uint256[](daysBack);
        
        uint256 today = block.timestamp / 1 days;
        
        for (uint256 i = 0; i < daysBack; i++) {
            uint256 date = today - i;
            dates[i] = date;
            volumes[i] = dailyVolume[nftContract][date];
        }
    }
    
    function getTopCollections(uint256 limit) external view returns (address[] memory) {
        // Simplified implementation - would need more sophisticated sorting
        address[] memory collections = new address[](limit);
        // Implementation would sort by volume or other metrics
        return collections;
    }
}
```

### Fee Management System
```solidity
// Advanced fee management for marketplace
contract MarketplaceFeeManager {
    struct FeeConfig {
        FeeReceiver[] receivers;
        uint256 totalFeePerMillion;
        bool active;
        uint256 lastUpdated;
    }
    
    mapping(address => FeeConfig) public collectionFees;
    FeeConfig public defaultFees;
    
    event FeeConfigUpdated(address indexed collection, FeeReceiver[] receivers);
    event FeeDistributed(address indexed collection, uint256 totalAmount, address[] receivers, uint256[] amounts);
    
    function setDefaultFees(FeeReceiver[] memory receivers) external onlyOwner {
        _validateFeeReceivers(receivers);
        
        delete defaultFees.receivers;
        uint256 totalFee = 0;
        
        for (uint256 i = 0; i < receivers.length; i++) {
            defaultFees.receivers.push(receivers[i]);
            totalFee += receivers[i].sharePerMillion;
        }
        
        defaultFees.totalFeePerMillion = totalFee;
        defaultFees.active = true;
        defaultFees.lastUpdated = block.timestamp;
        
        emit FeeConfigUpdated(address(0), receivers);
    }
    
    function setCollectionFees(address collection, FeeReceiver[] memory receivers) external onlyOwner {
        _validateFeeReceivers(receivers);
        
        delete collectionFees[collection].receivers;
        uint256 totalFee = 0;
        
        for (uint256 i = 0; i < receivers.length; i++) {
            collectionFees[collection].receivers.push(receivers[i]);
            totalFee += receivers[i].sharePerMillion;
        }
        
        collectionFees[collection].totalFeePerMillion = totalFee;
        collectionFees[collection].active = true;
        collectionFees[collection].lastUpdated = block.timestamp;
        
        emit FeeConfigUpdated(collection, receivers);
    }
    
    function calculateFees(address collection, uint256 salePrice) external view returns (
        address[] memory receivers,
        uint256[] memory amounts,
        uint256 totalFees
    ) {
        FeeConfig memory config = collectionFees[collection].active 
            ? collectionFees[collection] 
            : defaultFees;
        
        receivers = new address[](config.receivers.length);
        amounts = new uint256[](config.receivers.length);
        
        for (uint256 i = 0; i < config.receivers.length; i++) {
            receivers[i] = config.receivers[i].receiver;
            amounts[i] = (salePrice * config.receivers[i].sharePerMillion) / 1000000;
            totalFees += amounts[i];
        }
    }
    
    function distributeFees(
        address collection,
        uint256 salePrice,
        address paymentToken
    ) external payable onlyAuthorized {
        (address[] memory receivers, uint256[] memory amounts, uint256 totalFees) = 
            this.calculateFees(collection, salePrice);
        
        if (paymentToken == address(0)) {
            // ETH distribution
            require(msg.value >= totalFees, "Insufficient ETH for fees");
            
            for (uint256 i = 0; i < receivers.length; i++) {
                payable(receivers[i]).transfer(amounts[i]);
            }
        } else {
            // ERC20 distribution
            for (uint256 i = 0; i < receivers.length; i++) {
                IERC20(paymentToken).transferFrom(msg.sender, receivers[i], amounts[i]);
            }
        }
        
        emit FeeDistributed(collection, totalFees, receivers, amounts);
    }
    
    function _validateFeeReceivers(FeeReceiver[] memory receivers) internal pure {
        require(receivers.length > 0, "No fee receivers provided");
        
        uint256 totalFee = 0;
        for (uint256 i = 0; i < receivers.length; i++) {
            require(receivers[i].receiver != address(0), "Invalid receiver address");
            require(receivers[i].sharePerMillion > 0, "Invalid fee share");
            totalFee += receivers[i].sharePerMillion;
        }
        
        require(totalFee <= 100000, "Total fees exceed 10%"); // Max 10% total fees
    }
}
```

## Events

### Listing Events
```solidity
event Listings(
    address indexed nftContract,
    uint256 indexed tokenId,
    address seller,
    address receiver,
    address owner,
    uint256 price,
    bool sold,
    address paymentToken
);

event Delisted(uint256 indexed itemId);
```

### Trading Events
```solidity
event Sales(address indexed tokenAddress, uint256 indexed tokenId, address indexed owner);
event Bids(uint256 indexed itemId, address bidder, uint256 amount);
```

## Security Considerations

### Access Control
- Ownership verification for listing and delisting
- Payment validation for purchases
- Fee receiver address validation
- Proper authorization for administrative functions

### Payment Security
- Secure ETH and ERC20 token handling
- Proper fee calculation and distribution
- Protection against reentrancy attacks
- Validation of payment amounts

### NFT Security
- Ownership verification before listing
- Secure NFT transfer mechanisms
- Protection against unauthorized transfers
- Proper custody management

## Gas Optimization

### Efficient Operations
- Batch operations where possible
- Optimized storage layout for MarketItem
- Minimal external calls
- Efficient fee calculations

### Query Optimization
- Efficient item retrieval patterns
- Optimized array operations
- Minimal storage reads
- Cached calculations where appropriate

## Testing Considerations

### Unit Tests
- Interface compliance verification
- Data structure validation
- Function behavior testing
- Event emission verification

### Integration Tests
- End-to-end marketplace workflows
- Payment processing scenarios
- Fee distribution testing
- Multi-token payment testing

## Related Documentation

- [MarketplaceFacet](../facets/marketplace-facet.md) - Marketplace implementation
- [FeeDistributorFacet](../facets/fee-distributor-facet.md) - Fee management
- [ERC721 Standard](https://eips.ethereum.org/EIPS/eip-721) - NFT standard
- [ERC20 Standard](https://eips.ethereum.org/EIPS/eip-20) - Token standard
- [Marketplace Guide](../../developer-guides/automated-testing-setup.md) - Implementation guide

---

*This interface defines the core contract for NFT marketplace functionality within the Gemforce platform, providing standardized trading operations with flexible payment and fee management.*