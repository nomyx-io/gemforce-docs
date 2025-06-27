# TradeDealOperationsFacet

The `TradeDealOperationsFacet` is a core functional component within the Gemforce Diamond smart contract system, dedicated to the lifecycle management and execution of Trade Deals. It provides the necessary functions for creating, updating, settling, and liquidating trade deals, acting as the primary interface for users to interact with their trade agreements.

## Purpose

The primary purpose of the `TradeDealOperationsFacet` is to facilitate secure and transparent digital asset financing through collateralized trade deals. It ensures that trade agreements are executed according to predefined terms, provides mechanisms for collateral management, and supports both successful settlement and automated liquidation in case of default. This facet is crucial for the platform's ability to support various forms of on-chain asset-backed lending or financing.

## Key Features

*   **Trade Deal Creation**: Allows parties to initiate new trade deals with specified terms, collateral, and repayment schedules.
*   **Collateral Management**: Handles the locking and unlocking of collateral tokens associated with trade deals.
*   **Settlement and Repayment**: Enables the borrower to repay the loan and the lender to claim their principal and interest.
*   **Liquidation Mechanism**: Provides a process for lenders to liquidate collateral if a borrower defaults on their repayment obligations.
*   **Status Tracking**: Maintains the current state of each trade deal (e.g., active, settled, liquidated).
*   **Permissioned Actions**: Ensures that only authorized parties (borrower, lender, or designated liquidator) can perform specific actions on a trade deal.
*   **Event Emission**: Emits detailed events for every stage of a trade deal's lifecycle, providing complete transparency and auditability.

## Functions

### `createTradeDeal(address _lender, address _collateralToken, uint256 _collateralAmount, uint256 _loanAmount, uint256 _repaymentAmount, uint256 _settlementTime, uint256 _gracePeriod, bytes32 _invoiceNFTId)`

Creates a new trade deal. Callable by either party or an authorized third party.

*   **Parameters**:
    *   `_lender` (address): The address of the lender.
    *   `_collateralToken` (address): The address of the ERC-20 token used as collateral.
    *   `_collateralAmount` (uint256): The amount of collateral token to lock.
    *   `_loanAmount` (uint256): The amount of the loan disbursed by the lender.
    *   `_repaymentAmount` (uint256): The total amount to be repaid by the borrower (principal + interest).
    *   `_settlementTime` (uint256): The timestamp by which the loan must be repaid.
    *   `_gracePeriod` (uint256): Additional time after `_settlementTime` before collateral can be liquidated.
    *   `_invoiceNFTId` (bytes32): A unique identifier for the invoice NFT associated with this trade deal.
*   **Requirements**:
    *   `_collateralToken` must be a supported collateral token (checked via `TradeDealAdminFacet`).
    *   `_collateralAmount` must be transferred to this facet prior to or during this call.
*   **Emits**: `TradeDealCreated(bytes32 indexed tradeDealId, address indexed borrower, address indexed lender, ...)`

### `repayTradeDeal(bytes32 _tradeDealId)`

Allows the borrower to repay the loan and unlock their collateral.

*   **Parameters**:
    *   `_tradeDealId` (bytes32): The ID of the trade deal to repay.
*   **Requirements**:
    *   Caller must be the borrower of the `_tradeDealId`.
    *   The required `_repaymentAmount` must be sent to this facet (either via `msg.value` or ERC-20 `transferFrom`).
    *   Trade deal must be in an active state.
*   **Emits**: `TradeDealRepaid(bytes32 indexed tradeDealId)`

### `liquidateTradeDeal(bytes32 _tradeDealId)`

Allows the lender or an authorized liquidator to claim the collateral if the borrower defaults.

*   **Parameters**:
    *   `_tradeDealId` (bytes32): The ID of the trade deal to liquidate.
*   **Requirements**:
    *   Caller must be the lender or an authorized liquidator.
    *   The trade deal must be past its `_settlementTime` + `_gracePeriod`.
    *   Trade deal must be in an active state.
*   **Emits**: `TradeDealLiquidated(bytes32 indexed tradeDealId, address indexed liquidator)`

### `getTradeDeal(bytes32 _tradeDealId)`

Retrieves the current state and parameters of a trade deal.

*   **Parameters**:
    *   `_tradeDealId` (bytes32): The ID of the trade deal.
*   **Returns**: (`TradeDealLib.TradeDeal memory`) A struct containing all details of the trade deal.

## Events

### `TradeDealCreated(bytes32 indexed tradeDealId, address indexed borrower, address indexed lender, address collateralToken, uint256 collateralAmount, uint256 loanAmount, uint256 repaymentAmount, uint256 settlementTime, uint256 gracePeriod, bytes32 invoiceNFTId)`

Emitted when a new trade deal is successfully created.

*   **Parameters**: Comprehensive details of the newly created trade deal.

### `TradeDealRepaid(bytes32 indexed tradeDealId)`

Emitted when a trade deal is successfully repaid by the borrower.

*   **Parameters**:
    *   `tradeDealId` (bytes32): The ID of the repaid trade deal.

### `TradeDealLiquidated(bytes32 indexed tradeDealId, address indexed liquidator)`

Emitted when a trade deal is successfully liquidated due to borrower default.

*   **Parameters**:
    *   `tradeDealId` (bytes32): The ID of the liquidated trade deal.
    *   `liquidator` (address): The address that executed the liquidation.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond, 'borrower' and 'lender' are addresses

// Example: Create a new Trade Deal
bytes32 invoiceId = keccak256("my_invoice_123");
address collateralTok = 0xSomeERC20TokenAddress; // Address of a supported collateral token
uint256 collateralAmt = 1000 * (10 ** 18); // 1000 tokens
uint256 loanAmt = 500 * (10 ** 18);
uint256 repaymentAmt = 520 * (10 ** 18);
uint256 settlement = block.timestamp + 7 days;
uint256 grace = 1 days;

// Borrower approves transfer of collateral to the facet first
// IERC20(collateralTok).approve(address(gemforceDiamond), collateralAmt);

// Then create the trade deal (typically called by borrower or lender)
bytes32 newTradeDealId = IDiamond(diamond).createTradeDeal(
    lender,
    collateralTok,
    collateralAmt,
    loanAmt,
    repaymentAmt,
    settlement,
    grace,
    invoiceId
);

// Example: Repay the Trade Deal (by borrower)
// Borrower approves repayment token transfer to the facet first
// IERC20(repaymentToken).approve(address(gemforceDiamond), repaymentAmt);
IDiamond(diamond).repayTradeDeal(newTradeDealId);

// Example: Liquidate the Trade Deal (by lender, after default)
// Note: This call would only succeed if the settlement + grace period has passed
// IDiamond(diamond).liquidateTradeDeal(newTradeDealId);