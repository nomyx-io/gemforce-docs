# TradeDealLib Library

## Overview

The [`TradeDealLib`](../../smart-contracts/libraries/trade-deal-lib.md) library provides core utilities and data structures for managing collateralized trade deals within the Gemforce platform. This library implements the Diamond Standard storage pattern and provides essential functions for trade deal creation, management, participant handling, and financial operations including funding, repayment, and collateral management.

## Key Features

- **Diamond Storage Pattern**: Secure storage isolation using Diamond Standard
- **Trade Deal Lifecycle Management**: Complete CRUD operations for trade deals
- **Role-Based Access Control**: Flexible permission system with multiple operation modes
- **Collateral Management**: Invoice NFT collateral handling and redemption
- **Financial Operations**: USDC funding, withdrawal, and repayment tracking
- **Interest Distribution**: Automated interest calculation and distribution
- **Multi-Mode Operations**: Support for centralized, self-service, and hybrid models

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library TradeDealLib {
    // Enums
    enum OperationMode { CENTRALIZED, SELF_SERVICE, HYBRID, CUSTOM }
    enum Role { NONE, ADMIN, LENDER, BORROWER, UNDERWRITER, LIQUIDATOR }
    
    // Permission constants
    uint256 constant PERMISSION_DEPOSIT_FUNDS = 1;
    uint256 constant PERMISSION_WITHDRAW_FUNDS = 2;
    uint256 constant PERMISSION_DEPOSIT_COLLATERAL = 4;
    uint256 constant PERMISSION_WITHDRAW_COLLATERAL = 8;
    uint256 constant PERMISSION_DISTRIBUTE_INTEREST = 16;
    
    // Core data structures
    struct TradeDeal { ... }
    struct TradeDealStorage { ... }
    struct CreateTradeDealParams { ... }
    struct CreateTradeDealResult { ... }
    
    // Core functions
    function _createTradeDeal(CreateTradeDealParams memory params) internal returns (CreateTradeDealResult memory);
    function _updateTradeDeal(...) internal returns (UpdateTradeDealResult memory);
    function _activateTradeDeal(uint256 tradeDealId) internal returns (TradeDealStateChangeResult memory);
    function _deactivateTradeDeal(uint256 tradeDealId) internal returns (TradeDealStateChangeResult memory);
}
```

## Data Structures

### OperationMode Enum
```solidity
enum OperationMode {
    CENTRALIZED,    // Contract owner manages all fund operations
    SELF_SERVICE,   // Borrowers can directly withdraw/repay funds
    HYBRID,         // Mixed model with configurable permissions
    CUSTOM          // Fine-grained permission configuration
}
```

**Purpose**: Defines the operational model for trade deal management.

**Values**:
- `CENTRALIZED` (0): All operations controlled by contract owner/admin
- `SELF_SERVICE` (1): Borrowers have direct access to fund operations
- `HYBRID` (2): Mixed model with some automated and some manual operations
- `CUSTOM` (3): Fine-grained permission configuration per user

### Role Enum
```solidity
enum Role {
    NONE,        // No special role (default)
    ADMIN,       // Full control over the trade deal
    LENDER,      // Can deposit funds and receive Collateral tokens
    BORROWER,    // Can deposit invoices and withdraw funds (in self-service)
    UNDERWRITER, // Can approve/reject deals, modify terms
    LIQUIDATOR   // Can liquidate collateral if terms are violated
}
```

**Purpose**: Defines user roles within the trade deal system.

### Permission Constants
```solidity
uint256 constant PERMISSION_DEPOSIT_FUNDS = 1;
uint256 constant PERMISSION_WITHDRAW_FUNDS = 2;
uint256 constant PERMISSION_DEPOSIT_COLLATERAL = 4;
uint256 constant PERMISSION_WITHDRAW_COLLATERAL = 8;
uint256 constant PERMISSION_DISTRIBUTE_INTEREST = 16;
```

**Purpose**: Bit flags for fine-grained permission control.

### TradeDeal Struct
```solidity
struct TradeDeal {
    uint256 id;
    string name;
    string symbol;                      // Symbol for the trade deal Collateral token
    uint256 interestRate;
    uint256 collateralToInterestRatio;
    bool active;
    uint256[] requiredClaimTopics;      // Claim topics required for participation
    address collateralAddress;          // Address of the Collateral token contract
    address interestAddress;            // Address of the VABI token contract
    address usdcAddress;               // Address of the USDC token contract
    OperationMode operationMode;       // Operation mode for the trade deal
}
```

**Purpose**: Core data structure representing a trade deal.

**Key Fields**:
- `id`: Unique identifier for the trade deal
- `name`: Human-readable name for the trade deal
- `symbol`: Symbol used for the associated Collateral token
- `interestRate`: Interest rate for the trade deal (basis points)
- `collateralToInterestRatio`: Ratio of collateral to interest tokens
- `requiredClaimTopics`: Array of claim topics required for participation
- `operationMode`: Operational model for the trade deal

### TradeDealStorage Struct
```solidity
struct TradeDealStorage {
    // Trade deal tracking
    mapping(uint256 => TradeDeal) tradeDeals;
    uint256[] tradeDealIds;
    uint256 nextTradeDealId;
    
