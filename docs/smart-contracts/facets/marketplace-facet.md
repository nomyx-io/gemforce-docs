# MarketplaceFacet

## Overview

The [`MarketplaceFacet.sol`](../../smart-contracts/facets/marketplace-facet.md) provides comprehensive NFT marketplace functionality for the Gemforce diamond system. This facet enables users to list, purchase, and manage NFT sales with support for both ETH and ERC20 token payments, featuring a sophisticated fee distribution system.

## Contract Details

- **Contract Name**: `MarketplaceFacet`
- **Inheritance**: `IMarketplace`, `Modifiers`, `ReentrancyGuard`
- **License**: MIT
- **Solidity Version**: ^0.8.4

## Key Features

### 🔹 Multi-Payment Support
- ETH payments for traditional transactions
- ERC20 token payments for flexible payment options
- Automatic payment validation and processing

### 🔹 Configurable Fee Distribution
- Parts-per-million precision fee system (1,000,000 = 100%)
- Multiple fee receivers per transaction
- Transparent fee distribution with events

### 🔹 Identity-Based Access Control
- Integration with Gemforce identity system
- Buyer registration requirements
- Permissioned NFT transfers

### 🔹 Security Features
- Reentrancy protection via `ReentrancyGuard`
- Checks-effects-interactions pattern
- Price protection mechanisms

## Core Data Structures

### FeeReceiver
```solidity
struct FeeReceiver {
    address receiver;        // Address receiving the fee
    uint256 sharePerMillion; // Fee share in parts per million
}
```

**Usage**: Defines fee distribution recipients and their percentage shares.

### MarketItem
```solidity
struct MarketItem {
    address nftContract;  // Contract address of the NFT
    uint256 tokenId;      // Token ID of the NFT
    address seller;       // Original seller address
    address owner;        // Current owner address
    uint256 price;        // Listing price
    bool sold;           // Sale status
    address receiver;     // Payment receiver address
    address paymentToken; // Payment token (address(0) for ETH)
}
```

**Usage**: Stores complete listing information for marketplace items.

## Core Functions

### Initialization

#### `initializeMarketplace()`
```solidity
function initializeMarketplace(FeeReceiver[] memory _feeReceivers) external onlyOwner
```

**Purpose**: One-time initialization of the marketplace fee distribution system.

**Parameters**:
- `_feeReceivers`: Array of fee receivers with their respective shares

**Access Control**: Owner only

**Validation**:
- Can only be called once
- Fee receiver addresses must be valid (non-zero)
- Share amounts must be greater than zero
- Total shares cannot exceed 100% (1,000,000 parts per million)

**Events**: Emits `MarketplaceInitialized(feeReceivers)`

### Listing Management

#### `listItem()`
```solidity
function listItem(
    address,                // Unused parameter for interface compatibility
    address payable receiver,
    uint256 tokenId,
    uint256 price,
    bool transferNFT,
    address paymentToken
) external payable nonReentrant
```

**Purpose**: Lists an NFT for sale in the marketplace.

**Parameters**:
- `receiver`: Address that will receive payment when item is sold
- `tokenId`: ID of the NFT being listed
- `price`: Listing price in wei (ETH) or token units
- `transferNFT`: Whether to transfer NFT to contract (true) or keep with seller (false)
- `paymentToken`: ERC20 token address for payment, or `address(0)` for ETH

**Access Control**: NFT owner only

**Validation**:
- Caller must own the NFT
- Price must be greater than zero
- Item must not already be listed
- No ETH should be sent with listing

**Process**:
1. Validates ownership and listing status
2. Optionally transfers NFT to contract
3. Creates market item entry
4. Updates listing status
5. Adds to items array

**Events**: Emits `Listings` event with listing details

#### `purchaseItem()`
```solidity
function purchaseItem(
    address,        // Unused parameter for interface compatibility
    uint256 tokenId,
    uint256 maxPrice
) external payable nonReentrant
```

**Purpose**: Purchases a listed NFT from the marketplace.

**Parameters**:
- `tokenId`: ID of the NFT to purchase
- `maxPrice`: Maximum price buyer is willing to pay (price protection)

**Access Control**: Registered users only

**Validation**:
- Item must be listed and not sold
- Buyer must be registered in identity system
- Sufficient payment must be provided
- Price must not exceed maxPrice

**Process**:
1. Validates listing status and buyer registration
2. Checks payment sufficiency
3. Marks item as sold
4. Transfers NFT to buyer
5. Distributes payment to fee receivers
6. Sends remaining payment to receiver

**Events**: Emits `Sales` event upon successful purchase

### Utility Functions

#### `delistItem()`
```solidity
function delistItem(uint256 tokenId) external nonReentrant
```

**Purpose**: Removes an NFT listing from the marketplace.

**Parameters**:
- `tokenId`: ID of the NFT to delist

**Access Control**: NFT owner only

**Process**:
1. Validates ownership
2. Transfers NFT back to owner (if held by contract)
3. Updates listing status
4. Clears market item data

**Events**: Emits `Delisted` event

## Payment Distribution System

### Fee Calculation
```solidity
uint256 private constant SHARE_PRECISION = 1_000_000; // 100% = 1,000,000
```

