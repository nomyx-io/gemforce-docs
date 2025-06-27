# ITradeDeal Interface

## Overview

The `ITradeDeal.sol` defines the comprehensive interface for trade deal functionality within the Gemforce platform. This interface establishes the standard contract for creating, managing, and operating collateralized trade deals with invoice backing, USDC funding, interest distribution, and participant management.

## Interface Details

- **Interface Name**: `ITradeDeal`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Trade Deal Lifecycle Management
- Create and configure trade deals with flexible parameters
- Activate and deactivate trade deals
- Update trade deal settings and token addresses
- Comprehensive status tracking and reporting

### 🔹 Collateral and Funding Operations
- Invoice deposit and withdrawal as collateral
- USDC funding with automatic collateral token distribution
- Funding withdrawal for borrowers
- Collateral token redemption system

### 🔹 Interest and Fee Management
- Automated interest calculation and distribution
- Multi-pool interest allocation
- Fee distribution to configured receivers
- Interest token minting and management

### 🔹 Participant and Access Control
- Participant addition and removal
- Identity verification through claim topics
- Role-based access control
- Participant validation for operations

### 🔹 Repayment and Settlement
- Flexible repayment processing
- Partial and full repayment support
- Automated settlement calculations
- Repayment tracking and validation

## Core Functions

### Trade Deal Management Functions

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
) external returns (uint256)
```

**Purpose**: Creates a new trade deal with specified configuration parameters.

**Parameters**:
- `name` (string): Human-readable name for the trade deal
- `symbol` (string): Symbol for the trade deal's collateral token
- `interestRate` (uint256): Interest rate in basis points (100 = 1%)
- `collateralToInterestRatio` (uint256): Conversion ratio for collateral to interest tokens
- `requiredClaimTopics` (uint256[]): Array of claim topic IDs required for participation
- `collateralAddress` (address): Address of the collateral token contract
- `interestAddress` (address): Address of the interest token contract
- `usdcAddress` (address): Address of the USDC token contract
- `operationMode` (TradeDealLib.OperationMode): Operation mode for the trade deal

**Returns**:
- `uint256`: Unique identifier for the newly created trade deal

**Events**: `TradeDealCreated(tradeDealId, name, symbol, interestRate, collateralToInterestRatio, active, nftAddress, collateralAddress, interestAddress, usdcAddress, operationMode)`

**Example Usage**:
```solidity
// Create a new trade deal for invoice financing
string memory dealName = "Q1 2024 Invoice Financing";
string memory dealSymbol = "Q1IF";
uint256 interestRate = 800; // 8% annual interest
uint256 collateralRatio = 1000000; // 1:1 ratio (scaled)

uint256[] memory claimTopics = new uint256[](2);
claimTopics[0] = 1; // KYC verification
claimTopics[1] = 2; // Accredited investor

uint256 tradeDealId = ITradeDeal(diamond).createTradeDeal(
    dealName,
    dealSymbol,
    interestRate,
    collateralRatio,
    claimTopics,
    collateralTokenAddress,
    interestTokenAddress,
    usdcTokenAddress,
    TradeDealLib.OperationMode.STANDARD
);

console.log("Created trade deal with ID:", tradeDealId);
```

---

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
) external
```

**Purpose**: Updates the configuration of an existing trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal to update
- `name` (string): Updated name for the trade deal
- `symbol` (string): Updated symbol for the collateral token
- `interestRate` (uint256): Updated interest rate in basis points
- `collateralToInterestRatio` (uint256): Updated conversion ratio
- `collateralAddress` (address): Updated collateral token address
- `interestAddress` (address): Updated interest token address
- `usdcAddress` (address): Updated USDC token address

**Events**: `TradeDealUpdated(tradeDealId, name, symbol, interestRate, collateralToInterestRatio, active, collateralAddress, interestAddress, usdcAddress)`

**Example Usage**:
```solidity
// Update trade deal interest rate
uint256 tradeDealId = 1;
uint256 newInterestRate = 750; // Reduce to 7.5%

ITradeDeal(diamond).updateTradeDeal(
    tradeDealId,
    "Q1 2024 Invoice Financing - Updated",
    "Q1IF",
    newInterestRate,
    1000000, // Keep same ratio
    collateralTokenAddress,
    interestTokenAddress,
    usdcTokenAddress
);
```

---

#### `activateTradeDeal()` / `deactivateTradeDeal()`
```solidity
function activateTradeDeal(uint256 tradeDealId) external
function deactivateTradeDeal(uint256 tradeDealId) external
```

**Purpose**: Activates or deactivates a trade deal for participation and operations.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal

**Events**: 
- `TradeDealActivated(tradeDealId)`
- `TradeDealDeactivated(tradeDealId)`

