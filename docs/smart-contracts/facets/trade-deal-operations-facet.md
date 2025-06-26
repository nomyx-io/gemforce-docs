# TradeDealOperationsFacet

## Overview

The [`TradeDealOperationsFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/TradeDealOperationsFacet.sol) provides core operational functionality for trade deals within the Gemforce diamond system. This facet handles the essential operations including invoice deposits/withdrawals, USDC funding, collateral management, repayments, and redemptions that form the backbone of the trade deal lifecycle.

## Contract Details

- **Contract Name**: `TradeDealOperationsFacet`
- **Inheritance**: `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Invoice Management
- Deposit invoices as collateral backing
- Withdraw invoices from trade deals
- Participant validation for invoice operations
- Invoice ownership and transfer tracking

### 🔹 USDC Funding Operations
- Deposit USDC to fund trade deals
- Withdraw USDC with proper authorization
- Automatic collateral token distribution
- Funding target tracking and completion

### 🔹 Funding Withdrawal System
- Borrower funding withdrawal (admin-controlled)
- Self-service funding withdrawal
- Validation and authorization checks
- Funding distribution tracking

### 🔹 Repayment Processing
- Trade deal repayment by borrowers
- Admin-assisted repayment processing
- Partial and full repayment support
- Repayment validation and tracking

### 🔹 Collateral Redemption
- Redeem collateral tokens for USDC
- Proportional redemption calculations
- Participant validation for redemptions
- Collateral-to-USDC conversion tracking

## Core Functions

### Invoice Management Functions

#### `tdDepositInvoice()`
```solidity
function tdDepositInvoice(uint256 tradeDealId, uint256 tokenId) external
```

**Purpose**: Deposits an invoice NFT as collateral backing for a trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `tokenId` (uint256): ID of the invoice NFT to deposit

**Access Control**: Public (with validation)

**Process**:
1. Validates invoice deposit eligibility and ownership
2. Transfers invoice NFT to trade deal contract
3. Updates trade deal collateral backing
4. Records invoice in trade deal storage
5. Emits InvoiceDepositedToTradeDeal event

**Events**: `InvoiceDepositedToTradeDeal(tradeDealId, tokenId)`

**Example Usage**:
```solidity
// Deposit invoice as collateral for trade deal
uint256 tradeDealId = 1;
uint256 invoiceTokenId = 123;

// First approve the diamond to transfer the invoice
IERC721(invoiceContract).approve(diamond, invoiceTokenId);

// Deposit the invoice
ITradeDealOperations(diamond).tdDepositInvoice(tradeDealId, invoiceTokenId);
console.log("Invoice", invoiceTokenId, "deposited to trade deal", tradeDealId);
```

**Requirements**:
- Caller must own the invoice NFT
- Invoice must be eligible for the trade deal
- Trade deal must be active and accepting deposits
- Invoice must not already be deposited

---

#### `tdWithdrawInvoice()`
```solidity
function tdWithdrawInvoice(uint256 tradeDealId, uint256 tokenId) external
```

**Purpose**: Withdraws an invoice NFT from a trade deal back to the participant.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `tokenId` (uint256): ID of the invoice NFT to withdraw

**Access Control**: Trade deal participants only

**Process**:
1. Validates caller is a trade deal participant
2. Confirms invoice is deposited in the trade deal
3. Checks withdrawal eligibility (no outstanding obligations)
4. Transfers invoice NFT back to participant
5. Updates trade deal collateral records
6. Emits InvoiceWithdrawnFromTradeDeal event

**Events**: `InvoiceWithdrawnFromTradeDeal(tradeDealId, tokenId)`

**Example Usage**:
```solidity
// Withdraw invoice from trade deal
uint256 tradeDealId = 1;
uint256 invoiceTokenId = 123;

ITradeDealOperations(diamond).tdWithdrawInvoice(tradeDealId, invoiceTokenId);
console.log("Invoice", invoiceTokenId, "withdrawn from trade deal", tradeDealId);
```

**Requirements**:
- Caller must be a trade deal participant
- Invoice must be deposited in the trade deal
- No outstanding obligations against the invoice
- Trade deal must allow withdrawals

### USDC Funding Functions

#### `tdDepositUSDC()`
```solidity
function tdDepositUSDC(uint256 tradeDealId, uint256 amount) external
```

**Purpose**: Deposits USDC to fund a trade deal and receives collateral tokens in return.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal to fund
- `amount` (uint256): Amount of USDC to deposit

**Access Control**: Public (with validation)

**Process**:
1. Validates trade deal is active and accepting funding
2. Transfers USDC from depositor to trade deal
3. Calculates and mints collateral tokens for depositor
4. Updates trade deal funding status
5. Checks if trade deal is fully funded
6. Emits multiple events for tracking

**Events**: 
- `USDCDepositedToTradeDeal(tradeDealId, amount, depositor)`
- `CollateralTokensDistributed(tradeDealId, recipient, amount)`
- `TradeDealFullyFunded(tradeDealId, fundingTarget)` (if applicable)

**Example Usage**:
```solidity
// Fund trade deal with USDC
uint256 tradeDealId = 1;
uint256 fundingAmount = 10000 * 10**6; // 10,000 USDC

// First approve USDC transfer
IERC20(usdcToken).approve(diamond, fundingAmount);

// Deposit USDC and receive collateral tokens
ITradeDealOperations(diamond).tdDepositUSDC(tradeDealId, fundingAmount);
console.log("Deposited", fundingAmount, "USDC to trade deal", tradeDealId);
```

**Collateral Token Calculation**:
- Collateral tokens are minted based on the collateral-to-interest ratio
- Tokens represent proportional ownership in the trade deal
- Used for interest distribution and redemption rights

---

#### `tdWithdrawUSDC()`
```solidity
function tdWithdrawUSDC(uint256 tradeDealId, uint256 amount) external onlyOwner
```

**Purpose**: Withdraws USDC from a trade deal (admin function).

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `amount` (uint256): Amount of USDC to withdraw

**Access Control**: Owner only

**Process**:
1. Validates withdrawal amount and availability
2. Transfers USDC from trade deal to owner
3. Updates trade deal funding records
4. Emits USDCWithdrawnFromTradeDeal event

**Events**: `USDCWithdrawnFromTradeDeal(tradeDealId, amount)`

**Example Usage**:
```solidity
// Admin withdrawal of excess USDC
uint256 tradeDealId = 1;
uint256 withdrawAmount = 5000 * 10**6; // 5,000 USDC

ITradeDealOperations(diamond).tdWithdrawUSDC(tradeDealId, withdrawAmount);
console.log("Withdrew", withdrawAmount, "USDC from trade deal", tradeDealId);
```

### Funding Withdrawal Functions

#### `withdrawTradeDealFundingForBorrower()`
```solidity
function withdrawTradeDealFundingForBorrower(uint256 tradeDealId, address borrowerAddress) external onlyOwner
```

**Purpose**: Allows admin to withdraw funding on behalf of a borrower.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `borrowerAddress` (address): Address of the borrower receiving funds

**Access Control**: Owner only

**Process**:
1. Validates funding withdrawal eligibility
2. Confirms trade deal is fully funded
3. Transfers available funding to borrower
4. Updates trade deal status and records
5. Emits TradeDealFundingWithdrawn event

**Events**: `TradeDealFundingWithdrawn(tradeDealId, recipient, amount)`

**Example Usage**:
```solidity
// Admin withdraws funding for borrower
uint256 tradeDealId = 1;
address borrower = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;

ITradeDealOperations(diamond).withdrawTradeDealFundingForBorrower(tradeDealId, borrower);
console.log("Funding withdrawn for borrower:", borrower);
```

---

#### `withdrawTradeDealFundingSelf()`
```solidity
function withdrawTradeDealFundingSelf(uint256 tradeDealId) external
```

**Purpose**: Allows eligible borrowers to withdraw funding themselves.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal

**Access Control**: Validated borrowers only

**Process**:
1. Validates caller is authorized borrower for the trade deal
2. Confirms self-service withdrawal is enabled
3. Checks trade deal funding status and eligibility
4. Transfers available funding to caller
5. Updates trade deal records
6. Emits TradeDealFundingWithdrawn event

**Events**: `TradeDealFundingWithdrawn(tradeDealId, recipient, amount)`

**Example Usage**:
```solidity
// Borrower withdraws funding themselves
uint256 tradeDealId = 1;

ITradeDealOperations(diamond).withdrawTradeDealFundingSelf(tradeDealId);
console.log("Self-service funding withdrawal completed");
```

**Requirements**:
- Caller must be authorized borrower
- Trade deal must be fully funded
- Self-service withdrawal must be enabled
- No outstanding compliance issues

### Repayment Functions

#### `repayTradeDeal()`
```solidity
function repayTradeDeal(uint256 tradeDealId, uint256 amount) external
```

**Purpose**: Allows borrowers to repay trade deal obligations.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `amount` (uint256): Amount to repay (in USDC)

**Access Control**: Authorized repayers only

**Process**:
1. Validates repayment parameters and authorization
2. Transfers repayment amount from caller
3. Updates trade deal outstanding balance
4. Calculates interest and principal allocation
5. Updates repayment tracking and status
6. Emits TradeDealRepaid event

**Events**: `TradeDealRepaid(tradeDealId, repayer, amount, fullyRepaid)`

**Example Usage**:
```solidity
// Borrower makes partial repayment
uint256 tradeDealId = 1;
uint256 repaymentAmount = 5000 * 10**6; // 5,000 USDC

// First approve USDC transfer
IERC20(usdcToken).approve(diamond, repaymentAmount);

// Make repayment
ITradeDealOperations(diamond).repayTradeDeal(tradeDealId, repaymentAmount);
console.log("Repaid", repaymentAmount, "to trade deal", tradeDealId);
```

**Repayment Allocation**:
- Interest payments processed first
- Principal reduction after interest
- Full repayment triggers completion status
- Partial repayments update outstanding balance

---

#### `repayTradeDealForBorrower()`
```solidity
function repayTradeDealForBorrower(uint256 tradeDealId, address borrower, uint256 amount) external onlyOwner
```

**Purpose**: Allows admin to make repayments on behalf of borrowers.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `borrower` (address): Address of the borrower
- `amount` (uint256): Amount to repay

**Access Control**: Owner only

**Process**:
1. Validates repayment parameters
2. Processes repayment on behalf of borrower
3. Updates trade deal records and status
4. Handles interest and principal allocation
5. Emits TradeDealRepaid event

**Events**: `TradeDealRepaid(tradeDealId, repayer, amount, fullyRepaid)`

**Example Usage**:
```solidity
// Admin makes repayment for borrower
uint256 tradeDealId = 1;
address borrower = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;
uint256 repaymentAmount = 10000 * 10**6; // 10,000 USDC

ITradeDealOperations(diamond).repayTradeDealForBorrower(tradeDealId, borrower, repaymentAmount);
console.log("Admin repayment made for borrower:", borrower);
```

### Collateral Redemption Functions

#### `redeemCollateralTokens()`
```solidity
function redeemCollateralTokens(uint256 tradeDealId, uint256 collateralAmount) external
```

**Purpose**: Allows collateral token holders to redeem tokens for USDC.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `collateralAmount` (uint256): Amount of collateral tokens to redeem

**Access Control**: Collateral token holders only

**Process**:
1. Validates collateral redemption eligibility
2. Calculates USDC amount based on current redemption rate
3. Burns collateral tokens from caller
4. Transfers proportional USDC to caller
5. Updates trade deal collateral tracking
6. Emits CollateralTokensRedeemed event

**Events**: `CollateralTokensRedeemed(tradeDealId, redeemer, collateralAmount, usdcAmount)`

**Example Usage**:
```solidity
// Redeem collateral tokens for USDC
uint256 tradeDealId = 1;
uint256 collateralToRedeem = 1000 * 10**18; // 1,000 collateral tokens

ITradeDealOperations(diamond).redeemCollateralTokens(tradeDealId, collateralToRedeem);
console.log("Redeemed", collateralToRedeem, "collateral tokens");
```

**Redemption Calculation**:
- USDC amount = (collateral tokens × current USDC pool) ÷ total collateral tokens
- Redemption rate varies based on trade deal performance
- Interest distributions affect redemption value
- Early redemption may have different rates than maturity

## Integration Examples

### Complete Trade Deal Lifecycle
```solidity
// Comprehensive trade deal operations workflow
contract TradeDealWorkflow {
    struct TradeDealState {
        uint256 tradeDealId;
        uint256 fundingTarget;
        uint256 currentFunding;
        uint256 outstandingBalance;
        bool fullyFunded;
        bool repaid;
    }
    
    mapping(uint256 => TradeDealState) public tradeDealStates;
    
    function participateInTradeDeal(
        uint256 tradeDealId,
        uint256 usdcAmount,
        uint256[] memory invoiceTokenIds
    ) external {
        // Deposit invoices as collateral
        for (uint256 i = 0; i < invoiceTokenIds.length; i++) {
            ITradeDealOperations(diamond).tdDepositInvoice(tradeDealId, invoiceTokenIds[i]);
        }
        
        // Fund with USDC
        ITradeDealOperations(diamond).tdDepositUSDC(tradeDealId, usdcAmount);
        
        // Update state tracking
        TradeDealState storage state = tradeDealStates[tradeDealId];
        state.currentFunding += usdcAmount;
        
        emit ParticipationCompleted(tradeDealId, msg.sender, usdcAmount, invoiceTokenIds);
    }
    
    function borrowerWithdrawFunding(uint256 tradeDealId) external {
        require(tradeDealStates[tradeDealId].fullyFunded, "Not fully funded");
        
        // Borrower withdraws funding
        ITradeDealOperations(diamond).withdrawTradeDealFundingSelf(tradeDealId);
        
        // Update state
        tradeDealStates[tradeDealId].outstandingBalance = tradeDealStates[tradeDealId].currentFunding;
        
        emit FundingWithdrawn(tradeDealId, msg.sender);
    }
    
    function makeRepayment(uint256 tradeDealId, uint256 amount) external {
        // Make repayment
        ITradeDealOperations(diamond).repayTradeDeal(tradeDealId, amount);
        
        // Update state
        TradeDealState storage state = tradeDealStates[tradeDealId];
        if (amount >= state.outstandingBalance) {
            state.repaid = true;
            state.outstandingBalance = 0;
        } else {
            state.outstandingBalance -= amount;
        }
        
        emit RepaymentMade(tradeDealId, msg.sender, amount);
    }
    
    function redeemInvestment(uint256 tradeDealId, uint256 collateralAmount) external {
        require(tradeDealStates[tradeDealId].repaid, "Trade deal not repaid");
        
        // Redeem collateral tokens
        ITradeDealOperations(diamond).redeemCollateralTokens(tradeDealId, collateralAmount);
        
        emit InvestmentRedeemed(tradeDealId, msg.sender, collateralAmount);
    }
}
```

### Automated Repayment System
```solidity
// Automated repayment scheduling and processing
contract AutomatedRepaymentSystem {
    struct RepaymentSchedule {
        uint256 tradeDealId;
        uint256 totalAmount;
        uint256 installmentAmount;
        uint256 frequency; // in seconds
        uint256 nextPaymentDue;
        uint256 remainingPayments;
        bool active;
    }
    
    mapping(uint256 => RepaymentSchedule) public repaymentSchedules;
    mapping(address => uint256[]) public borrowerTradeDealIds;
    
    function setupRepaymentSchedule(
        uint256 tradeDealId,
        uint256 totalAmount,
        uint256 numberOfPayments,
        uint256 frequency
    ) external onlyAuthorizedBorrower(tradeDealId) {
        uint256 installmentAmount = totalAmount / numberOfPayments;
        
        repaymentSchedules[tradeDealId] = RepaymentSchedule({
            tradeDealId: tradeDealId,
            totalAmount: totalAmount,
            installmentAmount: installmentAmount,
            frequency: frequency,
            nextPaymentDue: block.timestamp + frequency,
            remainingPayments: numberOfPayments,
            active: true
        });
        
        borrowerTradeDealIds[msg.sender].push(tradeDealId);
        
        emit RepaymentScheduleCreated(tradeDealId, totalAmount, numberOfPayments, frequency);
    }
    
    function executeScheduledRepayment(uint256 tradeDealId) external {
        RepaymentSchedule storage schedule = repaymentSchedules[tradeDealId];
        
        require(schedule.active, "Schedule not active");
        require(block.timestamp >= schedule.nextPaymentDue, "Payment not due yet");
        require(schedule.remainingPayments > 0, "All payments completed");
        
        // Execute repayment
        ITradeDealOperations(diamond).repayTradeDeal(tradeDealId, schedule.installmentAmount);
        
        // Update schedule
        schedule.remainingPayments--;
        if (schedule.remainingPayments > 0) {
            schedule.nextPaymentDue += schedule.frequency;
        } else {
            schedule.active = false;
        }
        
        emit ScheduledRepaymentExecuted(tradeDealId, schedule.installmentAmount, schedule.remainingPayments);
    }
    
    function getOverduePayments(address borrower) external view returns (uint256[] memory) {
        uint256[] memory tradeDealIds = borrowerTradeDealIds[borrower];
        uint256[] memory overdueIds = new uint256[](tradeDealIds.length);
        uint256 overdueCount = 0;
        
        for (uint256 i = 0; i < tradeDealIds.length; i++) {
            RepaymentSchedule memory schedule = repaymentSchedules[tradeDealIds[i]];
            if (schedule.active && block.timestamp > schedule.nextPaymentDue) {
                overdueIds[overdueCount] = tradeDealIds[i];
                overdueCount++;
            }
        }
        
        // Resize array to actual count
        uint256[] memory result = new uint256[](overdueCount);
        for (uint256 i = 0; i < overdueCount; i++) {
            result[i] = overdueIds[i];
        }
        
        return result;
    }
}
```

### Collateral Management System
```solidity
// Advanced collateral tracking and management
contract CollateralManagementSystem {
    struct CollateralPosition {
        uint256 tradeDealId;
        uint256 collateralTokens;
        uint256 originalUSDCDeposit;
        uint256 accruedInterest;
        uint256 lastInterestUpdate;
    }
    
    mapping(address => mapping(uint256 => CollateralPosition)) public positions;
    mapping(uint256 => uint256) public totalCollateralTokens;
    mapping(uint256 => uint256) public totalUSDCPool;
    
    function trackCollateralDeposit(
        uint256 tradeDealId,
        address depositor,
        uint256 usdcAmount,
        uint256 collateralTokens
    ) external onlyAuthorized {
        CollateralPosition storage position = positions[depositor][tradeDealId];
        
        position.tradeDealId = tradeDealId;
        position.collateralTokens += collateralTokens;
        position.originalUSDCDeposit += usdcAmount;
        position.lastInterestUpdate = block.timestamp;
        
        totalCollateralTokens[tradeDealId] += collateralTokens;
        totalUSDCPool[tradeDealId] += usdcAmount;
        
        emit CollateralPositionUpdated(depositor, tradeDealId, position);
    }
    
    function calculateRedemptionValue(
        address holder,
        uint256 tradeDealId,
        uint256 collateralAmount
    ) external view returns (uint256 usdcAmount, uint256 interestAmount) {
        CollateralPosition memory position = positions[holder][tradeDealId];
        require(position.collateralTokens >= collateralAmount, "Insufficient collateral");
        
        // Calculate proportional USDC
        uint256 totalCollateral = totalCollateralTokens[tradeDealId];
        uint256 totalUSDC = totalUSDCPool[tradeDealId];
        
        usdcAmount = (collateralAmount * totalUSDC) / totalCollateral;
        
        // Calculate accrued interest
        uint256 proportionalInterest = (position.accruedInterest * collateralAmount) / position.collateralTokens;
        interestAmount = proportionalInterest;
    }
    
    function executeRedemption(
        uint256 tradeDealId,
        uint256 collateralAmount
    ) external {
        (uint256 usdcAmount, uint256 interestAmount) = calculateRedemptionValue(
            msg.sender,
            tradeDealId,
            collateralAmount
        );
        
        // Execute redemption through operations facet
        ITradeDealOperations(diamond).redeemCollateralTokens(tradeDealId, collateralAmount);
        
        // Update position tracking
        CollateralPosition storage position = positions[msg.sender][tradeDealId];
        position.collateralTokens -= collateralAmount;
        position.accruedInterest -= interestAmount;
        
        totalCollateralTokens[tradeDealId] -= collateralAmount;
        totalUSDCPool[tradeDealId] -= usdcAmount;
        
        emit RedemptionExecuted(msg.sender, tradeDealId, collateralAmount, usdcAmount, interestAmount);
    }
    
    function getPositionSummary(
        address holder,
        uint256 tradeDealId
    ) external view returns (
        uint256 collateralTokens,
        uint256 currentValue,
        uint256 originalDeposit,
        uint256 totalReturn,
        uint256 returnPercentage
    ) {
        CollateralPosition memory position = positions[holder][tradeDealId];
        
        collateralTokens = position.collateralTokens;
        originalDeposit = position.originalUSDCDeposit;
        
        if (collateralTokens > 0) {
            (currentValue,) = calculateRedemptionValue(holder, tradeDealId, collateralTokens);
            totalReturn = currentValue > originalDeposit ? currentValue - originalDeposit : 0;
            returnPercentage = originalDeposit > 0 ? (totalReturn * 10000) / originalDeposit : 0; // basis points
        }
    }
}
```

### Invoice Collateral Tracker
```solidity
// Track invoice collateral deposits and withdrawals
contract InvoiceCollateralTracker {
    struct InvoiceCollateral {
        uint256 tokenId;
        uint256 tradeDealId;
        address depositor;
        uint256 depositTime;
        uint256 invoiceValue;
        bool active;
    }
    
    mapping(uint256 => InvoiceCollateral) public invoiceCollaterals;
    mapping(uint256 => uint256[]) public tradeDealInvoices;
    mapping(address => uint256[]) public depositorInvoices;
    
    function trackInvoiceDeposit(
        uint256 tokenId,
        uint256 tradeDealId,
        address depositor,
        uint256 invoiceValue
    ) external onlyAuthorized {
        invoiceCollaterals[tokenId] = InvoiceCollateral({
            tokenId: tokenId,
            tradeDealId: tradeDealId,
            depositor: depositor,
            depositTime: block.timestamp,
            invoiceValue: invoiceValue,
            active: true
        });
        
        tradeDealInvoices[tradeDealId].push(tokenId);
        depositorInvoices[depositor].push(tokenId);
        
        emit InvoiceCollateralTracked(tokenId, tradeDealId, depositor, invoiceValue);
    }
    
    function executeInvoiceDeposit(
        uint256 tradeDealId,
        uint256 tokenId,
        uint256 invoiceValue
    ) external {
        // Execute deposit through operations facet
        ITradeDealOperations(diamond).tdDepositInvoice(tradeDealId, tokenId);
        
        // Track the deposit
        trackInvoiceDeposit(tokenId, tradeDealId, msg.sender, invoiceValue);
    }
    
    function executeInvoiceWithdrawal(
        uint256 tradeDealId,
        uint256 tokenId
    ) external {
        require(invoiceCollaterals[tokenId].depositor == msg.sender, "Not depositor");
        require(invoiceCollaterals[tokenId].active, "Invoice not active");
        
        // Execute withdrawal through operations facet
        ITradeDealOperations(diamond).tdWithdrawInvoice(tradeDealId, tokenId);
        
        // Update tracking
        invoiceCollaterals[tokenId].active = false;
        
        emit InvoiceWithdrawn(tokenId, tradeDealId, msg.sender);
    }
    
    function getTradeDealCollateralValue(uint256 tradeDealId) external view returns (uint256 totalValue) {
        uint256[] memory invoiceIds = tradeDealInvoices[tradeDealId];
        
        for (uint256 i = 0; i < invoiceIds.length; i++) {
            InvoiceCollateral memory collateral = invoiceCollaterals[invoiceIds[i]];
            if (collateral.active) {
                totalValue += collateral.invoiceValue;
            }
        }
    }
    
    function getDepositorCollateral(address depositor) external view returns (
        uint256[] memory tokenIds,
        uint256[] memory tradeDealIds,
        uint256[] memory values,
        bool[] memory activeStatus
    ) {
        uint256[] memory depositorTokenIds = depositorInvoices[depositor];
        
        tokenIds = new uint256[](depositorTokenIds.length);
        tradeDealIds = new uint256[](depositorTokenIds.length);
        values = new uint256[](depositorTokenIds.length);
        activeStatus = new bool[](depositorTokenIds.length);
        
        for (uint256 i = 0; i < depositorTokenIds.length; i++) {
            InvoiceCollateral memory collateral = invoiceCollaterals[depositorTokenIds[i]];
            tokenIds[i] = collateral.tokenId;
            tradeDealIds[i] = collateral.tradeDealId;
            values[i] = collateral.invoiceValue;
            activeStatus[i] = collateral.active;
        }
    }
}
```

## Events

### Invoice Management Events
```solidity
event InvoiceDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
event InvoiceWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
```

### USDC Funding Events
```solidity
event USDCDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 amount, address depositor);
event USDCWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 amount);
event TradeDealFullyFunded(uint256 indexed tradeDealId, uint256 fundingTarget);
event CollateralTokensDistributed(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
```

### Funding Withdrawal Events
```solidity
event TradeDealFundingWithdrawn(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
```

### Repayment Events
```solidity
event TradeDealRepaid(uint256 indexed tradeDealId, address indexed repayer, uint256 amount, bool fullyRepaid);
```

### Redemption Events
```solidity
event CollateralTokensRedeemed(uint256 indexed tradeDealId, address indexed redeemer, uint256 collateralAmount, uint256 usdcAmount);
```

## Security Considerations

### Access Control
- Participant validation for invoice operations
- Owner-only functions for admin operations
- Borrower authorization for funding withdrawals
- Collateral token holder validation for redemptions

### Financial Security
- Precise amount calculations and validations
- Overflow protection for large transactions
- Token transfer authorization checks
- Balance verification before operations

### Operational Security
- Trade deal state validation before operations
- Invoice ownership verification
- Funding availability checks
- Repayment authorization validation

## Gas Optimization

### Efficient Operations
- Batch invoice deposits when possible
- Optimize token transfer operations
- Minimize storage updates
- Use events for off-chain tracking

### Storage Optimization
- Pack related data efficiently
- Use mappings for participant tracking
- Cache frequently accessed values
- Optimize collateral calculations

## Error Handling

### Common Errors
- Insufficient balances or allowances
- Invalid trade deal states
- Unauthorized operations
- Invoice ownership mismatches
- Funding availability issues

### Best Practices
- Validate all inputs before processing
- Check authorization before