    // Per-trade deal mappings
    mapping(uint256 => uint256[]) tradeDealInvoices;
    mapping(uint256 => uint256) tradeDealUsdcBalances;
    mapping(uint256 => mapping(address => bool)) tradeDealParticipants;
    mapping(uint256 => uint256[]) tradeDealRequiredClaimTopics;
    
    // Role-based access control
    mapping(uint256 => mapping(address => Role)) userRoles;
    mapping(uint256 => mapping(address => uint256)) userPermissions;
    
    // Enhanced functionality
    mapping(uint256 => uint256) tradeDealFundingTargets;
    mapping(uint256 => bool) tradeDealFundingWithdrawn;
    mapping(uint256 => uint256) tradeDealRepaidAmounts;
    mapping(uint256 => uint256) tradeDealTotalDebt;
    mapping(uint256 => uint256) tradeDealTotalWithdrawn;
    mapping(uint256 => mapping(uint256 => address)) invoiceDepositors;
}
```

**Purpose**: Diamond storage structure for all trade deal data.

## Core Functions

### Trade Deal Creation

#### `_createTradeDeal()`
```solidity
function _createTradeDeal(CreateTradeDealParams memory params) internal returns (CreateTradeDealResult memory)
```

**Purpose**: Create a new trade deal with specified parameters.

**Parameters**:
- `params` (CreateTradeDealParams): Struct containing all creation parameters

**CreateTradeDealParams Structure**:
```solidity
struct CreateTradeDealParams {
    string name;
    string symbol;
    uint256 interestRate;
    uint256 collateralToInterestRatio;
    uint256[] requiredClaimTopics;
    address collateralAddress;
    address interestAddress;
    address usdcAddress;
    OperationMode operationMode;
}
```

**Returns**: `CreateTradeDealResult` struct with trade deal information

**Key Operations**:
- Assigns unique trade deal ID
- Initializes trade deal data structure
- Creates or assigns collateral token contract
- Sets up required claim topics
- Initializes financial tracking (funding targets, repayment amounts)

**Example Usage**:
```solidity
// Create a new trade deal
TradeDealLib.CreateTradeDealParams memory params = TradeDealLib.CreateTradeDealParams({
    name: "Invoice Financing Deal #1",
    symbol: "IFD1",
    interestRate: 1200, // 12% APR
    collateralToInterestRatio: 1000000, // 1:1 ratio
    requiredClaimTopics: new uint256[](1),
    collateralAddress: address(0), // Will create new token
    interestAddress: interestTokenAddress,
    usdcAddress: usdcTokenAddress,
    operationMode: TradeDealLib.OperationMode.SELF_SERVICE
});
params.requiredClaimTopics[0] = 1; // KYC required