**Example Usage**:
```solidity
// Activate trade deal for funding
uint256 tradeDealId = 1;
ITradeDeal(diamond).activateTradeDeal(tradeDealId);

// Later, deactivate when funding complete
ITradeDeal(diamond).deactivateTradeDeal(tradeDealId);
```

### Collateral and Funding Functions

#### `tdDepositInvoice()` / `tdWithdrawInvoice()`
```solidity
function tdDepositInvoice(uint256 tradeDealId, uint256 tokenId) external
function tdWithdrawInvoice(uint256 tradeDealId, uint256 tokenId) external
```

**Purpose**: Deposits or withdraws invoice NFTs as collateral for trade deals.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `tokenId` (uint256): ID of the invoice NFT

**Events**: 
- `InvoiceDepositedToTradeDeal(tradeDealId, tokenId)`
- `InvoiceWithdrawnFromTradeDeal(tradeDealId, tokenId)`

**Example Usage**:
```solidity
// Deposit invoice as collateral
uint256 tradeDealId = 1;
uint256 invoiceTokenId = 123;

// First approve the diamond to transfer the invoice
IERC721(invoiceContract).approve(diamond, invoiceTokenId);

// Deposit the invoice
ITradeDeal(diamond).tdDepositInvoice(tradeDealId, invoiceTokenId);
console.log("Invoice deposited as collateral");

// Later withdraw if needed
ITradeDeal(diamond).tdWithdrawInvoice(tradeDealId, invoiceTokenId);
```

---

#### `tdDepositUSDC()` / `tdWithdrawUSDC()`
```solidity
function tdDepositUSDC(uint256 tradeDealId, uint256 amount) external
function tdWithdrawUSDC(uint256 tradeDealId, uint256 amount) external
```

**Purpose**: Deposits or withdraws USDC for trade deal funding.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `amount` (uint256): Amount of USDC to deposit/withdraw

**Events**: 
- `USDCDepositedToTradeDeal(tradeDealId, amount, depositor)`
- `CollateralTokensDistributed(tradeDealId, recipient, amount)`
- `TradeDealFullyFunded(tradeDealId, fundingTarget)` (if applicable)
- `USDCWithdrawnFromTradeDeal(tradeDealId, amount)`

**Example Usage**:
```solidity
// Fund trade deal with USDC
uint256 tradeDealId = 1;
uint256 fundingAmount = 10000 * 10**6; // 10,000 USDC

// First approve USDC transfer
IERC20(usdcToken).approve(diamond, fundingAmount);

// Deposit USDC and receive collateral tokens
ITradeDeal(diamond).tdDepositUSDC(tradeDealId, fundingAmount);
console.log("Deposited USDC and received collateral tokens");
```

---

#### `withdrawTradeDealFundingForBorrower()`
```solidity
function withdrawTradeDealFundingForBorrower(uint256 tradeDealId, address borrowerAddress) external
```

**Purpose**: Allows admin to withdraw funding on behalf of a borrower.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `borrowerAddress` (address): Address of the borrower receiving funds

**Events**: `TradeDealFundingWithdrawn(tradeDealId, recipient, amount)`

**Example Usage**:
```solidity
// Admin withdraws funding for borrower
uint256 tradeDealId = 1;
address borrower = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;

ITradeDeal(diamond).withdrawTradeDealFundingForBorrower(tradeDealId, borrower);
console.log("Funding withdrawn for borrower");
```

### Interest and Fee Management Functions

#### `tdDistributeInterest()`
```solidity
function tdDistributeInterest(uint256 tradeDealId) external
```

**Purpose**: Distributes accumulated interest for a trade deal across pools and participants.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal

**Events**: `InterestDistributedForTradeDeal(tradeDealId, totalInterest, invoicePoolInterest, interestInterest, interestTokensMinted)`

**Example Usage**:
```solidity
// Distribute interest for trade deal
uint256 tradeDealId = 1;
ITradeDeal(diamond).tdDistributeInterest(tradeDealId);
console.log("Interest distributed for trade deal");
```

### Repayment Functions

#### `repayTradeDeal()` / `repayTradeDealForBorrower()`
```solidity
function repayTradeDeal(uint256 tradeDealId, uint256 amount) external
function repayTradeDealForBorrower(uint256 tradeDealId, address borrower, uint256 amount) external
```

**Purpose**: Processes repayments for trade deals, either by borrower or admin.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `amount` (uint256): Amount to repay
- `borrower` (address): Borrower address (for admin repayment)

**Events**: `TradeDealRepaid(tradeDealId, repayer, amount, fullyRepaid)`