**Example Fee Structure**:
- Platform fee: 25,000 parts per million (2.5%)
- Community treasury: 10,000 parts per million (1.0%)
- Creator royalty: 50,000 parts per million (5.0%)
- **Total fees**: 85,000 parts per million (8.5%)
- **Seller receives**: 915,000 parts per million (91.5%)

### Distribution Process
1. **Calculate total fees**: Sum all fee receiver shares
2. **Distribute to fee receivers**: Transfer calculated amounts
3. **Transfer remainder**: Send remaining payment to listing receiver
4. **Emit events**: Log all payment distributions

## Security Considerations

### Reentrancy Protection
- All state-changing functions use `nonReentrant` modifier
- Follows checks-effects-interactions pattern
- State updates before external calls

### Access Control
- Owner-only initialization
- NFT owner validation for listings
- Identity system integration for purchases

### Payment Security
- Price validation and protection
- Overflow protection in fee calculations
- Safe token transfers using OpenZeppelin's SafeERC20

### Input Validation
- Non-zero price requirements
- Valid address checks
- Listing status validation

## Gas Optimization

### Storage Efficiency
- Packed structs for market items
- Efficient mapping usage
- Minimal storage operations

### Function Optimization
- Early validation returns
- Batch operations where possible
- Optimized fee distribution loops

## Integration Examples

### Initialize Marketplace
```solidity
// Set up fee distribution
IMarketplace.FeeReceiver[] memory feeReceivers = new IMarketplace.FeeReceiver[](2);

feeReceivers[0] = IMarketplace.FeeReceiver({
    receiver: platformTreasury,
    sharePerMillion: 25000 // 2.5%
});

feeReceivers[1] = IMarketplace.FeeReceiver({
    receiver: communityTreasury,
    sharePerMillion: 10000 // 1.0%
});

IMarketplace(diamond).initializeMarketplace(feeReceivers);
```

### List NFT for Sale
```solidity
// List NFT for 1 ETH, keeping NFT with seller
IMarketplace(diamond).listItem(
    address(0),           // Unused parameter
    payable(seller),      // Payment receiver
    tokenId,              // NFT token ID
    1 ether,              // Price in wei
    false,                // Don't transfer NFT to contract
    address(0)            // ETH payment
);
```

### Purchase NFT
```solidity
// Purchase NFT with ETH
IMarketplace(diamond).purchaseItem{value: 1 ether}(
    address(0),           // Unused parameter
    tokenId,              // NFT token ID
    1.1 ether            // Maximum price willing to pay
);
```

### Purchase with ERC20 Token
```solidity
// Approve token spending first
IERC20(paymentToken).approve(diamond, price);

// Purchase with ERC20 token
IMarketplace(diamond).purchaseItem(
    address(0),           // Unused parameter
    tokenId,              // NFT token ID
    price                 // Maximum price
);
```

## Events

### MarketplaceInitialized
```solidity
event MarketplaceInitialized(FeeReceiver[] feeReceivers);
```
Emitted when marketplace is initialized with fee receivers.

### PaymentDistributed
```solidity
event PaymentDistributed(address token, address receiver, uint256 amount);
```
Emitted for each payment distribution during a sale.

### Listings
```solidity
event Listings(
    address indexed nftContract,
    uint256 indexed tokenId,
    address indexed seller,
    address receiver,
    address buyer,
    uint256 price,
    bool sold,
    address paymentToken
);
```
Emitted when an NFT is listed for sale.

### Sales
```solidity
event Sales(
    address indexed nftContract,
    uint256 indexed tokenId,
    address indexed buyer
);
```
Emitted when an NFT is successfully purchased.

### Delisted
```solidity
event Delisted(uint256 indexed tokenId);
```
Emitted when an NFT listing is removed.

## Error Handling

### Common Errors
- `"Marketplace already initialized"`: Attempting to initialize twice
- `"MARKETPLACE: NOT_OWNER"`: Non-owner trying to list NFT
- `"MARKETPLACE: SALE_COMPLETED"`: Attempting to purchase already sold item
- `"MARKETPLACE: INVALID_LISTING"`: Trying to purchase non-existent listing
- `"Price must be greater than 0"`: Invalid pricing
- `"Total shares exceed 100%"`: Invalid fee configuration

## Testing Considerations

### Unit Tests
- Fee calculation accuracy
- Payment distribution logic
- Access control validation
- State transition correctness
- Batch operation efficiency

### Integration Tests
- End-to-end marketplace flows
- Multi-token payment scenarios
- Identity system integration
- Error condition handling

## Related Documentation

- [IMarketplace Interface](../interfaces/imarketplace.md) - Marketplace interface
- [Identity Registry Facet](identity-registry-facet.md) - Identity system integration
- [Diamond](../diamond.md) - Core diamond contract
- [EIP-DRAFT-Diamond-Enhanced-Marketplace](../../eips/EIP-DRAFT-Diamond-Enhanced-Marketplace.md) - EIP specification

---

*This facet implements the Diamond-Enhanced NFT Marketplace standard as defined in the Gemforce EIP suite.*