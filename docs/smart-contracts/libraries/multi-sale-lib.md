# MultiSaleLib Library

## Overview

The [`MultiSaleLib`](../../../contracts/libraries/MultiSaleLib.sol) library provides core utilities and data structures for managing multi-token sales within the Gemforce platform. This library implements comprehensive token sale functionality including whitelist management, Merkle proof validation, variable pricing, and support for multiple payment methods (ETH and ERC20 tokens).

## Key Features

- **Multi-Token Support**: Support for ERC20, ERC721, and ERC1155 token sales
- **Whitelist Management**: Merkle tree-based whitelist with proof validation
- **Variable Pricing**: Dynamic pricing with configurable price structures
- **Payment Flexibility**: Support for ETH and ERC20 token payments
- **Quantity Controls**: Min/max quantity limits per sale and per account
- **Time-Based Sales**: Configurable start and end times for sales
- **Proof Validation**: Secure Merkle proof validation to prevent replay attacks

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library MultiSaleLib {
    using SafeERC20 for IERC20;
    
    // Core functions
    function _createTokenSale() internal returns (uint256 tokenSaleId);
    function _validatePurchase(MultiSaleContract storage self, VariablePriceContract storage priceContract, uint256 quantity, uint256 valueAttached) internal view;
    function _validateProof(MultiSaleContract storage self, MultiSalePurchase memory purchase, MultiSaleProof memory purchaseProof) internal;
    function _purchaseToken(...) internal;
    function _airdropRedeemed(MultiSaleContract storage self, address recipient) internal view returns (bool isRedeemed);
}
```

## Data Structures

### PaymentType Enum
```solidity
enum PaymentType {
    Ether,    // Payment with native ETH
    ERC20     // Payment with ERC20 tokens
}
```

### PaymentMethod Enum
```solidity
enum PaymentMethod {
    Native,   // Payment with the native currency (e.g., ETH)
    ERC20     // Payment with an ERC20 token
}
```

### MultiSalePurchase Struct
```solidity
struct MultiSalePurchase {
    uint256 multiSaleId;    // ID of the multi-sale
    address purchaser;      // Address making the purchase
    address receiver;       // Address receiving the tokens
    uint256 quantity;       // Quantity of tokens to purchase
}
```

**Purpose**: Represents a token purchase request with all necessary details.

### MultiSaleProof Struct
```solidity
struct MultiSaleProof {
    uint256 leaf;           // Leaf value for Merkle proof
    uint256 total;          // Total allocation for the address
    bytes32[] merkleProof;  // Merkle proof array
    bytes data;             // Additional proof data
}
```

**Purpose**: Contains Merkle proof data for whitelist validation.

### MultiSaleSettings Struct
```solidity
struct MultiSaleSettings {
    TokenType tokenType;                    // Type of token being sold
    address token;                          // Token contract address
    uint256 tokenHash;                      // Token hash (0 for auto-creation)
    
    uint256 whitelistHash;                  // Merkle root for whitelist
    bool whitelistOnly;                     // Whether sale is whitelist-only
    
    PaymentMethod paymentMethod;            // Payment method (Native/ERC20)
    address paymentToken;                   // ERC20 token address for payments
    
    address owner;                          // Owner of the sale
    address payee;                          // Recipient of sale proceeds
    
    string symbol;                          // Token symbol
    string name;                            // Token name
    string description;                     // Token description
    
    bool openState;                         // Whether sale is open
    uint256 startTime;                      // Sale start timestamp
    uint256 endTime;                        // Sale end timestamp
    
    uint256 maxQuantity;                    // Maximum total tokens for sale
    uint256 maxQuantityPerSale;             // Maximum tokens per transaction
    uint256 minQuantityPerSale;             // Minimum tokens per transaction
    uint256 maxQuantityPerAccount;          // Maximum tokens per account
    
    PaymentType paymentType;                // Legacy payment type
    address tokenAddress;                   // Legacy token address
    
    uint256 nextSaleId;                     // Next sale ID counter
    VariablePriceContract price;            // Variable pricing configuration
}
```

**Purpose**: Comprehensive configuration for a multi-token sale.

### MultiSaleContract Struct
```solidity
struct MultiSaleContract {
    MultiSaleSettings settings;             // Sale configuration
    uint256 nonce;                          // Nonce for unique operations
    uint256 totalPurchased;                 // Total tokens purchased
    