**Example Usage**:
```solidity
// Borrower makes repayment
uint256 tradeDealId = 1;
uint256 repaymentAmount = 5000 * 10**6; // 5,000 USDC

// First approve USDC transfer
IERC20(usdcToken).approve(diamond, repaymentAmount);

// Make repayment
ITradeDeal(diamond).repayTradeDeal(tradeDealId, repaymentAmount);
console.log("Repayment made");
```

---

#### `redeemCollateralTokens()`
```solidity
function redeemCollateralTokens(uint256 tradeDealId, uint256 collateralAmount) external
```

**Purpose**: Redeems collateral tokens for USDC based on current redemption rate.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `collateralAmount` (uint256): Amount of collateral tokens to redeem

**Events**: `CollateralTokensRedeemed(tradeDealId, redeemer, collateralAmount, usdcAmount)`

**Example Usage**:
```solidity
// Redeem collateral tokens for USDC
uint256 tradeDealId = 1;
uint256 collateralToRedeem = 1000 * 10**18; // 1,000 collateral tokens

ITradeDeal(diamond).redeemCollateralTokens(tradeDealId, collateralToRedeem);
console.log("Collateral tokens redeemed for USDC");
```

### Participant Management Functions

#### `setTradeDealRequiredClaimTopics()` / `getTradeDealRequiredClaimTopics()`
```solidity
function setTradeDealRequiredClaimTopics(uint256 tradeDealId, uint256[] memory claimTopics) external
function getTradeDealRequiredClaimTopics(uint256 tradeDealId) external view returns (uint256[] memory)
```

**Purpose**: Sets or retrieves required identity claim topics for trade deal participation.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `claimTopics` (uint256[]): Array of claim topic IDs

**Events**: `TradeDealRequiredClaimTopicsSet(tradeDealId, claimTopics)`

**Example Usage**:
```solidity
// Set required claim topics
uint256 tradeDealId = 1;
uint256[] memory requiredClaims = new uint256[](3);
requiredClaims[0] = 1; // KYC verification
requiredClaims[1] = 2; // Accredited investor
requiredClaims[2] = 3; // Geographic eligibility

ITradeDeal(diamond).setTradeDealRequiredClaimTopics(tradeDealId, requiredClaims);

// Get current requirements
uint256[] memory currentClaims = ITradeDeal(diamond).getTradeDealRequiredClaimTopics(tradeDealId);
```

---

#### `isTradeDealParticipant()`
```solidity
function isTradeDealParticipant(uint256 tradeDealId, address user) external view returns (bool)
```

**Purpose**: Checks if an address is a verified participant in a trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `user` (address): Address to check

**Returns**:
- `bool`: True if user is a verified participant

**Example Usage**:
```solidity
// Check if user is a participant
uint256 tradeDealId = 1;
address user = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;

bool isParticipant = ITradeDeal(diamond).isTradeDealParticipant(tradeDealId, user);
if (isParticipant) {
    console.log("User is verified participant");
} else {
    console.log("User is not a participant");
}
```

### Query Functions

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
- `tradeDealId` (uint256): ID of the trade deal

**Returns**: Trade deal configuration details

**Example Usage**:
```solidity
// Get trade deal information
uint256 tradeDealId = 1;
(
    string memory name,
    string memory symbol,
    uint256 interestRate,
    uint256 collateralRatio,
    bool active,
    TradeDealLib.OperationMode mode
) = ITradeDeal(diamond).getTradeDealInfo(tradeDealId);

console.log("Trade Deal:", name);
console.log("Interest Rate:", interestRate, "basis points");
console.log("Active:", active);
```

---

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

**Purpose**: Retrieves comprehensive status information for a trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal

**Returns**: Complete financial and operational status

**Example Usage**:
```solidity
// Get complete trade deal status
uint256 tradeDealId = 1;
(
    uint256 fundingTarget,
    uint256 currentBalance,
    bool isFunded,
    bool isFundingWithdrawn,
    uint256 totalDebt,
    uint256 repaidAmount,
    bool isRepaid
) = ITradeDeal(diamond).getTradeDealFullStatus(tradeDealId);

console.log("Funding Target:", fundingTarget);
console.log("Current Balance:", currentBalance);
console.log("Is Funded:", isFunded);
console.log("Total Debt:", totalDebt);
console.log("Repaid Amount:", repaidAmount);
console.log("Is Fully Repaid:", isRepaid);
```

---

#### `getAllTradeDealIds()`
```solidity
function getAllTradeDealIds() external view returns (uint256[] memory)
```

**Purpose**: Returns all trade deal IDs in the system.

**Returns**: Array of trade deal IDs

**Example Usage**:
```solidity
// Get all trade deal IDs
uint256[] memory allDeals = ITradeDeal(diamond).getAllTradeDealIds();
console.log("Total trade deals:", allDeals.length);

for (uint256 i = 0; i < allDeals.length; i++) {
    console.log("Trade Deal ID:", allDeals[i]);
}
```

