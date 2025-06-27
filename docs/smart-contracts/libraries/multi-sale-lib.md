# MultiSaleLib Library

## Overview

The [`MultiSaleLib`](../../smart-contracts/libraries/multi-sale-lib.md) library provides core utilities and data structures for managing multi-token sales within the Gemforce platform. This library implements comprehensive token sale functionality including whitelist management, Merkle proof validation, variable pricing, and support for multiple payment methods (ETH and ERC20 tokens).

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
        uint256 maxSupply
    ) external onlyOwner returns (uint256 tierId) {
        tierId = MultiSaleLib._createTokenSale();
        
        itemTiers[tierId] = ItemTier({
            name: name,
            basePrice: basePrice,
            maxSupply: maxSupply,
            sold: 0,
            active: true
        });
        
        // Configure common sale settings for this tier
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(tierId);
        sale.settings.name = name;
        sale.settings.paymentMethod = MultiSaleLib.PaymentMethod.ERC20;
        sale.settings.paymentToken = address(gameToken);
        sale.settings.price.price = basePrice;
        sale.settings.maxQuantity = maxSupply;
        sale.settings.maxQuantityPerAccount = 100; // Example limit
        
        emit ItemTierCreated(tierId, name, basePrice, maxSupply);
    }
    
    function purchaseItems(uint256 tierId, uint256 quantity) external {
        ItemTier storage tier = itemTiers[tierId];
        require(tier.active, "Item tier not active");
        require(quantity > 0, "Quantity must be positive");
        require(tier.sold + quantity <= tier.maxSupply, "Not enough items left");
        
        // Get sale contract for this tier
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(tierId);
        
        // Validate purchase
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: tierId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: quantity
        });
        
        MultiSaleLib._purchaseToken(sale, sale.settings.price, purchase, 0); // No ETH attached
        
        // Transfer ERC20 payment
        uint256 totalCost = sale.settings.price.price * quantity;
        gameToken.safeTransferFrom(msg.sender, address(this), totalCost);
        
        // Update sold count
        tier.sold += quantity;
        
        // Mint the actual game items (assuming separate minting logic)
        _mintGameItems(msg.sender, tierId, quantity);
        
        emit ItemsPurchased(tierId, msg.sender, quantity, totalCost);
    }
    
    function getSaleContract(uint256 saleId) internal view returns (MultiSaleLib.MultiSaleContract storage) {
        // Implementation would access Diamond storage
    }
    
    function _mintGameItems(address to, uint256 tierId, uint256 quantity) internal {
        // Placeholder for game item minting
    }
    
    modifier onlyOwner() {
        // Placeholder for ownership check
        _;
    }
}
```

### Airdrop Distribution System
```solidity
// Airdrop system for tokens using multi-sale capabilities
contract AirdropDistributor {
    using MultiSaleLib for MultiSaleLib.MultiSaleStorage;
    
    struct AirdropConfig {
        uint256 tokenSaleId;
        bytes32 merkleRoot;
        uint256 startTime;
        uint256 endTime;
        bool active;
    }
    
    mapping(uint256 => AirdropConfig) public airdrops;
    IERC20 public airdropToken;
    
    event AirdropCreated(uint256 indexed airdropId, bytes32 merkleRoot);
    event TokensClaimed(uint256 indexed airdropId, address indexed claimer, uint256 amount);
    
    constructor(address _airdropToken) {
        airdropToken = IERC20(_airdropToken);
    }
    
    function createAirdrop(
        bytes32 merkleRoot,
        uint256 startTime,
        uint256 endTime
    ) external onlyOwner returns (uint256 airdropId) {
        airdropId = MultiSaleLib._createTokenSale();
        
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(airdropId);
        sale.settings.name = "Airdrop";
        sale.settings.whitelistOnly = true;
        sale.settings.whitelistHash = uint256(merkleRoot);
        sale.settings.startTime = startTime;
        sale.settings.endTime = endTime;
        sale.settings.paymentMethod = MultiSaleLib.PaymentMethod.Native; // Not actually paying
        sale.settings.maxQuantityPerAccount = type(uint256).max; // No limit
        sale.settings.token = address(airdropToken);
        sale.settings.tokenType = MultiSaleLib.TokenType.ERC20;
        
        airdrops[airdropId] = AirdropConfig({
            tokenSaleId: airdropId,
            merkleRoot: merkleRoot,
            startTime: startTime,
            endTime: endTime,
            active: true
        });
        
        emit AirdropCreated(airdropId, merkleRoot);
    }
    
    function claimAirdrop(
        uint256 airdropId,
        uint256 amount,
        bytes32[] memory proof
    ) external {
        AirdropConfig storage config = airdrops[airdropId];
        require(config.active, "Airdrop not active");
        require(block.timestamp >= config.startTime, "Airdrop not started");
        require(block.timestamp <= config.endTime, "Airdrop ended");
        
        MultiSaleLib.MultiSaleContract storage sale = getSaleContract(airdropId);
        
        // Construct Merkle proof for claiming
        MultiSaleLib.MultiSalePurchase memory purchase = MultiSaleLib.MultiSalePurchase({
            multiSaleId: airdropId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: amount
        });
        
        MultiSaleLib.MultiSaleProof memory multiSaleProof = MultiSaleLib.MultiSaleProof({
            leaf: uint256(MerkleProver.getHash(msg.sender, amount)), // Assuming MerkleProver.getHash is available
            total: amount,
            merkleProof: proof,
            data: ""
        });
        
        // Validate proof and redeem
        MultiSaleLib._validateProof(sale, purchase, multiSaleProof);
        
        // Transfer tokens from contract balance
        require(airdropToken.transfer(msg.sender, amount), "Token transfer failed");
        
        emit TokensClaimed(airdropId, msg.sender, amount);
    }
    
    function getSaleContract(uint256 saleId) internal view returns (MultiSaleLib.MultiSaleContract storage) {
        // Implementation would access Diamond storage
    }
    
    // Assuming MerkleProver.sol is imported and available
    // using MerkleProver for bytes32;
    
    modifier onlyOwner() {
        // Placeholder for ownership check
        _;
    }
}
```

## Security Considerations

### Merkle Proof Security
- **Root Validation**: Ensure the Merkle root is correctly set and validated.
- **Used Leaves**: Implement robust tracking to prevent double-claiming of whitelist allocations.
- **Proof Generation**: Off-chain Merkle proof generation must be secure.

### Payment Security
- **Sufficient Funds**: Validate that the buyer sends enough funds for the purchase.
- **Excess Refund**: Properly handle and refund any excess payment.
- **Approvals**: For ERC20 payments, ensure the contract has received the necessary token approvals.

### Access Control
- **Sale Configuration**: Restrict configuration changes to authorized parties.
- **Minting Permissions**: Only the sale contract should be able to trigger token minting.

## Gas Optimization

### Storage Efficiency
- The primary storage for MultiSale is `MultiSaleStorage`, which is structured to minimize storage slots.
- Using mappings for `_tokenSales` allows for efficient retrieval of sale configurations.

### Function Efficiency
- `_validatePurchase` and `_validateProof` are key functions optimized for minimal gas usage.
- Batch operations (e.g., in `batchMintTo` if integrated with a minter) can further reduce overall transaction costs.

## Error Handling

### Common Errors
- `MultiSaleLib: Sale not active`: Attempting to purchase from an inactive sale.
- `MultiSaleLib: Invalid quantity`: Purchase quantity is out of bounds (min/max per sale).
- `MultiSaleLib: Insufficient payment`: Not enough ETH or ERC20 tokens sent.
- `MultiSaleLib: Merkle proof invalid`: Merkle proof validation failed.
- `MultiSaleLib: Allocation used`: User has already claimed their whitelist allocation.
- `MultiSaleLib: Max quantity per account reached`: User attempted to purchase more than their per-account limit.
- `MultiSaleLib: Sale ended`: Attempting to purchase after the sale's end time.

## Best Practices

### Sale Configuration
- Clearly define all sale parameters (prices, quantities, times, types).
- Use `whitelistOnly` and `whitelistHash` carefully for controlled access.

### Integration Checklist
- Ensure proper token approvals are handled on the frontend for ERC20 sales.
- Sync sale status and availability to the frontend regularly.
- Provide clear error messages to users based on common error conditions.

### Development Guidelines
- Write comprehensive unit tests for all sale logic, including edge cases.
- Conduct thorough security audits for the MultiSaleFacet and any contracts integrating it.
- Monitor sale events closely for analytics and anomaly detection.

## Related Documentation

- [Multi Sale Facet](../../smart-contracts/facets/multi-sale-facet.md) - Reference for the Multi Sale Facet implementation.
- [IMultiSale Interface](../../smart-contracts/interfaces/imultisale.md) - Interface definition.
- [EIP-DRAFT-Multi-Token-Sale-Standard](../../eips/EIP-DRAFT-Multi-Token-Sale-Standard.md) - The full EIP specification.
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)