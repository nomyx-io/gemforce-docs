# FeeDistributorLib Library

## Overview

The [`FeeDistributorLib`](../../smart-contracts/libraries/fee-distributor-lib.md) library provides core utilities for automated fee distribution within the Gemforce platform. This library implements flexible revenue sharing mechanisms that automatically distribute fees to multiple recipients based on configurable weights, supporting both native ETH and ERC20 token distributions.

## Key Features

- **Flexible Fee Distribution**: Configurable weight-based fee allocation
- **Multi-Currency Support**: Native ETH and ERC20 token distribution
- **Basis Point Precision**: Accurate percentage-based fee calculations
- **Automatic Distribution**: Seamless integration with transaction flows
- **Safety Checks**: Comprehensive validation and overflow protection
- **Gas Efficient**: Optimized batch distribution operations

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library FeeDistributorLib {
    // Initialization
    function _initializeFeeDistributor(IdentitySystemStorage.IdentitySystem storage ds, address _distributionToken, uint256 _totalWeightBasis) internal;
    
    // Configuration
    function _setFeeReceivers(IdentitySystemStorage.IdentitySystem storage ds, address[] memory _feeReceivers, uint256[] memory _feeWeights) internal returns (address[] memory, uint256[] memory);
    function _getFeeReceivers(IdentitySystemStorage.IdentitySystem storage ds) internal view returns (address[] memory, uint256[] memory);
    
    // Calculations
    function _calculateAmounts(IdentitySystemStorage.IdentitySystem storage ds, uint256 principalAmount) internal view returns (uint256 adjustedAmount, uint256[] memory feeAmounts);
    
    // Distribution
    function _distributeAmounts(IdentitySystemStorage.IdentitySystem storage ds, address self, address principalAmountReceiver, uint256 _principalAmount) internal returns (address, uint256, address[] memory, uint256[] memory);
}
```

## Data Structures

### FeeDistributorStorage Struct
```solidity
struct FeeDistributorStorage {
    address[] feeReceivers;         // Array of fee recipient addresses
    uint256[] feeWeights;           // Array of weights for each recipient
    uint256 totalWeightBasis;       // Total basis for weight calculations (e.g., 10000 for basis points)
    address distributionToken;      // ERC20 token address (address(0) for native ETH)
}
```

**Purpose**: Complete configuration for fee distribution system.

**Components**:
- **feeReceivers**: Addresses that will receive fee distributions
- **feeWeights**: Corresponding weights for each receiver (must sum to totalWeightBasis)
- **totalWeightBasis**: Denominator for percentage calculations (typically 10000 for basis points)
- **distributionToken**: Token contract address or address(0) for native ETH

## Core Functions

### Initialization

#### `_initializeFeeDistributor()`
```solidity
function _initializeFeeDistributor(
    IdentitySystemStorage.IdentitySystem storage ds,
    address _distributionToken,
    uint256 _totalWeightBasis
) internal
```

**Purpose**: Initialize the fee distribution system with basic parameters.

**Parameters**:
- `ds`: Diamond storage reference
- `_distributionToken`: Token contract address (address(0) for native ETH)
- `_totalWeightBasis`: Basis for weight calculations (e.g., 10000 for basis points)

**Validation**:
- Prevents re-initialization
- Requires positive weight basis
- Allows address(0) for native currency distribution

**Example Usage**:
```solidity
// Initialize for USDC distribution with basis points
FeeDistributorLib._initializeFeeDistributor(
    ds,
    0xA0b86a33E6441c8C06DD2b7c94b7E0e8c0c8c8c8, // USDC address
    10000 // 10000 basis points = 100%
);