### Status Check Functions

#### `isTradeDealFunded()` / `isTradeDealRepaid()`
```solidity
function isTradeDealFunded(uint256 tradeDealId) external view returns (bool)
function isTradeDealRepaid(uint256 tradeDealId) external view returns (bool)
```

**Purpose**: Quick status checks for funding and repayment status.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal

**Returns**: Boolean status

**Example Usage**:
```solidity
// Check trade deal status
uint256 tradeDealId = 1;

bool isFunded = ITradeDeal(diamond).isTradeDealFunded(tradeDealId);
bool isRepaid = ITradeDeal(diamond).isTradeDealRepaid(tradeDealId);

console.log("Trade Deal Funded:", isFunded);
console.log("Trade Deal Repaid:", isRepaid);
```

## Integration Examples

### Complete Trade Deal Workflow
```solidity
// Comprehensive trade deal integration example
contract TradeDealWorkflow {
    ITradeDeal public tradeDeal;
    
    struct WorkflowState {
        uint256 tradeDealId;
        uint256 phase; // 0: Created, 1: Funded, 2: Active, 3: Repaid
        uint256 totalFunding;
        uint256 totalRepaid;
        address[] participants;
    }
    
    mapping(uint256 => WorkflowState) public workflows;
    
    event WorkflowPhaseChanged(uint256 indexed tradeDealId, uint256 newPhase);
    
    constructor(address _tradeDeal) {
        tradeDeal = ITradeDeal(_tradeDeal);
    }
    
    function createAndFundTradeDeal(
        string memory name,
        string memory symbol,
        uint256 interestRate,
        uint256 fundingAmount,
        uint256[] memory invoiceTokenIds
    ) external returns (uint256 tradeDealId) {
        // Create trade deal
        uint256[] memory claimTopics = new uint256[](1);
        claimTopics[0] = 1; // KYC required
        
        tradeDealId = tradeDeal.createTradeDeal(
            name,
            symbol,
            interestRate,
            1000000, // 1:1 collateral ratio
            claimTopics,
            collateralTokenAddress,
            interestTokenAddress,
            usdcTokenAddress,
            TradeDealLib.OperationMode.STANDARD
        );
        
        // Initialize workflow
        workflows[tradeDealId] = WorkflowState({
            tradeDealId: tradeDealId,
            phase: 0,
            totalFunding: 0,
            totalRepaid: 0,
            participants: new address[](0)
        });
        
        // Activate trade deal
        tradeDeal.activateTradeDeal(tradeDealId);
        
        // Deposit invoices as collateral
        for (uint256 i = 0; i < invoiceTokenIds.length; i++) {
            tradeDeal.tdDepositInvoice(tradeDealId, invoiceTokenIds[i]);
        }
        
        // Fund the trade deal
        tradeDeal.tdDepositUSDC(tradeDealId, fundingAmount);
        workflows[tradeDealId].totalFunding = fundingAmount;
        workflows[tradeDealId].phase = 1;
        
        emit WorkflowPhaseChanged(tradeDealId, 1);
    }
    
    function processRepayment(uint256 tradeDealId, uint256 amount) external {
        WorkflowState storage workflow = workflows[tradeDealId];
        require(workflow.phase >= 1, "Trade deal not funded");
        
        // Process repayment
        tradeDeal.repayTradeDeal(tradeDealId, amount);
        workflow.totalRepaid += amount;
        
        // Check if fully repaid
        bool isRepaid = tradeDeal.isTradeDealRepaid(tradeDealId);
        if (isRepaid && workflow.phase < 3) {
            workflow.phase = 3;
            emit WorkflowPhaseChanged(tradeDealId, 3);
        }
    }
    
    function distributeInterestAndFees(uint256 tradeDealId) external {
        WorkflowState storage workflow = workflows[tradeDealId];
        require(workflow.phase >= 1, "Trade deal not funded");
        
        // Distribute interest
        tradeDeal.tdDistributeInterest(tradeDealId);
        
        // Update phase if needed
        if (workflow.phase == 1) {
            workflow.phase = 2; // Active phase
            emit WorkflowPhaseChanged(tradeDealId, 2);
        }
    }
    
    function getWorkflowStatus(uint256 tradeDealId) external view returns (
        uint256 phase,
        uint256 fundingProgress,
        uint256 repaymentProgress,
        bool canRedeem
    ) {
        WorkflowState memory workflow = workflows[tradeDealId];
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        phase = workflow.phase;
        fundingProgress = fundingTarget > 0 ? (currentBalance * 100) / fundingTarget : 0;
        repaymentProgress = totalDebt > 0 ? (repaidAmount * 100) / totalDebt : 0;
        canRedeem = isRepaid && workflow.totalFunding > 0;
    }
}
```