TradeDealLib.CreateTradeDealResult memory result = TradeDealLib._createTradeDeal(params);
```

### Trade Deal Management

#### `_updateTradeDeal()`
```solidity
function _updateTradeDeal(
    uint256 tradeDealId,
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
) internal returns (UpdateTradeDealResult memory)
```

**Purpose**: Update an existing trade deal's parameters.

**Parameters**:
- `tradeDealId`: ID of the trade deal to update
- Various parameters to update (name, symbol, rates, addresses)

**Requirements**:
- Trade deal must exist
- Caller must have appropriate permissions

**Example Usage**:
```solidity
// Update trade deal interest rate
TradeDealLib.UpdateTradeDealResult memory result = TradeDealLib._updateTradeDeal(
    tradeDealId,
    "Updated Invoice Financing Deal #1",
    "IFD1",
    1500, // Updated to 15% APR
    1000000,
    collateralAddress,
    interestAddress,
    usdcAddress
);
```

#### `_activateTradeDeal()` / `_deactivateTradeDeal()`
```solidity
function _activateTradeDeal(uint256 tradeDealId) internal returns (TradeDealStateChangeResult memory)
function _deactivateTradeDeal(uint256 tradeDealId) internal returns (TradeDealStateChangeResult memory)
```

**Purpose**: Activate or deactivate a trade deal.

**Parameters**:
- `tradeDealId`: ID of the trade deal to activate/deactivate

**Requirements**:
- Trade deal must exist
- Appropriate permissions required

**Example Usage**:
```solidity
// Activate a trade deal
TradeDealLib.TradeDealStateChangeResult memory result = TradeDealLib._activateTradeDeal(tradeDealId);

