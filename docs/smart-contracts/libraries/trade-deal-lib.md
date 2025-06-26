# TradeDealLib Library

## Overview

The [`TradeDealLib`](../../../contracts/libraries/TradeDealLib.sol) library provides core utilities and data structures for managing collateralized trade deals within the Gemforce platform. This library implements the Diamond Standard storage pattern and provides essential functions for trade deal creation, management, participant handling, and financial operations including funding, repayment, and collateral management.

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
        
        // Create result array
        tradeDealIds = new uint256[](count);
        uint256 index = 0;
        
        for (uint256 i = 0; i < allIds.length; i++) {
            TradeDealLib.TradeDeal memory tradeDeal = getTradeDeal(allIds[i]);
            if (tradeDeal.operationMode == mode) {
                tradeDealIds[index] = allIds[i];
                index++;
            }
        }
    }
    
    // Helper functions (would be implemented to access actual storage)
    function getTradeDeal(uint256 tradeDealId) internal view returns (TradeDealLib.TradeDeal memory) {
        // Implementation would access Diamond storage
    }
    
    function getFundingTarget(uint256 tradeDealId) internal view returns (uint256) {
        // Implementation would access storage
    }
    
    function getCurrentFunding(uint256 tradeDealId) internal view returns (uint256) {
        // Implementation would access storage
    }
    
    function getRepaidAmount(uint256 tradeDealId) internal view returns (uint256) {
        // Implementation would access storage
    }
    
    function getOutstandingDebt(uint256 tradeDealId) internal view returns (uint256) {
        // Implementation would access storage
    }
    
    function getAllTradeDealIds() internal view returns (uint256[] memory) {
        // Implementation would access storage
    }
    
    function isDefaulted(uint256 tradeDealId) internal view returns (bool) {
        // Implementation would check default conditions
    }
}
```

### Role-Based Access Control Manager
```solidity
// Manager for trade deal roles and permissions
contract TradeDealAccessManager {
    using TradeDealLib for TradeDealLib.TradeDealStorage;
    
    event RoleAssigned(uint256 indexed tradeDealId, address indexed user, TradeDealLib.Role role);
    event PermissionGranted(uint256 indexed tradeDealId, address indexed user, uint256 permission);
    event PermissionRevoked(uint256 indexed tradeDealId, address indexed user, uint256 permission);
    
    function assignRole(
        uint256 tradeDealId,
        address user,
        TradeDealLib.Role role
    ) external onlyTradeDealAdmin(tradeDealId) {
        // Set user role in storage
        setUserRole(tradeDealId, user, role);
        
        // Assign default permissions based on role
        uint256 defaultPermissions = getDefaultPermissionsForRole(role);
        setUserPermissions(tradeDealId, user, defaultPermissions);
        
        emit RoleAssigned(tradeDealId, user, role);
    }
    
    function grantPermission(
        uint256 tradeDealId,
        address user,
        uint256 permission
    ) external onlyTradeDealAdmin(tradeDealId) {
        uint256 currentPermissions = getUserPermissions(tradeDealId, user);
        uint256 newPermissions = currentPermissions | permission;
        
        setUserPermissions(tradeDealId, user, newPermissions);
        
        emit PermissionGranted(tradeDealId, user, permission);
    }
    
    function revokePermission(
        uint256 tradeDealId,
        address user,
        uint256 permission
    ) external onlyTradeDealAdmin(tradeDealId) {
        uint256 currentPermissions = getUserPermissions(tradeDealId, user);
        uint256 newPermissions = currentPermissions & ~permission;
        
        setUserPermissions(tradeDealId, user, newPermissions);
        
        emit PermissionRevoked(tradeDealId, user, permission);
    }
    
    function hasPermission(
        uint256 tradeDealId,
        address user,
        uint256 permission
    ) external view returns (bool) {
        uint256 userPermissions = getUserPermissions(tradeDealId, user);
        return (userPermissions & permission) != 0;
    }
    
    function getDefaultPermissionsForRole(
        TradeDealLib.Role role
    ) public pure returns (uint256 permissions) {
        if (role == TradeDealLib.Role.ADMIN) {
            permissions = TradeDealLib.PERMISSION_DEPOSIT_FUNDS |
                         TradeDealLib.PERMISSION_WITHDRAW_FUNDS |
                         TradeDealLib.PERMISSION_DEPOSIT_COLLATERAL |
                         TradeDealLib.PERMISSION_WITHDRAW_COLLATERAL |
                         TradeDealLib.PERMISSION_DISTRIBUTE_INTEREST;
        } else if (role == TradeDealLib.Role.LENDER) {
            permissions = TradeDealLib.PERMISSION_DEPOSIT_FUNDS;
        } else if (role == TradeDealLib.Role.BORROWER) {
            permissions = TradeDealLib.PERMISSION_DEPOSIT_COLLATERAL |
                         TradeDealLib.PERMISSION_WITHDRAW_FUNDS;
        } else if (role == TradeDealLib.Role.UNDERWRITER) {
            permissions = TradeDealLib.PERMISSION_DEPOSIT_FUNDS |
                         TradeDealLib.PERMISSION_DISTRIBUTE_INTEREST;
        } else if (role == TradeDealLib.Role.LIQUIDATOR) {
            permissions = TradeDealLib.PERMISSION_WITHDRAW_COLLATERAL;
        }
        // NONE role gets no permissions
    }
    
    function getUserAccessSummary(
        uint256 tradeDealId,
        address user
    ) external view returns (
        TradeDealLib.Role role,
        uint256 permissions,
        bool canDepositFunds,
        bool canWithdrawFunds,
        bool canDepositCollateral,
        bool canWithdrawCollateral,
        bool canDistributeInterest
    ) {
        role = getUserRole(tradeDealId, user);
        permissions = getUserPermissions(tradeDealId, user);
        
        canDepositFunds = (permissions & TradeDealLib.PERMISSION_DEPOSIT_FUNDS) != 0;
        canWithdrawFunds = (permissions & TradeDealLib.PERMISSION_WITHDRAW_FUNDS) != 0;
        canDepositCollateral = (permissions & TradeDealLib.PERMISSION_DEPOSIT_COLLATERAL) != 0;
        canWithdrawCollateral = (permissions & TradeDealLib.PERMISSION_WITHDRAW_COLLATERAL) != 0;
        canDistributeInterest = (permissions & TradeDealLib.PERMISSION_DISTRIBUTE_INTEREST) != 0;
    }
    
    // Helper functions (would access Diamond storage)
    function setUserRole(uint256 tradeDealId, address user, TradeDealLib.Role role) internal {
        // Implementation would access storage
    }
    
    function getUserRole(uint256 tradeDealId, address user) internal view returns (TradeDealLib.Role) {
        // Implementation would access storage
    }
    
    function setUserPermissions(uint256 tradeDealId, address user, uint256 permissions) internal {
        // Implementation would access storage
    }
    
    function getUserPermissions(uint256 tradeDealId, address user) internal view returns (uint256) {
        // Implementation would access storage
    }
    
    modifier onlyTradeDealAdmin(uint256 tradeDealId) {
        require(
            getUserRole(tradeDealId, msg.sender) == TradeDealLib.Role.ADMIN,
            "Only trade deal admin can perform this action"
        );
        _;
    }
}
```

## Events

The library defines comprehensive events for trade deal lifecycle tracking:

### Core Trade Deal Events
```solidity
event TradeDealCreated(
    uint256 indexed tradeDealId,
    string name,
    string symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    bool active,
    address nftAddress,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
);