### Trade Deal Analytics Dashboard
```solidity
// Analytics and reporting for trade deals
contract TradeDealAnalytics {
    ITradeDeal public tradeDeal;
    
    struct AnalyticsData {
        uint256 totalDeals;
        uint256 activeDeals;
        uint256 fundedDeals;
        uint256 repaidDeals;
        uint256 totalVolume;
        uint256 totalInterestPaid;
        uint256 averageInterestRate;
    }
    
    mapping(uint256 => uint256) public dealCreationTime;
    mapping(uint256 => uint256) public dealFundingTime;
    mapping(uint256 => uint256) public dealRepaymentTime;
    
    event AnalyticsUpdated(AnalyticsData data);
    
    function updateAnalytics() external returns (AnalyticsData memory data) {
        uint256[] memory allDeals = tradeDeal.getAllTradeDealIds();
        
        data.totalDeals = allDeals.length;
        uint256 totalInterestRate = 0;
        
        for (uint256 i = 0; i < allDeals.length; i++) {
            uint256 dealId = allDeals[i];
            
            (
                string memory name,
                string memory symbol,
                uint256 interestRate,
                uint256 collateralRatio,
                bool active,
                TradeDealLib.OperationMode mode
            ) = tradeDeal.getTradeDealInfo(dealId);
            
            if (active) {
                data.activeDeals++;
            }
            
            totalInterestRate += interestRate;
            
            (
                uint256 fundingTarget,
                uint256 currentBalance,
                bool isFunded,
                bool isFundingWithdrawn,
                uint256 totalDebt,
                uint256 repaidAmount,
                bool isRepaid
            ) = tradeDeal.getTradeDealFullStatus(dealId);
            
            if (isFunded) {
                data.fundedDeals++;
                data.totalVolume += fundingTarget;
            }
            
            if (isRepaid) {
                data.repaidDeals++;
                data.totalInterestPaid += (repaidAmount > fundingTarget ? repaidAmount - fundingTarget : 0);
            }
        }
        
        if (data.totalDeals > 0) {
            data.averageInterestRate = totalInterestRate / data.totalDeals;
        }
        
        emit AnalyticsUpdated(data);
    }
    
    function getDealPerformanceMetrics(uint256 tradeDealId) external view returns (
        uint256 timeToFunding,
        uint256 timeToRepayment,
        uint256 actualInterestRate,
        uint256 performanceScore
    ) {
        uint256 creationTime = dealCreationTime[tradeDealId];
        uint256 fundingTime = dealFundingTime[tradeDealId];
        uint256 repaymentTime = dealRepaymentTime[tradeDealId];
        
        if (fundingTime > creationTime) {
            timeToFunding = fundingTime - creationTime;
        }
        
        if (repaymentTime > fundingTime && fundingTime > 0) {
            timeToRepayment = repaymentTime - fundingTime;
        }
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        if (isRepaid && fundingTarget > 0) {
            actualInterestRate = ((repaidAmount - fundingTarget) * 10000) / fundingTarget;
            
            // Calculate performance score (0-100)
            performanceScore = 100;
            if (timeToFunding > 7 days) performanceScore -= 20;
            if (timeToRepayment > 90 days) performanceScore -= 30;
            if (!isRepaid) performanceScore -= 50;
        }
    }
}
```