    mapping(address => uint256) purchased;                      // Tokens purchased per address
    mapping(uint256 => uint256) _redeemedData;                 // Redeemed data tracking
    mapping(address => uint256) _redeemedDataQuantities;       // Redeemed quantities per address
    mapping(address => uint256) _totalDataQuantities;          // Total allocations per address
    mapping(address => uint256) _accountQuantities;            // Account quantity tracking
}
```

**Purpose**: Complete state management for a multi-token sale.

### MultiSaleStorage Struct
```solidity
struct MultiSaleStorage {
    uint256 tsnonce;                                    // Token sale nonce
    mapping(uint256 => MultiSaleContract) _tokenSales;  // All token sales
    uint256[] _tokenSaleIds;                            // Array of sale IDs
}
```

**Purpose**: Diamond storage structure for all multi-sale data.

## Core Functions

### Sale Creation

#### `_createTokenSale()`
```solidity
function _createTokenSale() internal returns (uint256 tokenSaleId)
```

**Purpose**: Generate a unique token sale ID for a new sale.

**Returns**: Unique token sale identifier

**Implementation**:
- Uses keccak256 hash of nonce and contract address
- Increments global nonce to ensure uniqueness
- Returns deterministic but unique sale ID

**Example Usage**:
```solidity
// Create a new token sale
uint256 saleId = MultiSaleLib._createTokenSale();
console.log("Created token sale with ID:", saleId);
```

### Purchase Validation

#### `_validatePurchase()`
```solidity
function _validatePurchase(
    MultiSaleContract storage self, 
    VariablePriceContract storage priceContract,
    uint256 quantity, 
    uint256 valueAttached
) internal view
```

**Purpose**: Validate a token purchase against sale parameters.

**Parameters**:
- `self`: Storage reference to the multi-sale contract
- `priceContract`: Variable price contract for pricing
- `quantity`: Number of tokens to purchase
- `valueAttached`: ETH value sent with transaction

**Validation Checks**:
- Sale not sold out (total quantity limit)
- Quantity within min/max per sale limits
- Sale has started (if start time set)
- Sale has not ended (if end time set)
- Sufficient payment value attached

**Example Usage**:
```solidity
// Validate a purchase before processing
MultiSaleLib._validatePurchase(
    saleContract,
    priceContract,
    5, // quantity
    msg.value
);
```

#### `_validateProof()`
```solidity
function _validateProof(
    MultiSaleContract storage self,
    MultiSalePurchase memory purchase,
    MultiSaleProof memory purchaseProof
) internal
```

**Purpose**: Validate Merkle proof for whitelist-only sales.

**Parameters**:
- `self`: Storage reference to the multi-sale contract
- `purchase`: Purchase details
- `purchaseProof`: Merkle proof data

**Validation Process**:
1. Check if sale is whitelist-only
2. Verify user hasn't already redeemed allocation
3. Construct leaf data with receiver and total allocation
4. Verify Merkle proof against whitelist root
5. Update redeemed quantities
6. Prevent replay attacks by tracking used leaves

**Security Features**:
- Prevents double-spending of whitelist allocations
- Protects against replay attacks
- Validates total allocation limits

**Example Usage**:
```solidity
// Validate whitelist proof
MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
    multiSaleId: saleId,
    purchaser: msg.sender,
    receiver: msg.sender,
    quantity: 3
});

MultiSaleLib.MultiSaleProof memory proof = MultiSaleLib.MultiSaleProof({
    leaf: leafValue,
    total: 10, // Total allocation
    merkleProof: merkleProofArray,
    data: ""
});