event TradeDealActivated(uint256 indexed tradeDealId);
event TradeDealDeactivated(uint256 indexed tradeDealId);
```

### Participant Management Events
```solidity
event TradeDealParticipantAdded(uint256 indexed tradeDealId, address indexed participant);
event TradeDealParticipantRemoved(uint256 indexed tradeDealId, address indexed participant);
event TradeDealRequiredClaimTopicsSet(uint256 indexed tradeDealId, uint256[] claimTopics);
```

### Financial Operation Events
```solidity
event InvoiceDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
event InvoiceWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
event USDCDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 amount);
event USDCWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 amount);
event InterestDistributedForTradeDeal(uint256 indexed tradeDealId, uint256 totalInterest, uint256 invoicePoolInterest, uint256 interestInterest, uint256 interestTokensMinted);
```

### Enhanced Financial Events
```solidity
event TradeDealFullyFunded(uint256 indexed tradeDealId, uint256 fundingTarget);
event TradeDealFundingWithdrawn(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
event TradeDealRepaid(uint256 indexed tradeDealId, address indexed repayer, uint256 amount, bool fullyRepaid);
event CollateralTokensRedeemed(uint256 indexed tradeDealId, address indexed redeemer, uint256 collateralAmount, uint256 usdcAmount);
```

## Security Considerations

### Access Control Security
- Role-based permissions with fine-grained control
- Operation mode validation for different access patterns
- Participant verification through claim topics
- Admin-only functions for critical operations

### Financial Security
- Precise tracking of funding, withdrawals, and repayments
- Collateral validation and ownership verification
- Interest calculation accuracy and overflow protection
- Secure token transfer mechanisms

### Storage Security
- Diamond Standard storage pattern for isolation
- Consistent state management across operations
- Prevention of storage collisions
- Secure access to storage slots

## Gas Optimization

### Storage Efficiency
- Optimized storage layout for trade deal data
- Efficient mapping structures for lookups
- Minimal storage writes during operations
- Packed data structures where possible

### Function Efficiency
- Internal functions for gas savings
- Batch operations for multiple updates
- Efficient event emission patterns
- Optimized validation logic

## Error Handling

### Common Errors
- "Trade deal does not exist" - Operating on non-existent trade deal
- Permission denied errors for unauthorized operations
- Invalid parameter errors during creation/updates
- Insufficient balance errors during financial operations
- Claim topic validation failures

### Best Practices
- Validate trade deal existence before operations
- Check permissions before executing functions
- Validate all input parameters
- Handle edge cases gracefully
- Provide clear error messages for debugging

## Testing Considerations

### Unit Tests
- Trade deal creation and management functions
- Role and permission system testing
- Financial operation validation
- Storage pattern implementation
- Event emission verification

### Integration Tests
- Diamond facet integration
- Multi-user workflow testing
- Financial operation sequences
- Access control scenarios
- Cross-contract interactions

## Related Documentation

- [ITradeDeal Interface](../interfaces/itradedeal.md) - Trade deal interface definition
- [TradeDealManagementFacet](../facets/trade-deal-management-facet.md) - Trade deal management implementation
- [TradeDealAdminFacet](../facets/trade-deal-admin-facet.md) - Administrative functions
- [TradeDealOperationsFacet](../facets/trade-deal-operations-facet.md) - Operational functions
- [Diamond Standard Guide](../../guides/diamond-standard.md) - Diamond pattern implementation
- [Trade Deal Guide](../../guides/trade-deals.md) - Implementation guide for trade deals

---

*This library provides the core utilities and data structures for trade deal management within the Gemforce platform, implementing secure storage patterns, role-based access control, and comprehensive financial operations for collateralized finance instruments.*