### Automated Trade Deal Manager
```solidity
// Automated management for trade deal operations
contract AutomatedTradeDealManager {
    ITradeDeal public tradeDeal;
    
    struct AutomationConfig {
        bool autoDistributeInterest;
        uint256 interestDistributionFrequency;
        uint256 lastInterestDistribution;
        bool autoProcessRepayments;
        uint256 gracePeriod;
        bool autoRedemption;
    }
    
    mapping(uint256 => AutomationConfig) public automationConfigs;
    mapping(uint256 => uint256) public nextScheduledAction;
    
    event AutomationConfigured(uint256 indexed tradeDealId, AutomationConfig config);
    event AutomatedActionExecuted(uint256 indexed tradeDealId, string action);
    
    function configureAutomation(
        uint256 tradeDealId,
        AutomationConfig memory config
    ) external onlyAuthorized {
        automationConfigs[tradeDealId] = config;
        
        if (config.autoDistributeInterest) {
            nextScheduledAction[tradeDealId] = block.timestamp + config.interestDistributionFrequency;
        }
        
        emit AutomationConfigured(tradeDealId, config);
    }
    
    function executeScheduledActions(uint256[] memory tradeDealIds) external {
        for (uint256 i = 0; i < tradeDealIds.length; i++) {
            uint256 tradeDealId = tradeDealIds[i];
            AutomationConfig memory config = automationConfigs[tradeDealId];
            
            // Auto-distribute interest
            if (config.autoDistributeInterest && 
                block.timestamp >= nextScheduledAction[tradeDealId]) {
                
                tradeDeal.tdDistributeInterest(tradeDealId);
                nextScheduledAction[tradeDealId] = block.timestamp + config.interestDistributionFrequency;
                
                emit AutomatedActionExecuted(tradeDealId, "interest_distribution");
            }
            
            // Auto-process repayments (would need additional logic)
            if (config.autoProcessRepayments) {
                _processAutomaticRepayments(tradeDealId, config);
            }
            
            // Auto-redemption for completed deals
            if (config.autoRedemption) {
                _processAutomaticRedemption(tradeDealId);
            }
        }
    }
    
    function _processAutomaticRepayments(uint256 tradeDealId, AutomationConfig memory config) internal {
        // Implementation would check for due repayments and process them
        // This would integrate with external payment systems or scheduled transfers
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        if (isFunded && !isRepaid) {
            // Check if repayment is due and process if available
            // This would require integration with external systems
            
            totalRepaid: 0,
            participants: new address[](0)
        });
        
        // Activate trade deal
        tradeDeal.activateTradeDeal(tradeDealId);
        
        // Deposit invoices as collateral
        for (uint256 i = 0; i < invoiceTokenIds.length; i++) {
            tradeDeal.tdDepositInvoice(tradeDealId, invoiceTokenIds[i]);
        }
        
        // Fund the trade deal
        tradeDeal.tdDepositUSDC(tradeDealId, fundingAmount);
        workflows[tradeDealId].totalFunding = fundingAmount;
        workflows[tradeDealId].phase = 1;
        
        emit WorkflowPhaseChanged(tradeDealId, 1);
    }
    
    function processRepayment(uint256 tradeDealId, uint256 amount) external {
        WorkflowState storage workflow = workflows[tradeDealId];
        require(workflow.phase >= 1, "Trade deal not funded");
        
        // Process repayment
        tradeDeal.repayTradeDeal(tradeDealId, amount);
        workflow.totalRepaid += amount;
        
        // Check if fully repaid
        bool isRepaid = tradeDeal.isTradeDealRepaid(tradeDealId);
        if (isRepaid && workflow.phase < 3) {
            workflow.phase = 3;
            emit WorkflowPhaseChanged(tradeDealId, 3);
        }
    }
    
    function distributeInterestAndFees(uint256 tradeDealId) external {
        WorkflowState storage workflow = workflows[tradeDealId];
        require(workflow.phase >= 1, "Trade deal not funded");
        
        // Distribute interest
        tradeDeal.tdDistributeInterest(tradeDealId);
        
        // Update phase if needed
        if (workflow.phase == 1) {
            workflow.phase = 2; // Active phase
            emit WorkflowPhaseChanged(tradeDealId, 2);
        }
    }
    
    function getWorkflowStatus(uint256 tradeDealId) external view returns (
        uint256 phase,
        uint256 fundingProgress,
        uint256 repaymentProgress,
        bool canRedeem
    ) {
        WorkflowState memory workflow = workflows[tradeDealId];
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        phase = workflow.phase;
        fundingProgress = fundingTarget > 0 ? (currentBalance * 100) / fundingTarget : 0;
        repaymentProgress = totalDebt > 0 ? (repaidAmount * 100) / totalDebt : 0;
        canRedeem = isRepaid && workflow.totalFunding > 0;
    }
}
```