// Deactivate a trade deal
TradeDealLib.TradeDealStateChangeResult memory result = TradeDealLib._deactivateTradeDeal(tradeDealId);
```

## Integration Examples

### Trade Deal Factory Contract
```solidity
// Factory contract for creating and managing trade deals
contract TradeDealFactory {
    using TradeDealLib for TradeDealLib.TradeDealStorage;
    
    event TradeDealCreated(
        uint256 indexed tradeDealId,
        string name,
        address indexed creator,
        TradeDealLib.OperationMode operationMode
    );
    
    event TradeDealUpdated(uint256 indexed tradeDealId, string name);
    
    function createStandardTradeDeal(
        string memory name,
        string memory symbol,
        uint256 interestRate,
        address interestTokenAddress,
        address usdcTokenAddress
    ) external returns (uint256 tradeDealId) {
        // Prepare creation parameters
        TradeDealLib.CreateTradeDealParams memory params = TradeDealLib.CreateTradeDealParams({
            name: name,
            symbol: symbol,
            interestRate: interestRate,
            collateralToInterestRatio: 1000000, // 1:1 ratio
            requiredClaimTopics: new uint256[](1),
            collateralAddress: address(0), // Will create new token
            interestAddress: interestTokenAddress,
            usdcAddress: usdcTokenAddress,
            operationMode: TradeDealLib.OperationMode.SELF_SERVICE
        });
        
        // Set KYC requirement
        params.requiredClaimTopics[0] = 1;
        
        // Create the trade deal
        TradeDealLib.CreateTradeDealResult memory result = TradeDealLib._createTradeDeal(params);
        
        emit TradeDealCreated(
            result.tradeDealId,
            result.name,
            msg.sender,
            result.operationMode
        );
        
        return result.tradeDealId;
    }
    
    function createCustomTradeDeal(
        TradeDealLib.CreateTradeDealParams memory params
    ) external returns (uint256 tradeDealId) {
        require(bytes(params.name).length > 0, "Name cannot be empty");
        require(params.interestRate > 0, "Interest rate must be positive");
        
        TradeDealLib.CreateTradeDealResult memory result = TradeDealLib._createTradeDeal(params);
        
        emit TradeDealCreated(
            result.tradeDealId,
            result.name,
            msg.sender,
            result.operationMode
        );
        
        return result.tradeDealId;
    }
    
    function updateTradeDeal(
        uint256 tradeDealId,
        string memory name,
        string memory symbol,
        uint256 interestRate
    ) external onlyTradeDealAdmin(tradeDealId) {
        // Get current trade deal info
        TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(tradeDealId);
        
        TradeDealLib.UpdateTradeDealResult memory result = TradeDealLib._updateTradeDeal(
            tradeDealId,
            name,
            symbol,
            interestRate,
            tradeDeal.collateralToInterestRatio,
            tradeDeal.collateralAddress,
            tradeDeal.interestAddress,
            tradeDeal.usdcAddress
        );
        
        emit TradeDealUpdated(tradeDealId, name);
    }
    
    function getTradeDeal(uint256 tradeDealId) public view returns (TradeDealLib.TradeDeal memory) {
        // Implementation would access storage and return trade deal
        // This is a simplified example
    }
    
    modifier onlyTradeDealAdmin(uint256 tradeDealId) {
        // Implementation would check if caller has admin role for the trade deal
        _;
    }
}
```

### Trade Deal Analytics System
```solidity
// Analytics system for trade deal performance tracking
contract TradeDealAnalytics {
    using TradeDealLib for TradeDealLib.TradeDealStorage;
    
    struct TradeDealMetrics {
        uint256 totalFunded;
        uint256 totalRepaid;
        uint256 totalInterestEarned;
        uint256 averageInterestRate;
        uint256 defaultRate;
        uint256 averageFundingTime;
    }
    
    struct TradeDealPerformance {
        uint256 tradeDealId;
        uint256 fundingTarget;
        uint256 currentFunding;
        uint256 repaidAmount;
        uint256 outstandingDebt;
        uint256 daysActive;
        bool fullyFunded;
        bool fullyRepaid;
        TradeDealLib.OperationMode operationMode;
    }
    
    mapping(uint256 => uint256) public tradeDealCreationTime;
    mapping(uint256 => uint256) public tradeDealFundingTime;
    mapping(uint256 => uint256) public tradeDealRepaymentTime;
    
    event MetricsUpdated(TradeDealMetrics metrics);
    event PerformanceCalculated(uint256 indexed tradeDealId, TradeDealPerformance performance);
    
    function calculateTradeDealPerformance(
        uint256 tradeDealId
    ) external view returns (TradeDealPerformance memory performance) {
        // Get trade deal information
        TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(tradeDealId);
        
        performance.tradeDealId = tradeDealId;
        performance.operationMode = tradeDeal.operationMode;
        
        // Get financial metrics from storage
        performance.fundingTarget = getFundingTarget(tradeDealId);
        performance.currentFunding = getCurrentFunding(tradeDealId);
        performance.repaidAmount = getRepaidAmount(tradeDealId);
        performance.outstandingDebt = getOutstandingDebt(tradeDealId);
        
        // Calculate status
        performance.fullyFunded = performance.currentFunding >= performance.fundingTarget;
        performance.fullyRepaid = performance.repaidAmount >= performance.outstandingDebt;
        
        // Calculate days active
        uint256 creationTime = tradeDealCreationTime[tradeDealId];
        if (creationTime > 0) {
            performance.daysActive = (block.timestamp - creationTime) / 1 days;
        }
    }
    
    function calculateOverallMetrics() external returns (TradeDealMetrics memory metrics) {
        uint256[] memory allTradeDealIds = getAllTradeDealIds();
        
        uint256 totalDeals = allTradeDealIds.length;
        uint256 totalInterestRateSum = 0;
        uint256 defaultedDeals = 0;
        uint256 totalFundingTimeSum = 0;
        uint256 fundedDealsCount = 0;
        
        for (uint256 i = 0; i < totalDeals; i++) {
            uint256 tradeDealId = allTradeDealIds[i];
            TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(tradeDealId);
            
            // Accumulate metrics
            totalInterestRateSum += tradeDeal.interestRate;
            metrics.totalFunded += getCurrentFunding(tradeDealId);
            metrics.totalRepaid += getRepaidAmount(tradeDealId);
            
            // Calculate funding time if deal is funded
            uint256 creationTime = tradeDealCreationTime[tradeDealId];
            uint256 fundingTime = tradeDealFundingTime[tradeDealId];
            
            if (fundingTime > creationTime) {
                totalFundingTimeSum += (fundingTime - creationTime);
                fundedDealsCount++;
            }
            
            // Check for defaults (simplified logic)
            if (isDefaulted(tradeDealId)) {
                defaultedDeals++;
            }
        }
        
        // Calculate averages
        if (totalDeals > 0) {
            metrics.averageInterestRate = totalInterestRateSum / totalDeals;
            metrics.defaultRate = (defaultedDeals * 10000) / totalDeals; // Basis points
        }
        
        if (fundedDealsCount > 0) {
            metrics.averageFundingTime = totalFundingTimeSum / fundedDealsCount;
        }
        
        metrics.totalInterestEarned = metrics.totalRepaid > metrics.totalFunded 
            ? metrics.totalRepaid - metrics.totalFunded 
            : 0;
        
        emit MetricsUpdated(metrics);
        return metrics;
    }
    
    function getTradeDealsByOperationMode(
        TradeDealLib.OperationMode mode
    ) external view returns (uint256[] memory tradeDealIds) {
        uint256[] memory allIds = getAllTradeDealIds();
        uint256 count = 0;
        
        // Count matching trade deals
        for (uint256 i = 0; i < allIds.length; i++) {
            TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(allIds[i]);
            if (tradeDeal.operationMode == mode) {
                count++;
            }
        }
        
        tradeDealIds = new uint256[](count);
        uint256 index = 0;
        
        for (uint256 i = 0; i < allIds.length; i++) {
            TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(allIds[i]);
            if (tradeDeal.operationMode == mode) {
                tradeDealIds[index] = allIds[i];
                index++;
            }
        }
        return tradeDealIds;
    }
    
    function getTradeDeal(uint256 tradeDealId) internal view returns (TradeDealLib.TradeDeal memory) {
        // Placeholder, assume external access to TradeDeal data
        return TradeDealLib.TradeDeal(tradeDealId, "", "", 0, 0, false, new uint256[](0), address(0), address(0), address(0), TradeDealLib.OperationMode.CENTRALIZED);
    }
    
    function getFundingTarget(uint256 tradeDealId) internal view returns (uint256) { return 0; }
    function getCurrentFunding(uint256 tradeDealId) internal view returns (uint256) { return 0; }
    function getRepaidAmount(uint256 tradeDealId) internal view returns (uint256) { return 0; }
    function getOutstandingDebt(uint256 tradeDealId) internal view returns (uint256) { return 0; }
    function getAllTradeDealIds() internal view returns (uint256[] memory) { return new uint256[](0); }
    function isDefaulted(uint256 tradeDealId) internal view returns (bool) { return false; }
}
```

### Role-Based Access Control Manager
```solidity
// Manages roles and permissions for trade deal participants
contract AccessControlManager {
    using TradeDealLib for TradeDealLib.TradeDealStorage;
    
    event ParticipantRoleSet(uint256 indexed tradeDealId, address indexed participant, TradeDealLib.Role role);
    event ParticipantPermissionsSet(uint256 indexed tradeDealId, address indexed participant, uint256 permissions);
    
    function setParticipantRole(
        uint256 tradeDealId,
        address participant,
        TradeDealLib.Role role
    ) external onlyOwner {
        TradeDealLib.TradeDealStorage storage tds = TradeDealLib.tradeDealStorage();
        tds.userRoles[tradeDealId][participant] = role;
        
        emit ParticipantRoleSet(tradeDealId, participant, role);
    }
    
    function setParticipantPermissions(
        uint256 tradeDealId,
        address participant,
        uint256 permissions
    ) external onlyOwner {
        TradeDealLib.TradeDealStorage storage tds = TradeDealLib.tradeDealStorage();
        tds.userPermissions[tradeDealId][participant] = permissions;
        
        emit ParticipantPermissionsSet(tradeDealId, participant, permissions);
    }
    
    function hasPermission(
        uint256 tradeDealId,
        address participant,
        uint256 permission
    ) public view returns (bool) {
        TradeDealLib.TradeDealStorage storage tds = TradeDealLib.tradeDealStorage();
        uint256 currentPermissions = tds.userPermissions[tradeDealId][participant];
        return (currentPermissions & permission) == permission;
    }
    
    function getParticipantRole(
        uint256 tradeDealId,
        address participant
    ) public view returns (TradeDealLib.Role) {
        TradeDealLib.TradeDealStorage storage tds = TradeDealLib.tradeDealStorage();
        return tds.userRoles[tradeDealId][participant];
    }
    
    modifier onlyOwner() {
        // Placeholder for contract ownership check
        _;
    }
}
```

## Events

### Core Trade Deal Events
- `TradeDealCreated(uint256 indexed tradeDealId, string name, string symbol, uint256 interestRate, uint256 collateralToInterestRatio, bool active, address nftAddress, address collateralAddress, address interestAddress, address usdcAddress, TradeDealLib.OperationMode operationMode)`: Emitted when a new trade deal is created.
- `TradeDealUpdated(uint256 indexed tradeDealId, string name, string symbol, uint256 interestRate, uint256 collateralToInterestRatio, bool active, address collateralAddress, address interestAddress, address usdcAddress)`: Emitted when a trade deal's parameters are updated.
- `TradeDealActivated(uint256 indexed tradeDealId)`: Emitted when a trade deal is activated.
- `TradeDealDeactivated(uint256 indexed tradeDealId)`: Emitted when a trade deal is deactivated.

### Participant Management Events
- `ParticipantAdded(uint256 indexed tradeDealId, address indexed participant, TradeDealLib.Role role)`: Emitted when a participant is added to a trade deal.
- `ParticipantRemoved(uint256 indexed tradeDealId, address indexed participant)`: Emitted when a participant is removed.
- `ParticipantRoleSet(uint256 indexed tradeDealId, address indexed participant, TradeDealLib.Role role)`: Emitted when a participant's role is set.

### Financial Operation Events
- `FundsDeposited(uint256 indexed tradeDealId, address indexed depositor, uint256 amount)`: Emitted when funds are deposited to a trade deal.
- `FundsWithdrawn(uint256 indexed tradeDealId, address indexed withdrawer, uint256 amount)`: Emitted when funds are withdrawn.
- `CollateralDeposited(uint256 indexed tradeDealId, uint256 indexed invoiceId, address indexed depositor)`: Emitted when collateral (invoice NFT) is deposited.
- `CollateralWithdrawn(uint256 indexed tradeDealId, uint256 indexed invoiceId, address indexed withdrawer)`: Emitted when collateral is withdrawn.

### Enhanced Financial Events
- `InterestDistributed(uint256 indexed tradeDealId, uint256 amount)`: Emitted when interest is distributed to lenders.
- `PrincipalRepaid(uint256 indexed tradeDealId, uint256 amount)`: Emitted when principal is repaid by borrower.
- `TradeDealDefaulted(uint256 indexed tradeDealId)`: Emitted when a trade deal goes into default.

## Security Considerations

### Access Control Security
- **Role-Based Permissions**: Use the `userRoles` and `userPermissions` to strictly control who can perform each operation.
- **Ownership Checks**: Critical functions (e.g., `_createTradeDeal`, `_updateTradeDeal`) should have strong ownership or admin checks.
- **Claim Topic Validation**: Ensure `requiredClaimTopics` logic prevents unauthorized participants.

### Financial Security
- **Reentrancy Protection**: Implement `nonReentrant` where funds are handled.
- **Sufficient Balance**: Always check for sufficient token balances before transfers.
- **Overflow/Underflow**: Use SafeMath or Solidity 0.8+ for all arithmetic.

### Storage Security
- **Diamond Storage**: Protects against storage collisions between facets.
- **Data Integrity**: Ensure consistency and validity of stored trade deal data.

## Gas Optimization

### Storage Efficiency
- The `TradeDealStorage` struct is designed to minimize storage slot usage.
- Uses packed structs and efficient mappings.

### Function Efficiency
- `_createTradeDeal` and `_updateTradeDeal` avoid redundant checks.
- Functions interacting with external contracts (e.g., token transfers) are handled carefully.

## Error Handling

### Common Errors
- `Tdl: Invalid params`: Generic error for invalid input parameters.
- `Tdl: Trade deal not found`: Attempting to operate on a non-existent trade deal.
- `Tdl: Unauthorized`: Caller does not have required role or permissions.
- `Tdl: Insufficient funds`: Not enough balance for a financial operation.
- `Tdl: Invalid state`: Operation not allowed in current trade deal state.

## Best Practices

### Trade Deal Design
- Clearly define the `OperationMode` for each trade deal to set expectations.
- Use `requiredClaimTopics` to enforce identity-based participation rules.
- Plan for potential default/liquidation scenarios and their on-chain handling.

### Development Guidelines
- Write comprehensive unit tests for all functions, covering all roles and operation modes.
- Implement clear event logging for all state changes and financial movements.
- Integrate with off-chain monitoring systems for alerts on trade deal status changes (e.g., nearing maturity, default).

## Related Documentation

- [Trade Deal Management Facet](../../smart-contracts/facets/trade-deal-management-facet.md) - Reference for the Trade Deal Management Facet implementation.
- [Trade Deal Operations Facet](../../smart-contracts/facets/trade-deal-operations-facet.md) - Reference for the Trade Deal Operations Facet implementation.
- [ITradeDeal Interface](../../smart-contracts/interfaces/itradedeal.md) - Interface definition.
- [EIP-DRAFT-Collateralized-Trade-Deal-Standard](../../eips/EIP-DRAFT-Collateralized-Trade-Deal-Standard.md) - The full EIP specification.
- [Diamond Standard Overview](../../smart-contracts/diamond.md)