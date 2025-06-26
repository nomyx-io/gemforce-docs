# TradeDealManagementFacet

## Overview

The [`TradeDealManagementFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/TradeDealManagementFacet.sol) provides comprehensive CRUD (Create, Read, Update, Delete) operations and basic management functionality for trade deals within the Gemforce diamond system. This facet enables the creation and management of collateralized trade deals for invoice financing and other financial instruments.

## Contract Details

- **Contract Name**: `TradeDealManagementFacet`
- **Inheritance**: `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Trade Deal Lifecycle Management
- Create new trade deals with configurable parameters
- Update existing trade deal configurations
- Activate and deactivate trade deals
- Comprehensive status tracking

### 🔹 Multi-Token Architecture
- NFT collateral support (invoice NFTs)
- Collateral token management
- Interest token distribution
- USDC payment integration

### 🔹 Operation Modes
- Flexible operation modes for different financing scenarios
- Configurable interest rates and collateral ratios
- Identity-based access control through claim topics

### 🔹 Comprehensive Monitoring
- Real-time funding status tracking
- Debt and repayment monitoring
- Participant verification
- Financial metrics calculation

## Core Data Structures

### Operation Modes
```solidity
enum OperationMode {
    STANDARD,           // Standard trade deal operation
    ACCELERATED,        // Accelerated funding mode
    CONSERVATIVE,       // Conservative risk mode
    CUSTOM             // Custom operation parameters
}
```

### Trade Deal Parameters
```solidity
struct CreateTradeDealParams {
    string name;                        // Trade deal name
    string symbol;                      // Trade deal symbol
    uint256 interestRate;              // Interest rate (basis points)
    uint256 collateralToInterestRatio; // Collateral to interest ratio
    uint256[] requiredClaimTopics;     // Required identity claims
    address collateralAddress;         // Collateral token contract
    address interestAddress;           // Interest token contract
    address usdcAddress;               // USDC payment token
    OperationMode operationMode;       // Operation mode
}
```

## Core Functions

### Trade Deal Creation

#### `createTradeDeal()`
```solidity
function createTradeDeal(
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    uint256[] memory requiredClaimTopics,
    address collateralAddress,
    address interestAddress,
    address usdcAddress,
    TradeDealLib.OperationMode operationMode
) public onlyOwner returns (uint256)
```

**Purpose**: Creates a new trade deal with specified parameters and returns the trade deal ID.

**Parameters**:
- `name`: Human-readable name for the trade deal
- `symbol`: Short symbol identifier for the trade deal
- `interestRate`: Interest rate in basis points (e.g., 500 = 5%)
- `collateralToInterestRatio`: Ratio of collateral to interest tokens
- `requiredClaimTopics`: Array of required identity claim topics for participation
- `collateralAddress`: Address of the collateral token contract
- `interestAddress`: Address of the interest token contract
- `usdcAddress`: Address of the USDC token contract for payments
- `operationMode`: Operation mode for the trade deal

**Access Control**: Owner only

**Returns**: `uint256` - The newly created trade deal ID

**Process**:
1. Validates all input parameters
2. Creates trade deal using TradeDealLib
3. Emits TradeDealCreated event
4. Sets required claim topics if provided
5. Returns the trade deal ID

**Events**: 
- `TradeDealCreated` - Emitted with complete trade deal information
- `TradeDealRequiredClaimTopicsSet` - Emitted if claim topics are set

**Example Usage**:
```solidity
// Create a standard trade deal for invoice financing
uint256 tradeDealId = ITradeDeal(diamond).createTradeDeal(
    "Invoice Financing Deal #1",
    "IFD1",
    750,  // 7.5% interest rate
    150,  // 1.5x collateral ratio
    [1, 2, 3], // Required claim topics
    collateralTokenAddress,
    interestTokenAddress,
    usdcTokenAddress,
    TradeDealLib.OperationMode.STANDARD
);
```

### Trade Deal Updates

#### `updateTradeDeal()`
```solidity
function updateTradeDeal(
    uint256 tradeDealId,
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
) external onlyOwner
```

**Purpose**: Updates an existing trade deal's configuration parameters.

**Parameters**:
- `tradeDealId`: ID of the trade deal to update
- `name`: New name for the trade deal
- `symbol`: New symbol for the trade deal
- `interestRate`: New interest rate in basis points
- `collateralToInterestRatio`: New collateral to interest ratio
- `collateralAddress`: New collateral token address
- `interestAddress`: New interest token address
- `usdcAddress`: New USDC token address

**Access Control**: Owner only

**Validation**:
- Trade deal must exist
- New parameters must be valid
- Cannot update active funded deals

**Events**: `TradeDealUpdated` - Emitted with updated parameters

### State Management

#### `activateTradeDeal()`
```solidity
function activateTradeDeal(uint256 tradeDealId) external onlyOwner
```

**Purpose**: Activates a trade deal, making it available for funding.

**Parameters**:
- `tradeDealId`: ID of the trade deal to activate

**Access Control**: Owner only

**Events**: `TradeDealActivated`

#### `deactivateTradeDeal()`
```solidity
function deactivateTradeDeal(uint256 tradeDealId) external onlyOwner
```

**Purpose**: Deactivates a trade deal, preventing new funding.

**Parameters**:
- `tradeDealId`: ID of the trade deal to deactivate

**Access Control**: Owner only

**Events**: `TradeDealDeactivated`

### Information Retrieval

#### `getTradeDealInfo()`
```solidity
function getTradeDealInfo(uint256 tradeDealId) external view returns (
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    bool active,
    TradeDealLib.OperationMode operationMode
)
```

**Purpose**: Retrieves basic information about a trade deal.

**Parameters**:
- `tradeDealId`: ID of the trade deal

**Returns**: Basic trade deal configuration and status

#### `getTradeDealFullStatus()`
```solidity
function getTradeDealFullStatus(uint256 tradeDealId) external view returns (
    uint256 fundingTarget,
    uint256 currentBalance,
    bool isFunded,
    bool isFundingWithdrawn,
    uint256 totalDebt,
    uint256 repaidAmount,
    bool isRepaid
)
```

**Purpose**: Retrieves comprehensive financial status of a trade deal.

**Parameters**:
- `tradeDealId`: ID of the trade deal

**Returns**: Complete financial status including funding, debt, and repayment information

#### `getAllTradeDealIds()`
```solidity
function getAllTradeDealIds() external view returns (uint256[] memory)
```

**Purpose**: Returns an array of all trade deal IDs in the system.

**Returns**: Array of trade deal IDs

### Participant and Status Checks

#### `isTradeDealParticipant()`
```solidity
function isTradeDealParticipant(uint256 tradeDealId, address user) external view returns (bool)
```

**Purpose**: Checks if a user is a participant in a specific trade deal.

**Parameters**:
- `tradeDealId`: ID of the trade deal
- `user`: Address to check

**Returns**: `true` if user is a participant, `false` otherwise

#### `isTradeDealFunded()`
```solidity
function isTradeDealFunded(uint256 tradeDealId) external view returns (bool)
```

**Purpose**: Checks if a trade deal has reached its funding target.

#### `isTradeDealRepaid()`
```solidity
function isTradeDealRepaid(uint256 tradeDealId) external view returns (bool)
```

**Purpose**: Checks if a trade deal has been fully repaid.

### Utility Functions

#### `getNFTInvoiceTotalAmount()`
```solidity
function getNFTInvoiceTotalAmount(uint256 tokenId) external view returns (uint256)
```

**Purpose**: Gets the total amount of an invoice NFT used as collateral.

**Parameters**:
- `tokenId`: ID of the invoice NFT

**Returns**: Total invoice amount

#### `getTradeDealTokenAddresses()`
```solidity
function getTradeDealTokenAddresses(uint256 tradeDealId) external view returns (
    address nftAddress,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
)
```

**Purpose**: Retrieves all token contract addresses associated with a trade deal.

## Events

### TradeDealCreated
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
    address usdcAddress,
    TradeDealLib.OperationMode operationMode
);
```
Emitted when a new trade deal is created.

### TradeDealUpdated
```solidity
event TradeDealUpdated(
    uint256 indexed tradeDealId,
    string name,
    string symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    bool active,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
);
```
Emitted when a trade deal is updated.

### TradeDealActivated / TradeDealDeactivated
```solidity
event TradeDealActivated(uint256 indexed tradeDealId);
event TradeDealDeactivated(uint256 indexed tradeDealId);
```
Emitted when trade deals are activated or deactivated.

### TradeDealRequiredClaimTopicsSet
```solidity
event TradeDealRequiredClaimTopicsSet(uint256 indexed tradeDealId, uint256[] claimTopics);
```
Emitted when required claim topics are set for a trade deal.

## Integration Examples

### Create Invoice Financing Deal
```solidity
// Deploy collateral and interest tokens first
address collateralToken = deployCollateralToken();
address interestToken = deployInterestToken();