### Trade Deal Analytics Dashboard
```solidity
// Analytics and reporting for trade deals
contract TradeDealAnalytics {
    ITradeDeal public tradeDeal;
    
    struct AnalyticsData {
        uint256 totalDeals;
        uint256 activeDeals;
        uint256 fundedDeals;
        uint256 repaidDeals;
        uint256 totalVolume;
        uint256 totalInterestPaid;
        uint256 averageInterestRate;
    }
    
    mapping(uint256 => uint256) public dealCreationTime;
    mapping(uint256 => uint256) public dealFundingTime;
    mapping(uint256 => uint256) public dealRepaymentTime;
    
    event AnalyticsUpdated(AnalyticsData data);
    
    function updateAnalytics() external returns (AnalyticsData memory data) {
        uint256[] memory allDeals = tradeDeal.getAllTradeDealIds();
        
        data.totalDeals = allDeals.length;
        uint256 totalInterestRate = 0;
        
        for (uint256 i = 0; i < allDeals.length; i++) {
            uint256 dealId = allDeals[i];
            
            (
                string memory name,
                string memory symbol,
                uint256 interestRate,
                uint256 collateralRatio,
                bool active,
                TradeDealLib.OperationMode mode
            ) = tradeDeal.getTradeDealInfo(dealId);
            
            if (active) {
                data.activeDeals++;
            }
            
            totalInterestRate += interestRate;
            
            (
                uint256 fundingTarget,
                uint256 currentBalance,
                bool isFunded,
                bool isFundingWithdrawn,
                uint256 totalDebt,
                uint256 repaidAmount,
                bool isRepaid
            ) = tradeDeal.getTradeDealFullStatus(dealId);
            
            if (isFunded) {
                data.fundedDeals++;
                data.totalVolume += fundingTarget;
            }
            
            if (isRepaid) {
                data.repaidDeals++;
                data.totalInterestPaid += (repaidAmount > fundingTarget ? repaidAmount - fundingTarget : 0);
            }
        }
        
        if (data.totalDeals > 0) {
            data.averageInterestRate = totalInterestRate / data.totalDeals;
        }
        
        emit AnalyticsUpdated(data);
    }
    
    function getDealPerformanceMetrics(uint256 tradeDealId) external view returns (
        uint256 timeToFunding,
        uint256 timeToRepayment,
        uint256 actualInterestRate,
        uint256 performanceScore
    ) {
        uint256 creationTime = dealCreationTime[tradeDealId];
        uint256 fundingTime = dealFundingTime[tradeDealId];
        uint256 repaymentTime = dealRepaymentTime[tradeDealId];
        
        if (fundingTime > creationTime) {
            timeToFunding = fundingTime - creationTime;
        }
        
        if (repaymentTime > fundingTime && fundingTime > 0) {
            timeToRepayment = repaymentTime - fundingTime;
        }
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        if (isRepaid && fundingTarget > 0) {
            actualInterestRate = ((repaidAmount - fundingTarget) * 10000) / fundingTarget;
            
            // Calculate performance score (0-100)
            performanceScore = 100;
            if (timeToFunding > 7 days) performanceScore -= 20;
            if (timeToRepayment > 90 days) performanceScore -= 30;
            if (!isRepaid) performanceScore -= 50;
        }
    }
}
```

### Automated Trade Deal Manager
```solidity
// Automated management for trade deal operations
contract AutomatedTradeDealManager {
    ITradeDeal public tradeDeal;
    
    struct AutomationConfig {
        bool autoDistributeInterest;
        uint256 interestDistributionFrequency;
        uint256 lastInterestDistribution;
        bool autoProcessRepayments;
        uint256 gracePeriod;
        bool autoRedemption;
    }
    
    mapping(uint256 => AutomationConfig) public automationConfigs;
    mapping(uint256 => uint256) public nextScheduledAction;
    
    event AutomationConfigured(uint256 indexed tradeDealId, AutomationConfig config);
    event AutomatedActionExecuted(uint256 indexed tradeDealId, string action);
    
    function configureAutomation(
        uint256 tradeDealId,
        AutomationConfig memory config
    ) external onlyAuthorized {
        automationConfigs[tradeDealId] = config;
        
        if (config.autoDistributeInterest) {
            nextScheduledAction[tradeDealId] = block.timestamp + config.interestDistributionFrequency;
        }
        
        emit AutomationConfigured(tradeDealId, config);
    }
    
    function executeScheduledActions(uint256[] memory tradeDealIds) external {
        for (uint256 i = 0; i < tradeDealIds.length; i++) {
            uint256 tradeDealId = tradeDealIds[i];
            AutomationConfig memory config = automationConfigs[tradeDealId];
            
            // Auto-distribute interest
            if (config.autoDistributeInterest && 
                block.timestamp >= nextScheduledAction[tradeDealId]) {
                
                tradeDeal.tdDistributeInterest(tradeDealId);
                nextScheduledAction[tradeDealId] = block.timestamp + config.interestDistributionFrequency;
                
                emit AutomatedActionExecuted(tradeDealId, "interest_distribution");
            }
            
            // Auto-process repayments (would need additional logic)
            if (config.autoProcessRepayments) {
                _processAutomaticRepayments(tradeDealId, config);
            }
            
            // Auto-redemption for completed deals
            if (config.autoRedemption) {
                _processAutomaticRedemption(tradeDealId);
            }
        }
    }
    
    function _processAutomaticRepayments(uint256 tradeDealId, AutomationConfig memory config) internal {
        // Implementation would check for due repayments and process them
        // This would integrate with external payment systems or scheduled transfers
        
        (
            uint256 fundingTarget,
            uint256 currentBalance,
            bool isFunded,
            bool isFundingWithdrawn,
            uint256 totalDebt,
            uint256 repaidAmount,
            bool isRepaid
        ) = tradeDeal.getTradeDealFullStatus(tradeDealId);
        
        if (isFunded && !isRepaid) {
            // Check if repayment is due and process if available
            // This would require integration with external systems
            emit AutomatedActionExecuted(tradeDealId, "repayment_check");
        }
    }
    
    function _processAutomaticRedemption(uint256 tradeDealId) internal {
        bool isRepaid = tradeDeal.isTradeDealRepaid(tradeDealId);
        
        if (isRepaid) {
            // Process automatic redemption for eligible participants
            // This would require additional participant tracking
            emit AutomatedActionExecuted(tradeDealId, "auto_redemption");
        }
    }
}
```