MultiSaleLib._validateProof(saleContract, purchase, proof);
```

### Purchase Processing

#### `_purchaseToken()` (with proof)
```solidity
function _purchaseToken(
    MultiSaleContract storage self,
    VariablePriceContract storage variablePrice,
    MultiSalePurchase memory purchase,
    MultiSaleProof memory purchaseProof,
    uint256 valueAttached
) internal
```

**Purpose**: Process a token purchase with whitelist proof validation.

**Process Flow**:
1. Validate purchase parameters
2. Validate Merkle proof (if whitelist-only)
3. Process payment and token transfer

#### `_purchaseToken()` (without proof)
```solidity
function _purchaseToken(
    MultiSaleContract storage self,
    VariablePriceContract storage variablePrice,
    MultiSalePurchase memory purchase,
    uint256 valueAttached
) internal
```

**Purpose**: Process a token purchase without proof (public sale).

**Process Flow**:
1. Validate purchase parameters
2. Process payment and token transfer

### Utility Functions

#### `_airdropRedeemed()`
```solidity
function _airdropRedeemed(MultiSaleContract storage self, address recipient) internal view returns (bool isRedeemed)
```

**Purpose**: Check if an address has fully redeemed their whitelist allocation.

**Parameters**:
- `self`: Storage reference to the multi-sale contract
- `recipient`: Address to check

**Returns**: Boolean indicating if allocation is fully redeemed

**Logic**:
- Compares total allocation with redeemed amount
- Returns true if fully redeemed, false otherwise

## Integration Examples

### NFT Collection Sale
```solidity
// NFT collection sale with whitelist and public phases
contract NFTCollectionSale {
    using MultiSaleLib for MultiSaleLib.MultiSaleStorage;
    
    struct SalePhase {
        uint256 saleId;
        string name;
        bool isWhitelistPhase;
        uint256 price;
        uint256 maxPerWallet;
        uint256 startTime;
        uint256 endTime;
    }
    
    mapping(uint256 => SalePhase) public salePhases;
    uint256 public currentPhaseId;
    
    event PhaseCreated(uint256 indexed phaseId, string name, bool isWhitelist);
    event TokensPurchased(uint256 indexed phaseId, address indexed buyer, uint256 quantity);
    
    function createWhitelistPhase(
        string memory name,
        uint256 price,
        uint256 maxPerWallet,
        uint256 startTime,
        uint256 endTime,
        bytes32 merkleRoot
    ) external onlyOwner returns (uint256 phaseId) {
        phaseId = MultiSaleLib._createTokenSale();
        
        // Configure whitelist phase
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(phaseId);
        sale.settings.name = name;
        sale.settings.whitelistOnly = true;
        sale.settings.whitelistHash = uint256(merkleRoot);
        sale.settings.maxQuantityPerAccount = maxPerWallet;
        sale.settings.startTime = startTime;
        sale.settings.endTime = endTime;
        sale.settings.paymentMethod = MultiSaleLib.PaymentMethod.Native;
        
        // Set pricing
        sale.settings.price.price = price;
        
        salePhases[phaseId] = SalePhase({
            saleId: phaseId,
            name: name,
            isWhitelistPhase: true,
            price: price,
            maxPerWallet: maxPerWallet,
            startTime: startTime,
            endTime: endTime
        });
        
        currentPhaseId = phaseId;
        emit PhaseCreated(phaseId, name, true);
    }
    
    function createPublicPhase(
        string memory name,
        uint256 price,
        uint256 maxPerWallet,
        uint256 startTime,
        uint256 endTime
    ) external onlyOwner returns (uint256 phaseId) {
        phaseId = MultiSaleLib._createTokenSale();
        
        // Configure public phase
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(phaseId);
        sale.settings.name = name;
        sale.settings.whitelistOnly = false;
        sale.settings.maxQuantityPerAccount = maxPerWallet;
        sale.settings.startTime = startTime;
        sale.settings.endTime = endTime;
        sale.settings.paymentMethod = MultiSaleLib.PaymentMethod.Native;
        
        // Set pricing
        sale.settings.price.price = price;
        
        salePhases[phaseId] = SalePhase({
            saleId: phaseId,
            name: name,
            isWhitelistPhase: false,
            price: price,
            maxPerWallet: maxPerWallet,
            startTime: startTime,
            endTime: endTime
        });
        
        emit PhaseCreated(phaseId, name, false);
    }
    
    function purchaseWhitelist(
        uint256 phaseId,
        uint256 quantity,
        MultiSaleLib.MultiSaleProof memory proof
    ) external payable {
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(phaseId);
        require(sale.settings.whitelistOnly, "Not a whitelist phase");
        
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: phaseId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: quantity
        });
        
        MultiSaleLib._purchaseToken(
            sale,
            sale.settings.price,
            purchase,
            proof,
            msg.value
        );
        
        // Mint NFTs to buyer
        _mintTokens(msg.sender, quantity);
        
        emit TokensPurchased(phaseId, msg.sender, quantity);
    }
    
    function purchasePublic(
        uint256 phaseId,
        uint256 quantity
    ) external payable {
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(phaseId);
        require(!sale.settings.whitelistOnly, "Whitelist phase only");
        
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: phaseId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: quantity
        });
        
        MultiSaleLib._purchaseToken(
            sale,
            sale.settings.price,
            purchase,
            msg.value
        );
        
        // Mint NFTs to buyer
        _mintTokens(msg.sender, quantity);
        
        emit TokensPurchased(phaseId, msg.sender, quantity);
    }
    
    function getSaleContract(uint256 saleId) internal view returns (MultiSaleLib.MultiSaleContract storage) {
        // Implementation would access Diamond storage
    }
    
    function _mintTokens(address to, uint256 quantity) internal {
        // Implementation would mint NFTs
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

### Gaming Item Sale
```solidity
// Gaming item sale with ERC20 payments and tiered pricing
contract GamingItemSale {
    using MultiSaleLib for MultiSaleLib.MultiSaleStorage;
    
    struct ItemTier {
        string name;
        uint256 basePrice;
        uint256 maxSupply;
        uint256 sold;
        bool active;
    }
    
    mapping(uint256 => ItemTier) public itemTiers;
    mapping(uint256 => uint256) public saleToTier;
    IERC20 public gameToken;
    
    event ItemTierCreated(uint256 indexed tierId, string name, uint256 basePrice, uint256 maxSupply);
    event ItemsPurchased(uint256 indexed tierId, address indexed buyer, uint256 quantity, uint256 totalCost);
    
    constructor(address _gameToken) {
        gameToken = IERC20(_gameToken);
    }
    
    function createItemTier(
        string memory name,
        uint256 basePrice,
        uint256 maxSupply,
        uint256 maxPerPurchase
    ) external onlyGameMaster returns (uint256 tierId) {
        tierId = MultiSaleLib._createTokenSale();
        
        // Configure item tier sale
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(tierId);
        sale.settings.name = name;
        sale.settings.paymentMethod = MultiSaleLib.PaymentMethod.ERC20;
        sale.settings.paymentToken = address(gameToken);
        sale.settings.maxQuantity = maxSupply;
        sale.settings.maxQuantityPerSale = maxPerPurchase;
        sale.settings.openState = true;
        
        // Set pricing
        sale.settings.price.price = basePrice;
        
        itemTiers[tierId] = ItemTier({
            name: name,
            basePrice: basePrice,
            maxSupply: maxSupply,
            sold: 0,
            active: true
        });
        
        saleToTier[tierId] = tierId;
        
        emit ItemTierCreated(tierId, name, basePrice, maxSupply);
    }
    
    function purchaseItems(
        uint256 tierId,
        uint256 quantity
    ) external {
        ItemTier storage tier = itemTiers[tierId];
        require(tier.active, "Tier not active");
        require(tier.sold + quantity <= tier.maxSupply, "Exceeds max supply");
        
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(tierId);
        
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: tierId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: quantity
        });
        
        // Calculate total cost
        uint256 totalCost = tier.basePrice * quantity;
        
        // Transfer payment tokens
        gameToken.safeTransferFrom(msg.sender, address(this), totalCost);
        
        // Process purchase
        MultiSaleLib._purchaseToken(
            sale,
            sale.settings.price,
            purchase,
            0 // No ETH value for ERC20 payments
        );
        
        // Update tier tracking
        tier.sold += quantity;
        
        // Mint game items
        _mintGameItems(msg.sender, tierId, quantity);
        
        emit ItemsPurchased(tierId, msg.sender, quantity, totalCost);
    }
    
    function getTierInfo(uint256 tierId) external view returns (
        string memory name,
        uint256 basePrice,
        uint256 maxSupply,
        uint256 sold,
        uint256 remaining,
        bool active
    ) {
        ItemTier memory tier = itemTiers[tierId];
        name = tier.name;
        basePrice = tier.basePrice;
        maxSupply = tier.maxSupply;
        sold = tier.sold;
        remaining = tier.maxSupply - tier.sold;
        active = tier.active;
    }
    
    function getSaleContract(uint256 saleId) internal view returns (MultiSaleLib.MultiSaleContract storage) {
        // Implementation would access Diamond storage
    }
    
    function _mintGameItems(address to, uint256 tierId, uint256 quantity) internal {
        // Implementation would mint game items
    }
    
    modifier onlyGameMaster() {
        // Implementation would check game master role
        _;
    }
}
```

### Airdrop Distribution System
```solidity
// Airdrop distribution using Merkle proofs
contract AirdropDistribution {
    using MultiSaleLib for MultiSaleLib.MultiSaleStorage;
    
    struct AirdropCampaign {
        uint256 saleId;
        string name;
        bytes32 merkleRoot;
        uint256 totalTokens;
        uint256 claimedTokens;
        uint256 startTime;
        uint256 endTime;
        bool active;
    }
    
    mapping(uint256 => AirdropCampaign) public campaigns;
    mapping(uint256 => mapping(address => bool)) public hasClaimed;
    uint256 public nextCampaignId;
    
    event CampaignCreated(uint256 indexed campaignId, string name, bytes32 merkleRoot);
    event TokensClaimed(uint256 indexed campaignId, address indexed claimer, uint256 amount);
    
    function createAirdropCampaign(
        string memory name,
        bytes32 merkleRoot,
        uint256 totalTokens,
        uint256 startTime,
        uint256 endTime
    ) external onlyOwner returns (uint256 campaignId) {
        campaignId = nextCampaignId++;
        uint256 saleId = MultiSaleLib._createTokenSale();
        
        // Configure airdrop as a zero-price sale
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(saleId);
        sale.settings.name = name;
        sale.settings.whitelistOnly = true;
        sale.settings.whitelistHash = uint256(merkleRoot);
        sale.settings.maxQuantity = totalTokens;
        sale.settings.startTime = startTime;
        sale.settings.endTime = endTime;
        sale.settings.openState = true;
        
        // Set price to zero for airdrop
        sale.settings.price.price = 0;
        
        campaigns[campaignId] = AirdropCampaign({
            saleId: saleId,
            name: name,
            merkleRoot: merkleRoot,
            totalTokens: totalTokens,
            claimedTokens: 0,
            startTime: startTime,
            endTime: endTime,
            active: true
        });
        
        emit CampaignCreated(campaignId, name, merkleRoot);
    }
    
    function claimAirdrop(
        uint256 campaignId,
        uint256 amount,
        MultiSaleLib.MultiSaleProof memory proof
    ) external {
        AirdropCampaign storage campaign = campaigns[campaignId];
        require(campaign.active, "Campaign not active");
        require(block.timestamp >= campaign.startTime, "Campaign not started");
        require(block.timestamp <= campaign.endTime, "Campaign ended");
        require(!hasClaimed[campaignId][msg.sender], "Already claimed");
        
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(campaign.saleId);
        
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: campaign.saleId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: amount
        });
        
        // Validate proof and process claim
        MultiSaleLib._validateProof(sale, purchase, proof);
        
        // Update tracking
        campaign.claimedTokens += amount;
        hasClaimed[campaignId][msg.sender] = true;
        
        // Transfer tokens
        _transferAirdropTokens(msg.sender, amount);
        
        emit TokensClaimed(campaignId, msg.sender, amount);
    }
    
    function getCampaignStatus(uint256 campaignId) external view returns (
        string memory name,
        uint256 totalTokens,
        uint256 claimedTokens,
        uint256 remainingTokens,
        bool active,
        bool hasStarted,
        bool hasEnded
    ) {
        AirdropCampaign memory campaign = campaigns[campaignId];
        name = campaign.name;
        totalTokens = campaign.totalTokens;
        claimedTokens = campaign.claimedTokens;
        remainingTokens = campaign.totalTokens - campaign.claimedTokens;
        active = campaign.active;
        hasStarted = block.timestamp >= campaign.startTime;
        hasEnded = block.timestamp > campaign.endTime;
    }
    
    function getUserClaimStatus(
        uint256 campaignId,
        address user
    ) external view returns (bool claimed, uint256 remainingAllocation) {
        claimed = hasClaimed[campaignId][user];
        
        if (!claimed) {
            MultiSaleLib.MultiSaleContract storage sale = getSaleContract(campaigns[campaignId].saleId);
            uint256 totalAllocation = sale._totalDataQuantities[user];
            uint256 redeemedAmount = sale._redeemedDataQuantities[user];
            remainingAllocation = totalAllocation - redeemedAmount;
        }
    }
    
    function getSaleContract(uint256 saleId) internal view returns (MultiSaleLib.MultiSaleContract storage) {
        // Implementation would access Diamond storage
    }
    
    function _transferAirdropTokens(address to, uint256 amount) internal {
        // Implementation would transfer tokens
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

## Events

### Multi-Sale Events
```solidity
event MultiSaleCreated(uint256 indexed tokenSaleId, MultiSaleSettings settings);
event MultiSaleOpen(uint256 indexed tokenSaleId, MultiSaleSettings tokenSale);
event MultiSaleClosed(uint256 indexed tokenSaleId);
event MultiSaleSold(uint256 indexed tokenSaleId, address indexed purchaser, uint256[] tokenIds, bytes data);
event MultiSaleUpdated(uint256 indexed tokenSaleId, MultiSaleSettings tokenSale);
```

## Security Considerations

### Merkle Proof Security
- Prevents replay attacks by tracking used leaves
- Validates total allocation limits per address
- Secure leaf construction with address and allocation data
- Protection against proof manipulation

### Payment Security
- Safe ERC20 token transfers using SafeERC20
- Validation of payment amounts before processing
- Support for both ETH and ERC20 payments
- Overflow protection in price calculations

### Access Control
- Whitelist validation through Merkle proofs
- Quantity limits per sale and per account
- Time-based sale controls
- Owner-only administrative functions

## Gas Optimization

### Storage Efficiency
- Efficient storage layout for sale data
- Minimal storage writes during purchases
- Optimized mapping structures for tracking
- Batch operations where possible

### Function Efficiency
- Internal functions for gas savings
- Efficient validation logic
- Minimal external calls
- Optimized Merkle proof verification

## Error Handling

### Common Errors
- "soldout" - Sale has reached maximum quantity
- "qtytoolow" / "qtytoohigh" - Quantity outside allowed range
- "notstarted" / "saleended" - Sale timing violations
- "notenoughvalue" - Insufficient payment
- "redeemed" - Allocation already claimed
- "Merkle proof failed" - Invalid whitelist proof

### Best Practices
- Validate all purchase parameters before processing
- Check Merkle proofs for whitelist sales
- Verify payment amounts and token transfers
- Handle edge cases gracefully
- Provide clear error messages for debugging

## Testing Considerations

### Unit Tests
- Sale creation and configuration
- Purchase validation logic
- Merkle proof verification
- Payment processing (ETH and ERC20)
- Quantity and timing controls

### Integration Tests
- Multi-phase sale workflows
- Whitelist and public sale combinations
- Cross-contract token transfers
- Airdrop distribution scenarios
- Gaming item sale mechanics

## Related Documentation

- [IMultiSale Interface](../interfaces/imulti-sale.md) - Multi-sale interface definition
- [MultiSaleFacet](../facets/multi-sale-facet.md) - Multi-sale facet implementation
- [VariablePriceLib](variable-price-lib.md) - Variable pricing utilities
- [MerkleProver](merkle-prover.md) - Merkle proof verification utilities
- [Token Sale Guide](../../guides/token-sales.md) - Implementation guide for token sales

---

*This library provides comprehensive utilities for multi-token sales within the Gemforce platform, supporting flexible payment methods, whitelist management, and secure proof validation for various sale scenarios including NFT drops, gaming items, and airdrop distributions.*