// Initialize for native ETH distribution
FeeDistributorLib._initializeFeeDistributor(
    ds,
    address(0), // Native ETH
    10000
);
```

### Configuration Management

#### `_setFeeReceivers()`
```solidity
function _setFeeReceivers(
    IdentitySystemStorage.IdentitySystem storage ds,
    address[] memory _feeReceivers,
    uint256[] memory _feeWeights
) internal returns (address[] memory, uint256[] memory)
```

**Purpose**: Configure fee recipients and their distribution weights.

**Parameters**:
- `ds`: Diamond storage reference
- `_feeReceivers`: Array of recipient addresses
- `_feeWeights`: Array of weights corresponding to each recipient

**Returns**: Arrays of receivers and weights for event emission

**Validation**:
- Array lengths must match
- At least one receiver required
- No zero addresses allowed
- All weights must be positive
- Weights must sum exactly to totalWeightBasis

**Example Usage**:
```solidity
// Set up 3-way fee distribution
address[] memory receivers = new address[](3);
receivers[0] = 0x1234...; // Platform treasury
receivers[1] = 0x5678...; // Development fund
receivers[2] = 0x9abc...; // Marketing fund

uint256[] memory weights = new uint256[](3);
weights[0] = 5000; // 50%
weights[1] = 3000; // 30%
weights[2] = 2000; // 20%

FeeDistributorLib._setFeeReceivers(ds, receivers, weights);
```

#### `_getFeeReceivers()`
```solidity
function _getFeeReceivers(
    IdentitySystemStorage.IdentitySystem storage ds
) internal view returns (
    address[] memory feeReceivers_,
    uint256[] memory feeWeights_
)
```

**Purpose**: Retrieve current fee distribution configuration.

**Parameters**:
- `ds`: Diamond storage reference

**Returns**: Arrays of current receivers and their weights

**Example Usage**:
```solidity
// Get current fee configuration
(address[] memory receivers, uint256[] memory weights) = FeeDistributorLib._getFeeReceivers(ds);

for (uint256 i = 0; i < receivers.length; i++) {
    console.log("Receiver:", receivers[i], "Weight:", weights[i]);
}
```

### Fee Calculations

#### `_calculateAmounts()`
```solidity
function _calculateAmounts(
    IdentitySystemStorage.IdentitySystem storage ds,
    uint256 principalAmount
) internal view returns (
    uint256 adjustedAmount,
    uint256[] memory feeAmounts
)
```

**Purpose**: Calculate fee distributions and remaining principal amount.

**Parameters**:
- `ds`: Diamond storage reference
- `principalAmount`: Total amount before fee deduction

**Returns**:
- `adjustedAmount`: Principal amount after fee deduction
- `feeAmounts`: Array of individual fee amounts for each receiver

**Calculation Logic**:
```solidity
// For each receiver:
feeAmount = (principalAmount * weight) / totalWeightBasis

// Adjusted amount:
adjustedAmount = principalAmount - sum(feeAmounts)
```

**Safety Features**:
- Handles zero receivers (pass-through mode)
- Prevents total fees from exceeding principal
- Protects against overflow in calculations

**Example Usage**:
```solidity
// Calculate fees for a 1000 USDC transaction
uint256 principal = 1000 * 10**6; // 1000 USDC
(uint256 adjusted, uint256[] memory fees) = FeeDistributorLib._calculateAmounts(ds, principal);

console.log("Principal after fees:", adjusted);
for (uint256 i = 0; i < fees.length; i++) {
    console.log("Fee", i, ":", fees[i]);
}
```

### Distribution Execution

#### `_distributeAmounts()`
```solidity
function _distributeAmounts(
    IdentitySystemStorage.IdentitySystem storage ds,
    address self,
    address principalAmountReceiver,
    uint256 _principalAmount
) internal returns (
    address adjustedAmountReceiver_,
    uint256 adjustedAmount_,
    address[] memory feeReceivers_,
    uint256[] memory feeAmounts_
)
```

**Purpose**: Execute the actual distribution of fees and principal amount.

**Parameters**:
- `ds`: Diamond storage reference
- `self`: Address of the calling contract (for balance checks)
- `principalAmountReceiver`: Address to receive the adjusted principal
- `_principalAmount`: Total amount to distribute

**Returns**: Distribution details for event emission

**Process**:
1. Calculate fee amounts and adjusted principal
2. Check contract balance sufficiency
3. Transfer fees to each receiver
4. Transfer adjusted principal to designated receiver
5. Return distribution details

**Native ETH Distribution**:
```solidity
// Check balance
require(self.balance >= _principalAmount, "Insufficient native balance");