## Events

### Trade Deal Lifecycle Events
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

event TradeDealActivated(uint256 indexed tradeDealId);
event TradeDealDeactivated(uint256 indexed tradeDealId);
```

### Participant Management Events
```solidity
event TradeDealParticipantAdded(uint256 indexed tradeDealId, address indexed participant);
event TradeDealParticipantRemoved(uint256 indexed tradeDealId, address indexed participant);
event TradeDealRequiredClaimTopicsSet(uint256 indexed tradeDealId, uint256[] claimTopics);
```

### Collateral and Funding Events
```solidity
event InvoiceDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
event InvoiceWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 indexed tokenId);
event USDCDepositedToTradeDeal(uint256 indexed tradeDealId, uint256 amount, address depositor);
event USDCWithdrawnFromTradeDeal(uint256 indexed tradeDealId, uint256 amount);
event TradeDealFullyFunded(uint256 indexed tradeDealId, uint256 fundingTarget);
event CollateralTokensDistributed(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
```

### Interest and Fee Events
```solidity
event InterestDistributedForTradeDeal(
    uint256 indexed tradeDealId,
    uint256 totalInterest,
    uint256 invoicePoolInterest,
    uint256 interestInterest,
    uint256 interestTokensMinted
);

event TradeDealFeesDistributed(
    uint256 indexed tradeDealId,
    address[] feeReceivers,
    uint256[] feeAmounts
);
```

### Repayment and Settlement Events
```solidity
event TradeDealFundingWithdrawn(uint256 indexed tradeDealId, address indexed recipient, uint256 amount);
event TradeDealRepaid(uint256 indexed tradeDealId, address indexed repayer, uint256 amount, bool fullyRepaid);
event CollateralTokensRedeemed(uint256 indexed tradeDealId, address indexed redeemer, uint256 collateralAmount, uint256 usdcAmount);
```

## Security Considerations

### Access Control
- Owner-only functions for trade deal creation and configuration
- Participant validation for operations and access
- Identity verification through claim topics
- Role-based permissions for administrative functions

### Financial Security
- Precise amount calculations and validations
- Overflow protection for large transactions
- Secure token transfer mechanisms
- Interest calculation accuracy and validation

### Operational Security
- Trade deal state validation before operations
- Invoice ownership verification
- Funding availability checks
- Repayment authorization validation

## Gas Optimization

### Efficient Operations
- Batch operations for multiple trade deals
- Optimized storage layout for trade deal data
- Minimal external calls in view functions
- Efficient event emission patterns

### Storage Optimization
- Packed storage structures where possible
- Efficient mapping usage for participant tracking
- Optimized array operations for bulk queries
- Cached calculations for frequently accessed data

## Error Handling

### Common Errors
- Trade deal not found or inactive
- Insufficient permissions or authorization
- Invalid amounts or parameters
- Funding or repayment validation failures
- Participant verification failures

### Best Practices
- Validate all inputs before processing
- Check trade deal state before operations
- Verify participant eligibility for operations
- Handle token transfer failures gracefully
- Provide clear error messages for debugging

## Testing Considerations

### Unit Tests
- Interface compliance verification
- Function parameter validation
- Return value accuracy testing
- Event emission verification
- Error condition handling

### Integration Tests
- Complete trade deal lifecycle workflows
- Multi-participant scenarios
- Interest distribution and fee calculations
- Repayment and redemption processes
- Cross-facet integration testing

## Related Documentation

- [TradeDealManagementFacet](../facets/trade-deal-management-facet.md) - Core trade deal operations
- [TradeDealAdminFacet](../facets/trade-deal-admin-facet.md) - Administrative functions
- [TradeDealOperationsFacet](../facets/trade-deal-operations-facet.md) - Operational functions
- [TradeDealLib](../libraries/trade-deal-lib.md) - Trade deal utilities
- [CollateralTokenFactoryFacet](../facets/collateral-token-factory-facet.md) - Token factory integration
- [Trade Deal Guide](../../developer-guides/automated-testing-setup.md) - Implementation guide

---

*This interface defines the comprehensive contract for trade deal functionality within the Gemforce platform, providing standardized operations for collateralized finance instruments with invoice backing and automated settlement.*