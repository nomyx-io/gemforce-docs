# EIP-DRAFT: Collateralized Trade Deal Standard for Invoice Financing

## Simple Summary

A standardized interface for creating and managing collateralized trade deals that enable invoice financing through tokenized collateral, interest distribution, and automated repayment mechanisms.

## Abstract

This EIP proposes a comprehensive standard for trade deals that enables:
- Creation of collateralized financing arrangements using invoice NFTs
- Tokenized collateral and interest distribution systems
- Multi-party funding with proportional token distribution
- Automated interest calculations and distributions
- Identity-based participation controls with claim requirements
- Multiple operation modes for different financing scenarios

## Motivation

Traditional invoice financing lacks transparency, standardization, and programmable automation. This standard addresses these limitations by providing a decentralized, transparent, and automated system for invoice financing that can be implemented across different platforms while maintaining interoperability.

## Specification

### Core Interface

```solidity
interface ICollateralizedTradeDeal {
    enum OperationMode { STANDARD, ADVANCED, CUSTOM }
    
    struct TradeDeal {
        string name;
        string symbol;
        uint256 interestRate; // In basis points (100 = 1%)
        uint256 collateralToInterestRatio;
        bool active;
        address nftAddress;
        address collateralAddress;
        address interestAddress;
        address usdcAddress;
        OperationMode operationMode;
        uint256[] requiredClaimTopics;
    }
    
    // Events
    event TradeDealCreated(uint256 indexed tradeDealId, string name, string symbol, uint256 interestRate, uint256 collateralToInterestRatio, bool active, address nftAddress, address collateralAddress, address interestAddress, address usdcAddress, OperationMode operationMode);
    event TradeDealFullyFunded(uint256 indexed tradeDealId, uint256 fundingTarget);
    event TradeDealFundingWithdrawn(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
    event TradeDealRepaid(uint256 indexed tradeDealId, address indexed repayer, uint256 amount, bool fullyRepaid);
    event CollateralTokensDistributed(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
    event CollateralTokensRedeemed(uint256 indexed tradeDealId, address indexed redeemer, uint256 collateralAmount, uint256 usdcAmount);
    event InterestDistributedForTradeDeal(uint256 indexed tradeDealId, uint256 totalInterest, uint256 invoicePoolInterest, uint256 interestInterest, uint256 interestTokensMinted);
    event InvoiceDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
    event InvoiceWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
    event USDCDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 amount, address depositor);
    event TradeDealRequiredClaimTopicsSet(uint256 indexed tradeDealId, uint256[] claimTopics);
    
    // Core Functions
    function createTradeDeal(string memory name, string memory symbol, uint256 interestRate, uint256 collateralToInterestRatio, uint256[] memory requiredClaimTopics, address collateralAddress, address interestAddress, address usdcAddress, OperationMode operationMode) external returns (uint256);
    function updateTradeDeal(uint256 tradeDealId, string memory name, string memory symbol, uint256 interestRate, uint256 collateralToInterestRatio, address collateralAddress, address interestAddress, address usdcAddress) external;
    function activateTradeDeal(uint256 tradeDealId) external;
    function deactivateTradeDeal(uint256 tradeDealId) external;
    
    // Funding Operations
    function tdDepositUSDC(uint256 tradeDealId, uint256 amount) external;
    function tdWithdrawUSDC(uint256 tradeDealId, uint256 amount) external;
    function withdrawTradeDealFundingForBorrower(uint256 tradeDealId, address borrowerAddress) external;
    
    // Collateral Operations
    function tdDepositInvoice(uint256 tradeDealId, uint256 tokenId) external;
    function tdWithdrawInvoice(uint256 tradeDealId, uint256 tokenId) external;
    
    // Repayment Operations
    function repayTradeDeal(uint256 tradeDealId, uint256 amount) external;
    function repayTradeDealForBorrower(uint256 tradeDealId, address borrower, uint256 amount) external;
    function redeemCollateralTokens(uint256 tradeDealId, uint256 collateralAmount) external;
    
    // Interest Distribution
    function tdDistributeInterest(uint256 tradeDealId) external;
    
    // Access Control
    function setTradeDealRequiredClaimTopics(uint256 tradeDealId, uint256[] memory claimTopics) external;
    function getTradeDealRequiredClaimTopics(uint256 tradeDealId) external view returns (uint256[] memory);
    
    // View Functions
    function getTradeDealInfo(uint256 tradeDealId) external view returns (string memory name, string memory symbol, uint256 interestRate, uint256 collateralToInterestRatio, bool active, OperationMode operationMode);
    function getTradeDealFullStatus(uint256 tradeDealId) external view returns (uint256 fundingTarget, uint256 currentBalance, bool isFunded, bool isFundingWithdrawn, uint256 totalDebt, uint256 repaidAmount, bool isRepaid);
    function isTradeDealParticipant(uint256 tradeDealId, address user) external view returns (bool);
    function isTradeDealFunded(uint256 tradeDealId) external view returns (bool);
    function isTradeDealRepaid(uint256 tradeDealId) external view returns (bool);
    function getAllTradeDealIds() external view returns (uint256[] memory);
}
```