// Transfer fees
for (uint i = 0; i < receivers.length; i++) {
    if (feeAmounts[i] > 0) {
        (bool success, ) = payable(receivers[i]).call{value: feeAmounts[i]}("");
        require(success, "Native fee transfer failed");
    }
}

// Transfer principal
if (adjustedAmount > 0) {
    (bool success, ) = payable(principalAmountReceiver).call{value: adjustedAmount}("");
    require(success, "Native principal transfer failed");
}
```

**ERC20 Token Distribution**:
```solidity
// Check balance
IERC20 token = IERC20(distributionToken);
require(token.balanceOf(self) >= _principalAmount, "Insufficient token balance");

// Transfer fees
for (uint i = 0; i < receivers.length; i++) {
    if (feeAmounts[i] > 0) {
        token.safeTransfer(receivers[i], feeAmounts[i]);
    }
}

// Transfer principal
if (adjustedAmount > 0) {
    token.safeTransfer(principalAmountReceiver, adjustedAmount);
}
```

## Integration Examples

### NFT Marketplace with Revenue Sharing
```solidity
// NFT marketplace with automatic fee distribution
contract NFTMarketplace {
    using FeeDistributorLib for IdentitySystemStorage.IdentitySystem;
    
    struct Listing {
        uint256 tokenId;
        address seller;
        uint256 price;
        bool active;
    }
    
    struct MarketplaceConfig {
        uint256 platformFee; // Basis points
        address platformTreasury;
        address developmentFund;
        address marketingFund;
    }
    
    mapping(uint256 => Listing) public listings;
    MarketplaceConfig public config;
    IdentitySystemStorage.IdentitySystem internal ds;
    
    event ListingCreated(uint256 indexed tokenId, address indexed seller, uint256 price);
    event ItemSold(uint256 indexed tokenId, address indexed buyer, address indexed seller, uint256 price);
    event FeesDistributed(address[] receivers, uint256[] amounts, uint256 totalFees);
    
    constructor(
        address _platformTreasury,
        address _developmentFund,
        address _marketingFund,
        uint256 _platformFee
    ) {
        config = MarketplaceConfig({
            platformFee: _platformFee,
            platformTreasury: _platformTreasury,
            developmentFund: _developmentFund,
            marketingFund: _marketingFund
        });
        
        // Initialize fee distributor for native ETH
        FeeDistributorLib._initializeFeeDistributor(ds, address(0), 10000);
        
        // Set up fee distribution: 60% treasury, 25% development, 15% marketing
        address[] memory receivers = new address[](3);
        receivers[0] = _platformTreasury;
        receivers[1] = _developmentFund;
        receivers[2] = _marketingFund;
        
        uint256[] memory weights = new uint256[](3);
        weights[0] = 6000; // 60%
        weights[1] = 2500; // 25%
        weights[2] = 1500; // 15%
        
        FeeDistributorLib._setFeeReceivers(ds, receivers, weights);
    }
    
    function createListing(uint256 tokenId, uint256 price) external {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        require(price > 0, "Price must be positive");
        
        listings[tokenId] = Listing({
            tokenId: tokenId,
            seller: msg.sender,
            price: price,
            active: true
        });
        
        emit ListingCreated(tokenId, msg.sender, price);
    }
    
    function buyItem(uint256 tokenId) external payable {
        Listing storage listing = listings[tokenId];
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        
        listing.active = false;
        
        // Calculate platform fee
        uint256 platformFeeAmount = (listing.price * config.platformFee) / 10000;
        uint256 sellerAmount = listing.price - platformFeeAmount;
        
        // Transfer NFT to buyer
        _transfer(listing.seller, msg.sender, tokenId);
        
        // Distribute platform fees
        if (platformFeeAmount > 0) {
            (
                address adjustedReceiver,
                uint256 adjustedAmount,
                address[] memory feeReceivers,
                uint256[] memory feeAmounts
            ) = FeeDistributorLib._distributeAmounts(
                ds,
                address(this),
                address(this), // Platform keeps any remainder
                platformFeeAmount
            );
            
            emit FeesDistributed(feeReceivers, feeAmounts, platformFeeAmount);
        }
        
        // Transfer remaining amount to seller
        if (sellerAmount > 0) {
            payable(listing.seller).transfer(sellerAmount);
        }
        
        // Refund excess payment
        if (msg.value > listing.price) {
            payable(msg.sender).transfer(msg.value - listing.price);
        }
        
        emit ItemSold(tokenId, msg.sender, listing.seller, listing.price);
    }
    
    function updateFeeDistribution(
        address[] memory newReceivers,
        uint256[] memory newWeights
    ) external onlyOwner {
        FeeDistributorLib._setFeeReceivers(ds, newReceivers, newWeights);
    }
    
    function getFeeDistribution() external view returns (
        address[] memory receivers,
        uint256[] memory weights
    ) {
        return FeeDistributorLib._getFeeReceivers(ds);
    }
    
    function _transfer(address from, address to, uint256 tokenId) internal {
        // Implementation would transfer ERC721 token
    }
    
    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        // Implementation would check ERC721 authorization
        return true;
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

### DeFi Protocol with Token Distribution
```solidity
// DeFi protocol with automatic token fee distribution
contract DeFiProtocol {
    using FeeDistributorLib for IdentitySystemStorage.IdentitySystem;
    using SafeERC20 for IERC20;
    
    struct ProtocolConfig {
        IERC20 rewardToken;
        uint256 protocolFee; // Basis points
        uint256 totalValueLocked;
        uint256 totalRewardsDistributed;
    }
    
    struct StakeInfo {
        uint256 amount;
        uint256 rewardDebt;
        uint256 lastStakeTime;
    }
    
    ProtocolConfig public config;
    mapping(address => StakeInfo) public stakes;
    IdentitySystemStorage.IdentitySystem internal ds;
    
    event Staked(address indexed user, uint256 amount);
    event Unstaked(address indexed user, uint256 amount);
    event RewardsDistributed(address[] stakeholders, uint256[] amounts, uint256 totalRewards);
    event ProtocolFeesDistributed(address[] receivers, uint256[] amounts);
    
    constructor(
        address _rewardToken,
        uint256 _protocolFee,
        address _treasury,
        address _developmentFund,
        address _stakeholderRewards
    ) {
        config.rewardToken = IERC20(_rewardToken);
        config.protocolFee = _protocolFee;
        
        // Initialize fee distributor for reward token
        FeeDistributorLib._initializeFeeDistributor(ds, _rewardToken, 10000);
        
        // Set up fee distribution: 40% treasury, 30% development, 30% stakeholder rewards
        address[] memory receivers = new address[](3);
        receivers[0] = _treasury;
        receivers[1] = _developmentFund;
        receivers[2] = _stakeholderRewards;
        
        uint256[] memory weights = new uint256[](3);
        weights[0] = 4000; // 40%
        weights[1] = 3000; // 30%
        weights[2] = 3000; // 30%
        
        FeeDistributorLib._setFeeReceivers(ds, receivers, weights);
    }
    
    function stake(uint256 amount) external {
        require(amount > 0, "Amount must be positive");
        
        // Transfer tokens from user
        config.rewardToken.safeTransferFrom(msg.sender, address(this), amount);
        
        // Update stake info
        stakes[msg.sender].amount += amount;
        stakes[msg.sender].lastStakeTime = block.timestamp;
        
        // Update TVL
        config.totalValueLocked += amount;
        
        emit Staked(msg.sender, amount);
    }
    
    function unstake(uint256 amount) external {
        require(amount > 0, "Amount must be positive");
        require(stakes[msg.sender].amount >= amount, "Insufficient staked amount");
        
        // Calculate accrued rewards
        uint256 rewards = calculateRewards(msg.sender);
        
        // Transfer rewards
        config.rewardToken.safeTransfer(msg.sender, rewards);
        config.totalRewardsDistributed += rewards;
        
        // Transfer tokens back to user
        config.rewardToken.safeTransfer(msg.sender, amount);
        
        // Update stake info
        stakes[msg.sender].amount -= amount;
        stakes[msg.sender].rewardDebt = 0; // Reset debt after withdrawal
        
        // Update TVL
        config.totalValueLocked -= amount;
        
        emit Unstaked(msg.sender, amount);
    }
    
    function calculateRewards(address user) public view returns (uint256) {
        // Implement reward calculation logic based on staking duration, amount, etc.
        return 0;
    }
    
    function distributeProtocolFees(uint256 amount) external onlyOwner {
        require(amount > 0, "Amount must be positive");
        config.rewardToken.safeTransferFrom(msg.sender, address(this), amount); // Assume sender is the fee collector
        
        (
            address adjustedReceiver,
            uint256 adjustedAmount,
            address[] memory feeReceivers,
            uint256[] memory feeAmounts
        ) = FeeDistributorLib._distributeAmounts(
            ds,
            address(this),
            address(this), // Unused if all fees are distributed
            amount
        );
        
        // Emit event for protocol fees
        emit ProtocolFeesDistributed(feeReceivers, feeAmounts);
    }
    
    modifier onlyOwner() {
        // Placeholder for access control
        _;
    }
}
```

### Gaming Platform Revenue Distribution
```solidity
// Gaming platform with revenue sharing for token holders and developers
contract GamingPlatform {
    using FeeDistributorLib for IdentitySystemStorage.IdentitySystem;
    using SafeERC20 for IERC20;
    
    struct GameConfig {
        IERC20 platformToken;
        uint256 gameRevenueShare; // Basis points for game developers
        uint256 tokenHolderShare; // Basis points for token holders
        uint256 platformShare;    // Basis points for platform treasury
        address platformTreasury;
        address gameDeveloperFund;
    }
    
    GameConfig public config;
    IdentitySystemStorage.IdentitySystem internal ds;
    
    event RevenueReceived(uint256 amount);
    event PlatformFeesDistributed(address[] receivers, uint256[] amounts);
    
    constructor(
        address _platformToken,
        uint256 _gameRevenueShare,
        uint256 _tokenHolderShare,
        uint256 _platformShare,
        address _platformTreasury,
        address _gameDeveloperFund,
        address _tokenHolderRewards  // Address where token holders can claim/receive
    ) {
        config.platformToken = IERC20(_platformToken);
        config.gameRevenueShare = _gameRevenueShare;
        config.tokenHolderShare = _tokenHolderShare;
        config.platformShare = _platformShare;
        config.platformTreasury = _platformTreasury;
        config.gameDeveloperFund = _gameDeveloperFund;
        
        // Initialize fee distributor for platform token
        FeeDistributorLib._initializeFeeDistributor(ds, _platformToken, 10000);
        
        // Set up fee distribution based on configured shares
        address[] memory receivers = new address[](3);
        receivers[0] = _gameDeveloperFund;
        receivers[1] = _tokenHolderRewards;
        receivers[2] = _platformTreasury;
        
        uint256[] memory weights = new uint256[](3);
        weights[0] = _gameRevenueShare;
        weights[1] = _tokenHolderShare;
        weights[2] = _platformShare;
        
        FeeDistributorLib._setFeeReceivers(ds, receivers, weights);
    }
    
    function depositRevenue(uint256 amount) external payable { // Assuming ETH revenue
        require(amount > 0, "Amount must be positive");
        
        // Distribute revenue among receivers
        (
            address adjustedReceiver, // Not used if self is 0x0
            uint256 adjustedAmount,   // Not used if all is distributed
            address[] memory feeReceivers,
            uint256[] memory feeAmounts
        ) = FeeDistributorLib._distributeAmounts(
            ds,
            address(0), // Distribute from msg.value
            address(0), // No single principal receiver, all distributed
            amount
        );
        
        emit RevenueReceived(amount);
        emit PlatformFeesDistributed(feeReceivers, feeAmounts);
    }
}
```

## Security Considerations

### Financial Security
- **Accurate Calculations**: Ensures correct distribution based on weights and principal.
- **No Funds Locked**: Prevents funds from being stuck in the contract.
- **Overflow/Underflow Protection**: Uses safe math for all arithmetic operations.

### Access Control
- **Owner-Only Configuration**: Only authorized entities can set/update fee receivers and weights.
- **Trusted Distribution**: Ensures funds are sent only to configured recipients.

### Distribution Integrity
- **Atomic Transfers**: Ensures all transfers within a distribution are atomic (all or nothing).
- **Event Logging**: Provides a clear, auditable trail of all distributions.

## Gas Optimization

### Batch Operations
- `_setFeeReceivers` and `_distributeAmounts` are designed to handle arrays, optimizing gas for multiple recipients.

### Storage Efficiency
- The `FeeDistributorStorage` struct uses packed storage to minimize gas costs.
- Minimizes state writes during distribution by pre-calculating amounts.

## Error Handling

### Common Errors
- `FeeDistributorLib: Invalid balance basis`: `_totalWeightBasis` is zero.
- `FeeDistributorLib: Already initialized`: Attempting to re-initialize the component.
- `FeeDistributorLib: Arrays length mismatch`: `_feeReceivers` and `_feeWeights` have different lengths.
- `FeeDistributorLib: Invalid receiver`: Attempting to set a zero address as a receiver.
- `FeeDistributorLib: Invalid weight`: Attempting to set a zero weight.
- `FeeDistributorLib: Total weights must sum to totalWeightBasis`: Weights don't sum correctly.
- `FeeDistributorLib: Insufficient native balance`: Contract doesn't have enough ETH for distribution.
- `FeeDistributorLib: Insufficient token balance`: Contract doesn't have enough ERC20 tokens for distribution.
- `Native fee transfer failed` / `Native principal transfer failed` / `ERC20 transfer failed`: Underlying transfer operation failed.

## Best Practices

### Configuration Management
- Set `_totalWeightBasis` to a power of 10 (e.g., 100 or 10000) for easier percentage calculations.
- Regularly review and update fee receiver configurations as business needs change.

### Integration Checklist
- Ensure the calling contract has sufficient balance (ETH or ERC20) before initiating a distribution.
- Implement clear event listeners to track `FeesDistributed` events for off-chain analytics.
- Grant `call` permission to the FeeDistributor facet if it needs to transfer native ETH.

### Development Guidelines
- Write comprehensive unit tests for all distribution scenarios, including edge cases.
- Use `SafeERC20` (from OpenZeppelin) for robust ERC20 token interactions.
- Consider a separate facet for managing fee distribution parameters if administrative control is needed.

## Related Documentation

- [Fee Distributor Facet](../../smart-contracts/facets/fee-distributor-facet.md) - Reference for the Fee Distributor Facet implementation.
- [IFeeDistributor Interface](../../smart-contracts/interfaces/ifee-distributor.md) - Interface definition.
- [EIP-DRAFT-Diamond-Enhanced-Marketplace](../../eips/EIP-DRAFT-Diamond-Enhanced-Marketplace.md) - Contains marketplace use cases.
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)