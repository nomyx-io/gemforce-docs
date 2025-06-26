# FeeDistributorFacet

## Overview

The [`FeeDistributorFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/FeeDistributorFacet.sol) provides comprehensive fee distribution functionality within the Gemforce diamond system. This facet enables automated distribution of fees and revenues to multiple recipients based on configurable weights, supporting complex revenue sharing models for marketplace operations, trade deals, and other platform activities.

## Contract Details

- **Contract Name**: `FeeDistributorFacet`
- **Inheritance**: `IFeeDistributor`, `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Flexible Fee Distribution
- Configure multiple fee recipients with custom weights
- Proportional distribution based on weight ratios
- Support for any ERC20 token distribution
- Automated calculation and distribution

### 🔹 Revenue Sharing Models
- Marketplace commission distribution
- Trade deal fee sharing
- Platform revenue allocation
- Partner revenue sharing

### 🔹 Transparent Calculations
- View-only calculation functions
- Precise fee amount calculations
- Adjustable weight basis for flexibility
- Real-time distribution previews

### 🔹 Secure Operations
- Owner-only configuration functions
- Validation of all parameters
- Event logging for transparency
- Integration with diamond access control

## Core Data Structures

### Fee Distribution Configuration
- **Distribution Token**: ERC20 token used for fee payments (e.g., USDC)
- **Total Weight Basis**: Denominator for weight calculations (e.g., 10000 for basis points)
- **Fee Receivers**: Array of addresses that receive fee distributions
- **Fee Weights**: Corresponding weights for each receiver

### Storage Pattern
Uses diamond storage pattern via [`FeeDistributorLib`](../libraries/fee-distributor-lib.md):
- Prevents storage collisions in diamond architecture
- Efficient storage of fee configuration
- Persistent settings across contract upgrades

## Core Functions

### Initialization Functions

#### `initializeFeeDistributor()`
```solidity
function initializeFeeDistributor(
    address _distributionToken,
    uint256 _totalWeightBasis
) external onlyOwner
```

**Purpose**: Initializes the fee distributor with token and weight basis configuration.

**Parameters**:
- `_distributionToken` (address): ERC20 token address for fee distributions
- `_totalWeightBasis` (uint256): Total weight basis for calculations (e.g., 10000)

**Access Control**: Owner only

**Validation**:
- Distribution token cannot be zero address
- Total weight basis must be greater than zero
- Can only be called once during deployment

**Process**:
1. Validates input parameters
2. Initializes fee distributor storage
3. Sets distribution token and weight basis
4. Emits FeeDistributorInitialized event

**Events**: `FeeDistributorInitialized(distributionToken, totalWeightBasis)`

**Example Usage**:
```solidity
// Initialize with USDC and basis points (10000)
IFeeDistributor(diamond).initializeFeeDistributor(
    usdcAddress,
    10000  // 10000 basis points = 100%
);
```

**Error Conditions**:
- `"FeeDistributor: Distribution token cannot be zero address"` - Invalid token address
- Already initialized error from library

---

### Configuration Functions

#### `setFeeReceivers()`
```solidity
function setFeeReceivers(
    address[] calldata _feeReceivers,
    uint256[] calldata _feeWeights
) external override onlyOwner
```

**Purpose**: Configures the addresses and weights for fee distribution.

**Parameters**:
- `_feeReceivers` (address[]): Array of addresses to receive fees
- `_feeWeights` (uint256[]): Array of weights corresponding to each receiver

**Access Control**: Owner only

**Validation**:
- Arrays must have equal length
- No zero addresses in receivers
- Weights must be greater than zero
- Total weights should not exceed weight basis

**Process**:
1. Validates input arrays
2. Updates fee receiver configuration
3. Stores receivers and weights
4. Emits FeeReceiversSet event

**Events**: `FeeReceiversSet(receivers, weights)`

**Example Usage**:
```solidity
// Set up fee distribution: 70% to treasury, 20% to team, 10% to marketing
address[] memory receivers = new address[](3);
receivers[0] = treasuryAddress;
receivers[1] = teamAddress;
receivers[2] = marketingAddress;

uint256[] memory weights = new uint256[](3);
weights[0] = 7000;  // 70%
weights[1] = 2000;  // 20%
weights[2] = 1000;  // 10%

IFeeDistributor(diamond).setFeeReceivers(receivers, weights);
```

**Weight Calculation Example**:
```solidity
// With totalWeightBasis = 10000
// Treasury: 7000/10000 = 70%
// Team: 2000/10000 = 20%
// Marketing: 1000/10000 = 10%
```

---

### Query Functions

#### `getFeeReceivers()`
```solidity
function getFeeReceivers() external view override returns (
    address[] memory feeReceivers,
    uint256[] memory feeWeights
)
```

**Purpose**: Returns the current fee receiver configuration.

**Returns**:
- `feeReceivers` (address[]): Array of fee recipient addresses
- `feeWeights` (uint256[]): Array of corresponding weights

**Example Usage**:
```solidity
(address[] memory receivers, uint256[] memory weights) = 
    IFeeDistributor(diamond).getFeeReceivers();

for (uint256 i = 0; i < receivers.length; i++) {
    console.log("Receiver:", receivers[i]);
    console.log("Weight:", weights[i]);
}
```

---

#### `calculateAmounts()`
```solidity
function calculateAmounts(uint256 principalAmount) external view override returns (
    uint256 adjustedAmount,
    uint256[] memory feeAmounts
)
```

**Purpose**: Calculates fee distribution amounts without executing transfers.

**Parameters**:
- `principalAmount` (uint256): Total amount before fee deduction

**Returns**:
- `adjustedAmount` (uint256): Principal amount after deducting fees
- `feeAmounts` (uint256[]): Individual fee amounts for each receiver

**Example Usage**:
```solidity
// Calculate fees for a $1000 transaction
uint256 totalAmount = 1000 * 10**6; // 1000 USDC (6 decimals)

(uint256 adjustedAmount, uint256[] memory feeAmounts) = 
    IFeeDistributor(diamond).calculateAmounts(totalAmount);

console.log("Amount after fees:", adjustedAmount);
console.log("Treasury fee:", feeAmounts[0]);
console.log("Team fee:", feeAmounts[1]);
console.log("Marketing fee:", feeAmounts[2]);
```

**Calculation Logic**:
```solidity
// Example with 1000 USDC and weights [7000, 2000, 1000]
// Total fee weight: 10000 (100%)
// Treasury fee: 1000 * 7000 / 10000 = 700 USDC
// Team fee: 1000 * 2000 / 10000 = 200 USDC
// Marketing fee: 1000 * 1000 / 10000 = 100 USDC
// Adjusted amount: 1000 - (700 + 200 + 100) = 0 USDC
```

---

### Distribution Functions

#### `distributeAmounts()`
```solidity
function distributeAmounts(
    address principalAmountReceiver,
    uint256 _principalAmount
) external override returns (
    address adjustedAmountReceiver,
    uint256 adjustedAmount,
    address[] memory feeReceivers,
    uint256[] memory feeAmounts
)
```

**Purpose**: Executes fee distribution by transferring calculated amounts to recipients.

**Parameters**:
- `principalAmountReceiver` (address): Address to receive the adjusted principal amount
- `_principalAmount` (uint256): Total amount to distribute

**Returns**:
- `adjustedAmountReceiver` (address): Address that received the adjusted amount
- `adjustedAmount` (uint256): Adjusted principal amount transferred
- `feeReceivers` (address[]): Addresses that received fee amounts
- `feeAmounts` (uint256[]): Individual fee amounts transferred

**Access Control**: Can be called by authorized contracts (marketplace, trade deals)

**Requirements**:
- Contract must have sufficient token balance or allowance
- All recipient addresses must be valid
- Distribution token must be properly configured

**Process**:
1. Calculates fee amounts using internal logic
2. Transfers fees to each configured receiver
3. Transfers adjusted amount to principal receiver
4. Emits AmountsDistributed event
5. Returns distribution details

**Events**: `AmountsDistributed(principalReceiver, originalAmount, adjustedReceiver, adjustedAmount, feeReceivers, feeAmounts)`

**Example Usage**:
```solidity
// Distribute marketplace sale proceeds
(
    address adjustedReceiver,
    uint256 adjustedAmount,
    address[] memory feeReceivers,
    uint256[] memory feeAmounts
) = IFeeDistributor(diamond).distributeAmounts(
    sellerAddress,
    saleAmount
);

console.log("Seller received:", adjustedAmount);
console.log("Fees distributed to", feeReceivers.length, "recipients");
```

## Integration Examples

### Marketplace Integration
```solidity
// Integrate fee distribution with marketplace sales
contract MarketplaceIntegration {
    function completeSale(
        uint256 tokenId,
        uint256 salePrice,
        address seller,
        address buyer
    ) external {
        // Transfer payment from buyer to contract
        IERC20(paymentToken).transferFrom(buyer, address(this), salePrice);
        
        // Distribute fees and pay seller
        (
            address adjustedReceiver,
            uint256 sellerAmount,
            address[] memory feeReceivers,
            uint256[] memory feeAmounts
        ) = IFeeDistributor(diamond).distributeAmounts(seller, salePrice);
        
        // Transfer NFT to buyer
        IERC721(nftContract).transferFrom(seller, buyer, tokenId);
        
        // Log the transaction
        emit SaleCompleted(
            tokenId,
            salePrice,
            seller,
            buyer,
            sellerAmount,
            feeReceivers,
            feeAmounts
        );
    }
}
```

### Trade Deal Fee Distribution
```solidity
// Distribute trade deal fees and interest
contract TradeDealFeeDistribution {
    function distributeInterestPayment(
        uint256 tradeDealId,
        uint256 interestAmount,
        address borrower
    ) external {
        // Calculate and distribute fees from interest payment
        (
            address adjustedReceiver,
            uint256 netInterest,
            address[] memory feeReceivers,
            uint256[] memory feeAmounts
        ) = IFeeDistributor(diamond).distributeAmounts(
            getTradeDealLender(tradeDealId),
            interestAmount
        );
        
        // Record interest distribution
        emit InterestDistributed(
            tradeDealId,
            interestAmount,
            netInterest,
            feeReceivers,
            feeAmounts
        );
    }
}
```

### Dynamic Fee Configuration
```solidity
// Dynamically adjust fee structure based on conditions
contract DynamicFeeManager {
    function updateFeeStructure(
        uint256 marketCondition,
        uint256 volumeTier
    ) external onlyOwner {
        address[] memory receivers = new address[](4);
        receivers[0] = treasuryAddress;
        receivers[1] = teamAddress;
        receivers[2] = marketingAddress;
        receivers[3] = developmentAddress;
        
        uint256[] memory weights = new uint256[](4);
        
        if (marketCondition == HIGH_VOLUME) {
            // Reduce fees during high volume periods
            weights[0] = 5000;  // 50% treasury
            weights[1] = 1500;  // 15% team
            weights[2] = 1000;  // 10% marketing
            weights[3] = 2500;  // 25% development
        } else {
            // Standard fee structure
            weights[0] = 6000;  // 60% treasury
            weights[1] = 2000;  // 20% team
            weights[2] = 1000;  // 10% marketing
            weights[3] = 1000;  // 10% development
        }
        
        IFeeDistributor(diamond).setFeeReceivers(receivers, weights);
    }
}
```

### Multi-Token Fee Distribution
```solidity
// Handle fee distribution for multiple tokens
contract MultiTokenFeeDistribution {
    mapping(address => bool) public supportedTokens;
    
    function distributeFees(
        address token,
        uint256 amount,
        address principalReceiver
    ) external {
        require(supportedTokens[token], "Token not supported");
        
        // Temporarily set distribution token if different
        address currentToken = getCurrentDistributionToken();
        if (currentToken != token) {
            // Would need additional function to change distribution token
            // or handle multiple tokens in the fee distributor
        }
        
        IFeeDistributor(diamond).distributeAmounts(principalReceiver, amount);
    }
}
```

### Fee Analytics and Reporting
```solidity
// Track and report fee distribution analytics
contract FeeAnalytics {
    struct FeeReport {
        uint256 totalDistributed;
        uint256 totalFees;
        address[] recipients;
        uint256[] amounts;
        uint256 timestamp;
    }
    
    FeeReport[] public feeReports;
    
    function recordFeeDistribution(
        uint256 originalAmount,
        address principalReceiver
    ) external returns (uint256 reportId) {
        (
            address adjustedReceiver,
            uint256 adjustedAmount,
            address[] memory feeReceivers,
            uint256[] memory feeAmounts
        ) = IFeeDistributor(diamond).distributeAmounts(
            principalReceiver,
            originalAmount
        );
        
        uint256 totalFees = originalAmount - adjustedAmount;
        
        feeReports.push(FeeReport({
            totalDistributed: originalAmount,
            totalFees: totalFees,
            recipients: feeReceivers,
            amounts: feeAmounts,
            timestamp: block.timestamp
        }));
        
        return feeReports.length - 1;
    }
    
    function getFeeReport(uint256 reportId) external view returns (FeeReport memory) {
        require(reportId < feeReports.length, "Invalid report ID");
        return feeReports[reportId];
    }
}
```

## Events

### Configuration Events
```solidity
event FeeDistributorInitialized(address indexed distributionToken, uint256 totalWeightBasis);
event FeeReceiversSet(address[] receivers, uint256[] weights);
```

### Distribution Events
```solidity
event AmountsDistributed(
    address indexed principalReceiver,
    uint256 originalAmount,
    address indexed adjustedReceiver,
    uint256 adjustedAmount,
    address[] feeReceivers,
    uint256[] feeAmounts
);
```

## Security Considerations

### Access Control
- Only contract owner can initialize and configure fee distribution
- Distribution function can be called by authorized contracts
- Validation of all addresses and amounts

### Financial Security
- Requires sufficient token balance or allowance for distributions
- Atomic transfers ensure all-or-nothing distribution
- Precise calculations prevent rounding errors

### Configuration Validation
- Prevents zero addresses in fee receivers
- Validates weight arrays match receiver arrays
- Ensures total weights don't exceed basis

## Gas Optimization

### Efficient Calculations
- Single-pass calculation of all fee amounts
- Minimal storage reads and writes
- Optimized array operations

### Batch Operations
- Single transaction for all fee distributions
- Reduced gas costs compared to individual transfers
- Efficient event emission

## Error Handling

### Configuration Errors
- `"FeeDistributor: Distribution token cannot be zero address"` - Invalid token
- Array length mismatch errors
- Invalid weight configuration errors

### Distribution Errors
- Insufficient balance or allowance errors
- Transfer failure errors
- Invalid recipient address errors

## Testing Considerations

### Unit Tests
- Fee calculation accuracy
- Distribution execution
- Configuration validation
- Access control enforcement

### Integration Tests
- Marketplace integration
- Trade deal integration
- Multi-contract workflows
- Event emission verification

## Related Documentation

- [IFeeDistributor](../interfaces/ifee-distributor.md) - Fee distributor interface
- [FeeDistributorLib](../libraries/fee-distributor-lib.md) - Fee distribution utilities
- [MarketplaceFacet](./marketplace-facet.md) - Marketplace integration
- [TradeDealManagementFacet](./trade-deal-management-facet.md) - Trade deal integration
- [Revenue Sharing Guide](../../guides/revenue-sharing.md) - Implementation guide

---

*This facet provides the foundation for flexible and transparent fee distribution within the Gemforce platform, enabling complex revenue sharing models and automated fee management across all platform operations.*