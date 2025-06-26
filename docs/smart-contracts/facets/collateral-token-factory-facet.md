# CollateralTokenFactoryFacet

## Overview

The [`CollateralTokenFactoryFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/CollateralTokenFactoryFacet.sol) provides factory functionality for creating and managing restricted collateral tokens within the Gemforce diamond system. This facet enables the creation of ERC20 tokens that represent collateral positions in trade deals, with built-in transfer restrictions and participant validation.

## Contract Details

- **Contract Name**: `CollateralTokenFactoryFacet`
- **Inheritance**: `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Factory Pattern Implementation
- Automated factory deployment and initialization
- Template-based token creation using proxy patterns
- Centralized token management and tracking
- Efficient deployment of multiple token instances

### 🔹 Restricted Collateral Tokens
- ERC20 tokens with transfer restrictions
- Trade deal participant validation
- Configurable restriction enforcement
- Integration with identity verification systems

### 🔹 Trade Deal Integration
- One-to-one mapping between trade deals and tokens
- Automatic token creation for trade deals
- Participant management integration
- Collateral position tracking

### 🔹 Administrative Controls
- Owner-only token creation and configuration
- Restriction management for existing tokens
- Factory address management
- Comprehensive event logging

## Core Architecture

### Factory Pattern
The facet implements a factory pattern where:
- **Factory Contract**: `CollateralTokenFactory` manages token creation
- **Implementation Contract**: `RestrictedCollateralToken` serves as the template
- **Proxy Pattern**: New tokens are created as minimal proxies to save gas
- **Diamond Integration**: Factory is managed through the diamond storage

### Storage Structure
```solidity
struct CollateralTokenFactoryStorage {
    address factoryAddress;                    // Address of the factory contract
    mapping(uint256 => address) tradeDealToToken;  // Trade deal ID to token address mapping
}
```

### Token Restrictions
Collateral tokens can enforce restrictions on:
- **Transfer Limitations**: Only verified participants can transfer tokens
- **Participant Validation**: Integration with identity verification
- **Trade Deal Compliance**: Tokens tied to specific trade deal rules
- **Administrative Controls**: Owner can enable/disable restrictions

## Core Functions

### Initialization Functions

#### `CollateralTokenFactoryFacet_init()`
```solidity
function CollateralTokenFactoryFacet_init() external
```

**Purpose**: Initializes the collateral token factory when the facet is added to the diamond.

**Access Control**: Public (with initialization check)

**Process**:
1. Checks that factory hasn't been initialized already
2. Deploys a new `RestrictedCollateralToken` implementation
3. Creates a `CollateralTokenFactory` with the implementation
4. Stores the factory address in diamond storage
5. Emits CollateralTokenFactoryInitialized event

**Events**: `CollateralTokenFactoryInitialized(factoryAddress)`

**Example Usage**:
```solidity
// Called automatically when facet is added to diamond
// Or can be called manually if needed
ICollateralTokenFactory(diamond).CollateralTokenFactoryFacet_init();
console.log("Collateral token factory initialized");
```

**Requirements**:
- Factory must not be already initialized
- Caller must have appropriate permissions
- Sufficient gas for contract deployments

---

#### `getCollateralTokenFactory()`
```solidity
function getCollateralTokenFactory() external view returns (address)
```

**Purpose**: Returns the address of the deployed CollateralTokenFactory contract.

**Returns**:
- `address`: Address of the factory contract

**Example Usage**:
```solidity
address factoryAddress = ICollateralTokenFactory(diamond).getCollateralTokenFactory();
console.log("Factory deployed at:", factoryAddress);

// Use factory directly if needed
CollateralTokenFactory factory = CollateralTokenFactory(factoryAddress);
```

### Token Creation Functions

#### `createCollateralToken()`
```solidity
function createCollateralToken(
    uint256 _tradeDealId,
    string memory _name,
    string memory _symbol,
    bool _restrictionsEnabled
) external onlyOwner returns (address)
```

**Purpose**: Creates a new restricted collateral token for a specific trade deal.

**Parameters**:
- `_tradeDealId` (uint256): Unique identifier for the trade deal
- `_name` (string): Human-readable name for the token
- `_symbol` (string): Token symbol (typically 3-5 characters)
- `_restrictionsEnabled` (bool): Whether to enforce transfer restrictions

**Returns**:
- `address`: Address of the newly created token contract

**Access Control**: Owner only

**Process**:
1. Validates factory is initialized
2. Checks no token exists for the trade deal ID
3. Calls factory to create new token with specified parameters
4. Stores token address in trade deal mapping
5. Emits CollateralTokenCreated event

**Events**: `CollateralTokenCreated(tradeDealId, tokenAddress, name, symbol)`

**Example Usage**:
```solidity
// Create collateral token for trade deal
uint256 tradeDealId = 1;
string memory tokenName = "Trade Deal 1 Collateral";
string memory tokenSymbol = "TD1C";
bool restrictionsEnabled = true;

address tokenAddress = ICollateralTokenFactory(diamond).createCollateralToken(
    tradeDealId,
    tokenName,
    tokenSymbol,
    restrictionsEnabled
);

console.log("Created collateral token:", tokenAddress);
console.log("For trade deal:", tradeDealId);
```

**Token Configuration**:
- **Name**: Descriptive name for the collateral token
- **Symbol**: Short identifier for the token
- **Diamond Address**: Set as the token's diamond for integration
- **Trade Deal ID**: Links token to specific trade deal
- **Restrictions**: Controls transfer limitations

**Error Conditions**:
- `"factory not set"` - Factory not initialized
- `"token already exists for this trade deal"` - Duplicate creation attempt
- Access control failures for non-owners

### Query Functions

#### `getCollateralTokenAddress()`
```solidity
function getCollateralTokenAddress(uint256 _tradeDealId) external view returns (address)
```

**Purpose**: Retrieves the token address for a specific trade deal.

**Parameters**:
- `_tradeDealId` (uint256): ID of the trade deal

**Returns**:
- `address`: Address of the collateral token (zero address if not found)

**Process**:
1. Checks local mapping for token address
2. Falls back to factory lookup if not found locally
3. Returns zero address if token doesn't exist

**Example Usage**:
```solidity
uint256 tradeDealId = 1;
address tokenAddress = ICollateralTokenFactory(diamond).getCollateralTokenAddress(tradeDealId);

if (tokenAddress != address(0)) {
    console.log("Token found:", tokenAddress);
    
    // Interact with the token
    IERC20 collateralToken = IERC20(tokenAddress);
    uint256 balance = collateralToken.balanceOf(msg.sender);
    console.log("My collateral balance:", balance);
} else {
    console.log("No token found for trade deal:", tradeDealId);
}
```

---

#### `isCollateralTokenParticipant()`
```solidity
function isCollateralTokenParticipant(uint256 _tradeDealId, address _address) external view returns (bool)
```

**Purpose**: Checks if an address is a verified participant for a trade deal's collateral token.

**Parameters**:
- `_tradeDealId` (uint256): ID of the trade deal
- `_address` (address): Address to check for participation

**Returns**:
- `bool`: True if address is a verified participant

**Process**:
1. Retrieves token address for the trade deal
2. Calls token's participant verification function
3. Returns false if token doesn't exist

**Example Usage**:
```solidity
uint256 tradeDealId = 1;
address userAddress = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;

bool isParticipant = ICollateralTokenFactory(diamond).isCollateralTokenParticipant(
    tradeDealId,
    userAddress
);

if (isParticipant) {
    console.log("User is verified participant");
    // Allow token operations
} else {
    console.log("User is not a participant");
    // Restrict token operations
}
```

### Administrative Functions

#### `setCollateralTokenRestrictions()`
```solidity
function setCollateralTokenRestrictions(uint256 _tradeDealId, bool _restrictionsEnabled) external onlyOwner
```

**Purpose**: Enables or disables transfer restrictions for a collateral token.

**Parameters**:
- `_tradeDealId` (uint256): ID of the trade deal
- `_restrictionsEnabled` (bool): Whether to enforce restrictions

**Access Control**: Owner only

**Process**:
1. Retrieves token address for the trade deal
2. Validates token exists
3. Calls token's restriction configuration function
4. Updates restriction status

**Example Usage**:
```solidity
uint256 tradeDealId = 1;

// Enable restrictions (only participants can transfer)
ICollateralTokenFactory(diamond).setCollateralTokenRestrictions(tradeDealId, true);
console.log("Restrictions enabled for trade deal:", tradeDealId);

// Later, disable restrictions (open transfers)
ICollateralTokenFactory(diamond).setCollateralTokenRestrictions(tradeDealId, false);
console.log("Restrictions disabled for trade deal:", tradeDealId);
```

**Use Cases**:
- **Enable Restrictions**: During active trade deal phase
- **Disable Restrictions**: After trade deal completion
- **Compliance Requirements**: Based on regulatory needs
- **Emergency Controls**: For security incidents

## Integration Examples

### Complete Trade Deal Token Lifecycle
```solidity
// Comprehensive trade deal collateral token management
contract TradeDealTokenManager {
    struct TokenInfo {
        uint256 tradeDealId;
        address tokenAddress;
        string name;
        string symbol;
        bool restrictionsEnabled;
        uint256 totalSupply;
        uint256 participantCount;
    }
    
    mapping(uint256 => TokenInfo) public tradeDealTokens;
    mapping(address => uint256[]) public userTradeDealIds;
    
    function createTradeDealWithToken(
        uint256 tradeDealId,
        string memory dealName,
        string memory tokenSymbol,
        bool initialRestrictions
    ) external onlyOwner returns (address tokenAddress) {
        // Create collateral token for trade deal
        string memory tokenName = string(abi.encodePacked(dealName, " Collateral"));
        
        tokenAddress = ICollateralTokenFactory(diamond).createCollateralToken(
            tradeDealId,
            tokenName,
            tokenSymbol,
            initialRestrictions
        );
        
        // Store token information
        tradeDealTokens[tradeDealId] = TokenInfo({
            tradeDealId: tradeDealId,
            tokenAddress: tokenAddress,
            name: tokenName,
            symbol: tokenSymbol,
            restrictionsEnabled: initialRestrictions,
            totalSupply: 0,
            participantCount: 0
        });
        
        emit TradeDealTokenCreated(tradeDealId, tokenAddress, tokenName, tokenSymbol);
    }
    
    function addParticipantToTradeDeal(
        uint256 tradeDealId,
        address participant
    ) external onlyOwner {
        TokenInfo storage tokenInfo = tradeDealTokens[tradeDealId];
        require(tokenInfo.tokenAddress != address(0), "Trade deal token not found");
        
        // Add participant to trade deal (this would integrate with TradeDealAdminFacet)
        // ITradeDealAdmin(diamond).addTradeDealParticipant(tradeDealId, participant);
        
        // Track user's trade deal participation
        userTradeDealIds[participant].push(tradeDealId);
        tokenInfo.participantCount++;
        
        emit ParticipantAdded(tradeDealId, participant);
    }
    
    function updateTokenRestrictions(
        uint256 tradeDealId,
        bool restrictionsEnabled
    ) external onlyOwner {
        TokenInfo storage tokenInfo = tradeDealTokens[tradeDealId];
        require(tokenInfo.tokenAddress != address(0), "Trade deal token not found");
        
        // Update restrictions through factory facet
        ICollateralTokenFactory(diamond).setCollateralTokenRestrictions(
            tradeDealId,
            restrictionsEnabled
        );
        
        // Update local tracking
        tokenInfo.restrictionsEnabled = restrictionsEnabled;
        
        emit TokenRestrictionsUpdated(tradeDealId, restrictionsEnabled);
    }
    
    function getTokenInfo(uint256 tradeDealId) external view returns (TokenInfo memory) {
        return tradeDealTokens[tradeDealId];
    }
    
    function getUserTradeDealTokens(address user) external view returns (
        uint256[] memory tradeDealIds,
        address[] memory tokenAddresses,
        uint256[] memory balances
    ) {
        uint256[] memory userDeals = userTradeDealIds[user];
        
        tradeDealIds = new uint256[](userDeals.length);
        tokenAddresses = new address[](userDeals.length);
        balances = new uint256[](userDeals.length);
        
        for (uint256 i = 0; i < userDeals.length; i++) {
            uint256 tradeDealId = userDeals[i];
            TokenInfo memory tokenInfo = tradeDealTokens[tradeDealId];
            
            tradeDealIds[i] = tradeDealId;
            tokenAddresses[i] = tokenInfo.tokenAddress;
            
            if (tokenInfo.tokenAddress != address(0)) {
                balances[i] = IERC20(tokenInfo.tokenAddress).balanceOf(user);
            }
        }
    }
}
```

### Collateral Token Distribution System
```solidity
// System for distributing collateral tokens to participants
contract CollateralTokenDistributor {
    struct DistributionRecord {
        uint256 tradeDealId;
        address recipient;
        uint256 amount;
        uint256 timestamp;
        string reason;
    }
    
    mapping(uint256 => DistributionRecord[]) public distributionHistory;
    mapping(uint256 => mapping(address => uint256)) public totalDistributed;
    
    function distributeCollateralTokens(
        uint256 tradeDealId,
        address[] memory recipients,
        uint256[] memory amounts,
        string memory reason
    ) external onlyOwner {
        require(recipients.length == amounts.length, "Array length mismatch");
        
        // Get token address
        address tokenAddress = ICollateralTokenFactory(diamond).getCollateralTokenAddress(tradeDealId);
        require(tokenAddress != address(0), "Token not found for trade deal");
        
        RestrictedCollateralToken token = RestrictedCollateralToken(tokenAddress);
        
        for (uint256 i = 0; i < recipients.length; i++) {
            address recipient = recipients[i];
            uint256 amount = amounts[i];
            
            // Verify recipient is a participant
            bool isParticipant = ICollateralTokenFactory(diamond).isCollateralTokenParticipant(
                tradeDealId,
                recipient
            );
            require(isParticipant, "Recipient is not a trade deal participant");
            
            // Mint tokens to recipient
            token.mint(recipient, amount);
            
            // Record distribution
            distributionHistory[tradeDealId].push(DistributionRecord({
                tradeDealId: tradeDealId,
                recipient: recipient,
                amount: amount,
                timestamp: block.timestamp,
                reason: reason
            }));
            
            totalDistributed[tradeDealId][recipient] += amount;
            
            emit CollateralTokensDistributed(tradeDealId, recipient, amount, reason);
        }
    }
    
    function batchDistributeEqual(
        uint256 tradeDealId,
        address[] memory recipients,
        uint256 totalAmount,
        string memory reason
    ) external onlyOwner {
        require(recipients.length > 0, "No recipients provided");
        
        uint256 amountPerRecipient = totalAmount / recipients.length;
        uint256[] memory amounts = new uint256[](recipients.length);
        
        for (uint256 i = 0; i < recipients.length; i++) {
            amounts[i] = amountPerRecipient;
        }
        
        distributeCollateralTokens(tradeDealId, recipients, amounts, reason);
    }
    
    function getDistributionHistory(uint256 tradeDealId) external view returns (DistributionRecord[] memory) {
        return distributionHistory[tradeDealId];
    }
    
    function getTotalDistributed(uint256 tradeDealId, address recipient) external view returns (uint256) {
        return totalDistributed[tradeDealId][recipient];
    }
}
```

### Token Restriction Manager
```solidity
// Advanced restriction management for collateral tokens
contract TokenRestrictionManager {
    struct RestrictionPolicy {
        bool transfersEnabled;
        bool onlyParticipants;
        uint256 maxTransferAmount;
        uint256 cooldownPeriod;
        bool emergencyFreeze;
    }
    
    mapping(uint256 => RestrictionPolicy) public restrictionPolicies;
    mapping(uint256 => mapping(address => uint256)) public lastTransferTime;
    
    function setRestrictionPolicy(
        uint256 tradeDealId,
        RestrictionPolicy memory policy
    ) external onlyOwner {
        restrictionPolicies[tradeDealId] = policy;
        
        // Apply basic restrictions through factory facet
        ICollateralTokenFactory(diamond).setCollateralTokenRestrictions(
            tradeDealId,
            policy.onlyParticipants || !policy.transfersEnabled
        );
        
        emit RestrictionPolicyUpdated(tradeDealId, policy);
    }
    
    function validateTransfer(
        uint256 tradeDealId,
        address from,
        address to,
        uint256 amount
    ) external view returns (bool allowed, string memory reason) {
        RestrictionPolicy memory policy = restrictionPolicies[tradeDealId];
        
        // Check emergency freeze
        if (policy.emergencyFreeze) {
            return (false, "Emergency freeze active");
        }
        
        // Check if transfers are enabled
        if (!policy.transfersEnabled) {
            return (false, "Transfers disabled");
        }
        
        // Check participant requirement
        if (policy.onlyParticipants) {
            bool fromIsParticipant = ICollateralTokenFactory(diamond).isCollateralTokenParticipant(tradeDealId, from);
            bool toIsParticipant = ICollateralTokenFactory(diamond).isCollateralTokenParticipant(tradeDealId, to);
            
            if (!fromIsParticipant || !toIsParticipant) {
                return (false, "Both parties must be participants");
            }
        }
        
        // Check transfer amount limit
        if (policy.maxTransferAmount > 0 && amount > policy.maxTransferAmount) {
            return (false, "Amount exceeds maximum transfer limit");
        }
        
        // Check cooldown period
        if (policy.cooldownPeriod > 0) {
            uint256 timeSinceLastTransfer = block.timestamp - lastTransferTime[tradeDealId][from];
            if (timeSinceLastTransfer < policy.cooldownPeriod) {
                return (false, "Cooldown period not elapsed");
            }
        }
        
        return (true, "Transfer allowed");
    }
    
    function emergencyFreeze(uint256 tradeDealId, bool freeze) external onlyOwner {
        restrictionPolicies[tradeDealId].emergencyFreeze = freeze;
        
        // Apply emergency restrictions
        ICollateralTokenFactory(diamond).setCollateralTokenRestrictions(tradeDealId, freeze);
        
        emit EmergencyFreezeUpdated(tradeDealId, freeze);
    }
    
    function updateTransferTime(uint256 tradeDealId, address user) external onlyAuthorized {
        lastTransferTime[tradeDealId][user] = block.timestamp;
    }
}
```

### Factory Analytics and Monitoring
```solidity
// Analytics and monitoring for collateral token factory
contract CollateralTokenAnalytics {
    struct TokenMetrics {
        uint256 tradeDealId;
        address tokenAddress;
        uint256 totalSupply;
        uint256 holderCount;
        uint256 transferCount;
        uint256 creationTime;
        bool restrictionsActive;
    }
    
    mapping(uint256 => TokenMetrics) public tokenMetrics;
    mapping(address => uint256) public tokenToTradeDeal;
    uint256[] public allTradeDealIds;
    
    function trackTokenCreation(
        uint256 tradeDealId,
        address tokenAddress,
        string memory name,
        string memory symbol
    ) external onlyAuthorized {
        tokenMetrics[tradeDealId] = TokenMetrics({
            tradeDealId: tradeDealId,
            tokenAddress: tokenAddress,
            totalSupply: 0,
            holderCount: 0,
            transferCount: 0,
            creationTime: block.timestamp,
            restrictionsActive: true
        });
        
        tokenToTradeDeal[tokenAddress] = tradeDealId;
        allTradeDealIds.push(tradeDealId);
        
        emit TokenCreationTracked(tradeDealId, tokenAddress, name, symbol);
    }
    
    function updateTokenMetrics(uint256 tradeDealId) external {
        TokenMetrics storage metrics = tokenMetrics[tradeDealId];
        require(metrics.tokenAddress != address(0), "Token not found");
        
        IERC20 token = IERC20(metrics.tokenAddress);
        metrics.totalSupply = token.totalSupply();
        
        // Update restriction status
        // This would require additional interface methods on RestrictedCollateralToken
        
        emit TokenMetricsUpdated(tradeDealId, metrics.totalSupply, metrics.holderCount);
    }
    
    function getFactoryOverview() external view returns (
        uint256 totalTokens,
        uint256 totalSupplyAcrossTokens,
        uint256 averageHoldersPerToken,
        uint256 tokensWithRestrictions
    ) {
        totalTokens = allTradeDealIds.length;
        uint256 totalHolders = 0;
        
        for (uint256 i = 0; i < allTradeDealIds.length; i++) {
            TokenMetrics memory metrics = tokenMetrics[allTradeDealIds[i]];
            totalSupplyAcrossTokens += metrics.totalSupply;
            totalHolders += metrics.holderCount;
            
            if (metrics.restrictionsActive) {
                tokensWithRestrictions++;
            }
        }
        
        if (totalTokens > 0) {
            averageHoldersPerToken = totalHolders / totalTokens;
        }
    }
    
    function getTokensByRestrictionStatus(bool restrictionsActive) external view returns (uint256[] memory) {
        uint256[] memory matchingTokens = new uint256[](allTradeDealIds.length);
        uint256 count = 0;
        
        for (uint256 i = 0; i < allTradeDealIds.length; i++) {
            if (tokenMetrics[allTradeDealIds[i]].restrictionsActive == restrictionsActive) {
                matchingTokens[count] = allTradeDealIds[i];
                count++;
            }
        }
        
        // Resize array to actual count
        uint256[] memory result = new uint256[](count);
        for (uint256 i = 0; i < count; i++) {
            result[i] = matchingTokens[i];
        }
        
        return result;
    }
}
```

## Events

### Factory Events
```solidity
event CollateralTokenFactoryInitialized(address indexed factoryAddress);
```

### Token Management Events
```solidity
event CollateralTokenCreated(uint256 indexed tradeDealId, address indexed tokenAddress, string name, string symbol);
```

## Security Considerations

### Factory Security
- Factory initialization protection against re-initialization
- Owner-only access for token creation and configuration
- Validation of trade deal uniqueness
- Proper factory address management

### Token Security
- Transfer restrictions based on participant verification
- Integration with identity verification systems
- Emergency controls for restriction management
- Proper access control for administrative functions

### Integration Security
- Validation of trade deal existence before token creation
- Proper participant verification before token operations
- Secure storage of factory and token addresses
- Event logging for audit trails

## Gas Optimization

### Factory Efficiency
- Minimal proxy pattern for token deployment
- Efficient storage layout for mappings
- Batch operations where possible
- Optimized factory initialization

### Token Operations
- Gas-efficient participant verification
- Optimized restriction checking
- Minimal storage operations
- Efficient event emission

## Error Handling

### Common Errors
- `"factory not set"` - Factory not initialized
- `"token already exists for this trade deal"` - Duplicate token creation
- `"token does not exist for this trade deal"` - Invalid token reference
- Access control failures for unauthorized operations

### Best Practices
- Always check factory initialization before operations
- Validate trade deal existence before token creation
- Verify participant status before token operations
- Handle zero address returns gracefully

## Testing Considerations

### Unit Tests
- Factory initialization and configuration
- Token creation and management
- Restriction setting and validation
- Participant verification
- Access control enforcement

### Integration Tests
- Trade deal integration workflows
- Participant management integration
- Token distribution scenarios
- Restriction policy enforcement

## Related Documentation

- [RestrictedCollateralToken](../tokens/restricted-collateral-token.md) - Token implementation
- [CollateralTokenFactory](../tokens/collateral-token-factory.md) - Factory contract
- [TradeDealAdminFacet](./trade-deal-admin-facet.md) - Trade deal administration
- [TradeDealOperationsFacet](./trade-deal-operations-facet.md) - Trade deal operations
- [IdentityRegistryFacet](./identity-registry-facet.md) - Participant verification
- [Token Factory Guide](../../guides/token-factory.md) - Implementation guide

---

*This facet provides factory functionality for creating and managing restricted collateral tokens within the Gemforce platform, enabling secure and compliant token distribution for trade deal participants.*