// Create trade deal for invoice financing
uint256 tradeDealId = ITradeDeal(diamond).createTradeDeal(
    "Q1 2024 Invoice Financing",
    "Q1IF24",
    800,  // 8% annual interest rate
    120,  // 1.2x collateral ratio
    [1, 2], // KYC and accreditation claims required
    collateralToken,
    interestToken,
    usdcAddress,
    TradeDealLib.OperationMode.STANDARD
);

// Activate the trade deal
ITradeDeal(diamond).activateTradeDeal(tradeDealId);
```

### Monitor Trade Deal Status
```solidity
// Get basic trade deal information
(
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralRatio,
    bool active,
    TradeDealLib.OperationMode mode
) = ITradeDeal(diamond).getTradeDealInfo(tradeDealId);

// Get detailed financial status
(
    uint256 fundingTarget,
    uint256 currentBalance,
    bool isFunded,
    bool isFundingWithdrawn,
    uint256 totalDebt,
    uint256 repaidAmount,
    bool isRepaid
) = ITradeDeal(diamond).getTradeDealFullStatus(tradeDealId);

console.log("Trade Deal:", name);
console.log("Funding Progress:", currentBalance, "/", fundingTarget);
console.log("Is Funded:", isFunded);
console.log("Total Debt:", totalDebt);
console.log("Repaid Amount:", repaidAmount);
```

### Update Trade Deal Parameters
```solidity
// Update interest rate and collateral ratio
ITradeDeal(diamond).updateTradeDeal(
    tradeDealId,
    "Updated Invoice Financing Deal",
    "UIFD24",
    750,  // Reduced to 7.5% interest rate
    110,  // Reduced to 1.1x collateral ratio
    collateralToken,
    interestToken,
    usdcAddress
);
```

## Security Considerations

### Access Control
- All management functions are owner-only
- Participant verification through identity claims
- State validation before operations

### Financial Security
- Interest rate and ratio validation
- Funding target verification
- Debt tracking and repayment monitoring

### Operational Security
- Trade deal state management
- Token address validation
- Operation mode enforcement

## Integration with Other Facets

### TradeDealOperationsFacet
- Handles funding, withdrawal, and repayment operations
- Manages participant interactions
- Processes collateral and interest distributions

### IdentityRegistryFacet
- Validates participant eligibility
- Enforces claim topic requirements
- Manages access control

### Token Contracts
- Collateral token for securing deals
- Interest token for profit distribution
- USDC for stable value payments

## Error Handling

### Common Errors
- `"Trade deal does not exist"` - Invalid trade deal ID
- `"Only owner can perform this action"` - Unauthorized access
- `"Invalid parameters"` - Parameter validation failure
- `"Trade deal already active/inactive"` - Invalid state transition

## Testing Considerations

### Unit Tests
- Trade deal creation with various parameters
- Update operations and validation
- State transitions (activate/deactivate)
- Information retrieval accuracy

### Integration Tests
- Multi-facet interactions
- Token contract integration
- Identity system integration
- End-to-end trade deal lifecycle

## Related Documentation

- [TradeDealOperationsFacet](trade-deal-operations-facet.md) - Trade deal operations
- [ITradeDeal](../interfaces/itrade-deal.md) - Trade deal interface
- [TradeDealLib](../libraries/trade-deal-lib.md) - Trade deal utility library
- [EIP-DRAFT-Collateralized-Trade-Deal-Standard](../../eips/EIP-DRAFT-Collateralized-Trade-Deal-Standard.md) - EIP specification

---

*This facet provides the foundation for trade deal management in the Gemforce collateralized finance system.*