# FeeDistributorLib Library

## Overview

The [`FeeDistributorLib`](../../../contracts/libraries/FeeDistributorLib.sol) library provides core utilities for automated fee distribution within the Gemforce platform. This library implements flexible revenue sharing mechanisms that automatically distribute fees to multiple recipients based on configurable weights, supporting both native ETH and ERC20 token distributions.

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
        StakeInfo storage stakeInfo = stakes[msg.sender];
        stakeInfo.amount += amount;
        stakeInfo.lastStakeTime = block.timestamp;
        
        config.totalValueLocked += amount;
        
        emit Staked(msg.sender, amount);
    }
    
    function unstake(uint256 amount) external {
        StakeInfo storage stakeInfo = stakes[msg.sender];
        require(stakeInfo.amount >= amount, "Insufficient stake");
        
        stakeInfo.amount -= amount;
        config.totalValueLocked -= amount;
        
        // Calculate protocol fee on unstaking
        uint256 protocolFeeAmount = (amount * config.protocolFee) / 10000;
        uint256 userAmount = amount - protocolFeeAmount;
        
        // Distribute protocol fees
        if (protocolFeeAmount > 0) {
            (
                address adjustedReceiver,
                uint256 adjustedAmount,
                address[] memory feeReceivers,
                uint256[] memory feeAmounts
            ) = FeeDistributorLib._distributeAmounts(
                ds,
                address(this),
                address(this), // Protocol keeps remainder
                protocolFeeAmount
            );
            
            emit ProtocolFeesDistributed(feeReceivers, feeAmounts);
        }
        
        // Transfer remaining amount to user
        if (userAmount > 0) {
            config.rewardToken.safeTransfer(msg.sender, userAmount);
        }
        
        emit Unstaked(msg.sender, amount);
    }
    
    function distributeRewards(uint256 totalRewards) external onlyOwner {
        require(totalRewards > 0, "Rewards must be positive");
        require(config.totalValueLocked > 0, "No stakes to distribute to");
        
        // Transfer rewards to contract
        config.rewardToken.safeTransferFrom(msg.sender, address(this), totalRewards);
        
        // Calculate individual rewards based on stake proportion
        // This is a simplified example - real implementation would be more complex
        address[] memory stakeholders = _getStakeholders();
        uint256[] memory rewardAmounts = new uint256[](stakeholders.length);
        
        for (uint256 i = 0; i < stakeholders.length; i++) {
            address stakeholder = stakeholders[i];
            uint256 stakeAmount = stakes[stakeholder].amount;
            uint256 reward = (totalRewards * stakeAmount) / config.totalValueLocked;
            rewardAmounts[i] = reward;
            
            if (reward > 0) {
                config.rewardToken.safeTransfer(stakeholder, reward);
            }
        }
        
        config.totalRewardsDistributed += totalRewards;
        
        emit RewardsDistributed(stakeholders, rewardAmounts, totalRewards);
    }
    
    function updateFeeDistribution(
        address[] memory newReceivers,
        uint256[] memory newWeights
    ) external onlyOwner {
        FeeDistributorLib._setFeeReceivers(ds, newReceivers, newWeights);
    }
    
    function getProtocolStats() external view returns (
        uint256 totalValueLocked,
        uint256 totalRewardsDistributed,
        uint256 protocolFee,
        address[] memory feeReceivers,
        uint256[] memory feeWeights
    ) {
        totalValueLocked = config.totalValueLocked;
        totalRewardsDistributed = config.totalRewardsDistributed;
        protocolFee = config.protocolFee;
        (feeReceivers, feeWeights) = FeeDistributorLib._getFeeReceivers(ds);
    }
    
    function _getStakeholders() internal view returns (address[] memory) {
        // Implementation would return array of addresses with active stakes
        // This is simplified for example purposes
        address[] memory stakeholders = new address[](1);
        stakeholders[0] = msg.sender;
        return stakeholders;
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

### Gaming Platform Revenue Distribution
```solidity
// Gaming platform with multi-tier revenue sharing
contract GamePlatform {
    using FeeDistributorLib for IdentitySystemStorage.IdentitySystem;
    using SafeERC20 for IERC20;
    
    struct GameDeveloper {
        address developerAddress;
        uint256 revenueShare; // Basis points
        uint256 totalEarnings;
        bool active;
    }
    
    struct GameSession {
        uint256 gameId;
        address player;
        uint256 cost;
        uint256 timestamp;
        bool completed;
    }
    
    mapping(uint256 => GameDeveloper) public developers;
    mapping(uint256 => GameSession) public sessions;
    mapping(uint256 => uint256) public gameDeveloper; // gameId => developerId
    
    IERC20 public platformToken;
    IdentitySystemStorage.IdentitySystem internal ds;
    uint256 public nextDeveloperId;
    uint256 public nextSessionId;
    
    event DeveloperRegistered(uint256 indexed developerId, address developer, uint256 revenueShare);
    event GameSessionStarted(uint256 indexed sessionId, uint256 gameId, address player, uint256 cost);
    event RevenueDistributed(uint256 indexed sessionId, address developer, uint256 developerShare, uint256 platformShare);
    
    constructor(address _platformToken) {
        platformToken = IERC20(_platformToken);
        
        // Initialize fee distributor for platform token
        FeeDistributorLib._initializeFeeDistributor(ds, _platformToken, 10000);
        
        // Set up platform revenue distribution
        address[] memory receivers = new address[](4);
        receivers[0] = address(this); // Platform treasury
        receivers[1] = 0x1234567890123456789012345678901234567890; // Operations
        receivers[2] = 0x2345678901234567890123456789012345678901; // Development
        receivers[3] = 0x3456789012345678901234567890123456789012; // Marketing
        
        uint256[] memory weights = new uint256[](4);
        weights[0] = 4000; // 40% platform treasury
        weights[1] = 2500; // 25% operations
        weights[2] = 2000; // 20% development
        weights[3] = 1500; // 15% marketing
        
        FeeDistributorLib._setFeeReceivers(ds, receivers, weights);
    }
    
    function registerDeveloper(
        address developerAddress,
        uint256 revenueShare
    ) external onlyOwner returns (uint256 developerId) {
        require(developerAddress != address(0), "Invalid developer address");
        require(revenueShare <= 7000, "Revenue share too high"); // Max 70%
        
        developerId = nextDeveloperId++;
        
        developers[developerId] = GameDeveloper({
            developerAddress: developerAddress,
            revenueShare: revenueShare,
            totalEarnings: 0,
            active: true
        });
        
        emit DeveloperRegistered(developerId, developerAddress, revenueShare);
    }
    
    function startGameSession(
        uint256 gameId,
        uint256 cost
    ) external returns (uint256 sessionId) {
        require(cost > 0, "Cost must be positive");
        require(gameDeveloper[gameId] != 0, "Game not registered");
        
        // Transfer payment from player
        platformToken.safeTransferFrom(msg.sender, address(this), cost);
        
        sessionId = nextSessionId++;
        
        sessions[sessionId] = GameSession({
            gameId: gameId,
            player: msg.sender,
            cost: cost,
            timestamp: block.timestamp,
            completed: false
        });
        
        emit GameSessionStarted(sessionId, gameId, msg.sender, cost);
    }
    
    function completeGameSession(uint256 sessionId) external {
        GameSession storage session = sessions[sessionId];
        require(!session.completed, "Session already completed");
        require(session.player == msg.sender, "Not session owner");
        
        session.completed = true;
        
        uint256 developerId = gameDeveloper[session.gameId];
        GameDeveloper storage developer = developers[developerId];
        require(developer.active, "Developer not active");
        
        // Calculate developer share
        uint256 developerShare = (session.cost * developer.revenueShare) / 10000;
        uint256 platformShare = session.cost - developerShare;
        
        // Transfer developer share directly
        if (developerShare > 0) {
            platformToken.safeTransfer(developer.developerAddress, developerShare);
            developer.totalEarnings += developerShare;
        }
        
        // Distribute platform share among platform stakeholders
        if (platformShare > 0) {
            (
                address adjustedReceiver,
                uint256 adjustedAmount,
                address[] memory feeReceivers,
                uint256[] memory feeAmounts
            ) = FeeDistributorLib._distributeAmounts(
                ds,
                address(this),
                address(this), // Platform treasury gets adjusted amount
                platformShare
            );
        }
        
        emit RevenueDistributed(sessionId, developer.developerAddress, developerShare, platformShare);
    }
    
    function updatePlatformDistribution(
        address[] memory newReceivers,
        uint256[] memory newWeights
    ) external onlyOwner {
        FeeDistributorLib._setFeeReceivers(ds, newReceivers, newWeights);
    }
    
    function getDeveloperStats(uint256 developerId) external view returns (
        address developerAddress,
        uint256 revenueShare,
        uint256 totalEarnings,
        bool active
    ) {
        GameDeveloper memory dev = developers[developerId];
        return (dev.developerAddress, dev.revenueShare, dev.totalEarnings, dev.active);
    }
    
    function getPlatformDistribution() external view returns (
        address[] memory receivers,
        uint256[] memory weights
    ) {
        return FeeDistributorLib._getFeeReceivers(ds);
    }
    
    function registerGame(uint256 gameId, uint256 developerId) external onlyOwner {
        require(developers[developerId].active, "Developer not active");
        gameDeveloper[gameId] = developerId;
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

## Security Considerations

### Financial Security
- **Overflow Protection**: Safe arithmetic operations prevent calculation errors
- **Balance Validation**: Checks contract balance before distribution
- **Transfer Safety**: Uses SafeERC20 for token transfers and proper error handling for ETH
- **Weight Validation**: Ensures weights sum exactly to basis to prevent over/under distribution

### Access Control
- **Initialization Protection**: Prevents re-initialization of the system
- **Configuration Validation**: Validates all receiver addresses and weights
- **Administrative Functions**: Proper access control for configuration changes

### Distribution Integrity
- **Atomic Operations**: All distributions complete or fail together
- **Remainder Handling**: Proper handling of rounding remainders
- **Zero Amount Protection**: Skips zero-amount transfers to save gas

## Gas Optimization

### Batch Operations
- **Single Loop Distribution**: Efficient batch transfer operations
- **Minimal Storage Reads**: Optimized storage access patterns
- **Skip Zero Transfers**: Avoids unnecessary transfer operations
- **Efficient Calculations**: Optimized fee calculation algorithms

### Storage Efficiency
- **Compact Data Structures**: Efficient storage layout
- **Minimal Storage Writes**: Reduced storage operations
- **Array Operations**: Efficient array handling and iteration

## Error Handling

### Common Errors
- "Already initialized" - Attempt to re-initialize the system
- "Array lengths mismatch" - Mismatched receiver and weight arrays
- "Weights must sum to basis" - Invalid weight configuration
- "Insufficient balance" - Contract lacks funds for distribution
- "Transfer failed" - ETH or token transfer failure

### Best Practices
- Validate all inputs before processing
- Check contract balances before distribution
- Use proper error messages for debugging
- Handle edge cases gracefully (zero receivers, zero amounts)

## Testing Considerations

### Unit Tests
- Fee calculation accuracy
- Distribution logic validation
- Edge case handling (zero amounts, single receiver)
- Error condition testing

### Integration Tests
- Multi-contract distribution scenarios
- Token and ETH distribution workflows
- Configuration update scenarios
- Real-world usage patterns

### Security Testing
- Overflow/underflow protection
- Reentrancy attack prevention
- Access control validation
- Financial integrity verification

## Related Documentation

- [FeeDistributorFacet](../facets/fee-distributor-facet.md) - Fee distributor facet implementation
- [Revenue Sharing Guide](../../guides/revenue-sharing.md) - Revenue sharing implementation patterns
- [Tokenomics Guide](../../guides/tokenomics.md) - Token economics best practices
- [Financial Security](../../guides/financial-security.md) - Financial security considerations
- [Gas Optimization Guide](../../guides/gas-optimization.md) - Performance optimization techniques

---

*This library provides comprehensive utilities for automated fee distribution within the Gemforce platform, supporting flexible revenue sharing mechanisms with configurable weights, multi-currency support, and robust security features for various business models and tokenomics scenarios.*