### Key Features

#### 1. Collateralized Financing
- **Invoice NFTs as Collateral**: Use tokenized invoices as collateral for financing
- **Proportional Token Distribution**: Distribute collateral tokens based on funding contributions
- **Automated Collateral Management**: Handle collateral deposits and withdrawals

#### 2. Multi-Token System
- **Collateral Tokens**: Represent funding contributions and ownership stakes
- **Interest Tokens**: Represent earned interest and distribution rights
- **USDC Integration**: Standardized stablecoin for funding and repayments

#### 3. Interest Distribution
- **Automated Calculations**: Calculate interest based on configurable rates
- **Proportional Distribution**: Distribute interest based on token holdings
- **Dual Distribution**: Split between invoice pool and interest token holders

#### 4. Identity-Based Access Control
- **Claim Requirements**: Require specific identity claims for participation
- **Trusted Issuer Validation**: Verify claims through trusted issuer system
- **Flexible Access Control**: Configure different requirements per trade deal

### Operation Modes

The standard supports multiple operation modes:

1. **STANDARD**: Basic invoice financing with standard terms
2. **ADVANCED**: Enhanced features with complex interest structures
3. **CUSTOM**: Fully customizable parameters for specialized use cases

### Funding Lifecycle

```solidity
// 1. Create Trade Deal
uint256 tradeDealId = createTradeDeal("Invoice Deal #1", "ID1", 500, 10000, claimTopics, collateralToken, interestToken, usdcToken, OperationMode.STANDARD);

// 2. Deposit Invoices as Collateral
tdDepositInvoice(tradeDealId, invoiceTokenId);

// 3. Fund the Deal
tdDepositUSDC(tradeDealId, fundingAmount);

// 4. Withdraw Funding (when fully funded)
withdrawTradeDealFundingForBorrower(tradeDealId, borrowerAddress);

// 5. Distribute Interest (periodically)
tdDistributeInterest(tradeDealId);

// 6. Repay the Deal
repayTradeDeal(tradeDealId, repaymentAmount);

// 7. Redeem Collateral Tokens
redeemCollateralTokens(tradeDealId, collateralAmount);
```

## Rationale

### Tokenized Collateral System
Using ERC20 tokens to represent collateral ownership enables fractional ownership, transferability, and automated distribution mechanisms.

### Interest Rate in Basis Points
Using basis points (1/100th of a percent) provides sufficient precision for interest calculations while remaining human-readable.

### Identity Integration
Integration with ERC734/ERC735 identity standards enables compliance with regulatory requirements and sophisticated access control.

### Multi-Mode Operation
Different operation modes allow the standard to accommodate various financing scenarios while maintaining a consistent interface.

## Implementation Details

### Interest Calculation

```solidity
function calculateInterest(uint256 principal, uint256 rate, uint256 timeElapsed) internal pure returns (uint256) {
    // rate is in basis points (100 = 1%)
    // timeElapsed is in seconds
    // Returns interest for the time period
    return (principal * rate * timeElapsed) / (10000 * 365 days);
}
```

### Collateral Token Distribution

```solidity
function distributeCollateralTokens(uint256 tradeDealId, address recipient, uint256 usdcAmount) internal {
    TradeDeal storage deal = tradeDeals[tradeDealId];
    uint256 collateralAmount = usdcAmount; // 1:1 ratio, can be customized
    
    IERC20Mint(deal.collateralAddress).mintTo(recipient, collateralAmount);
    emit CollateralTokensDistributed(tradeDealId, recipient, collateralAmount);
}
```

### Security Considerations

1. **Access Control**: Proper validation of participant eligibility and claim requirements
2. **Integer Overflow**: Use SafeMath or Solidity 0.8+ overflow protection
3. **Reentrancy Protection**: Guard against reentrancy attacks in funding operations
4. **Interest Calculation**: Prevent manipulation of interest calculations
5. **Collateral Management**: Secure handling of collateral deposits and withdrawals

### Gas Optimization

- Batch operations for multiple participants
- Efficient storage patterns for trade deal data
- Optimized interest calculation algorithms
- Minimal external calls during operations

## Backwards Compatibility

This standard is designed to work with existing ERC20, ERC721, and identity standard implementations.

## Test Cases

Comprehensive test cases should cover:
- Trade deal lifecycle management
- Funding and withdrawal operations
- Interest calculation and distribution
- Collateral token management
- Identity-based access control
- Edge cases and error conditions

## Reference Implementation

The reference implementation includes:
- [Trade Deal Management Facet](../smart-contracts/facets/trade-deal-management-facet.md) - Core management functionality
- [Trade Deal Operations Facet](../smart-contracts/facets/trade-deal-operations-facet.md) - Operational functions
- [ITradeDeal Interface](../smart-contracts/interfaces/itradedeal.md) - Interface definition
- [Trade Deal Lib](../smart-contracts/libraries/trade-deal-lib.md) - Supporting library functions

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).