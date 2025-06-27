# TradeDealAdminFacet

The `TradeDealAdminFacet` is a key administrative component within the Gemforce Diamond smart contract system, specifically designed to manage and configure the parameters related to Trade Deals. This facet provides authorized entities with the ability to set global parameters, manage supported collateral tokens, and define standard terms for trade agreements on the platform.

## Purpose

The primary purpose of the `TradeDealAdminFacet` is to provide centralized control and flexibility over the Trade Deal mechanism. It allows the platform administrators to adjust critical settings that impact all trade deals created or processed through the system, ensuring adaptability to changing market conditions, legal requirements, or business strategies. This separation of administrative concerns into a dedicated facet enhances modularity and security.

## Key Features

*   **Global Trade Deal Configuration**: Allows setting parameters like default settlement periods, grace periods, or dispute resolution mechanisms.
*   **Collateral Token Management**: Enables listing and delisting supported ERC-20 tokens that can be used as collateral in trade deals.
*   **Fee Configuration**: Defines fees associated with trade deal creation or settlement.
*   **Administrator Control**: All sensitive functions are restricted to authorized administrators (Diamond owner or designated roles).
*   **Event Emission**: Emits events for every administrative action, providing transparency and auditability of changes to trade deal settings.

## Functions

### `setTradeDealParameters(uint256 _defaultSettlementPeriod, uint256 _gracePeriod, address _disputeResolutionService)`

Sets global parameters for trade deals.

*   **Parameters**:
    *   `_defaultSettlementPeriod` (uint256): The default time (in seconds) for a trade deal to be settled after creation.
    *   `_gracePeriod` (uint256): The additional time (in seconds) allowed beyond the settlement period before a trade deal can be liquidated.
    *   `_disputeResolutionService` (address): The address of a contract or entity responsible for resolving disputes. Set to `address(0)` if no on-chain service.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
*   **Emits**: `TradeDealParametersSet(uint256 defaultSettlementPeriod, uint256 gracePeriod, address disputeResolutionService)`

### `addSupportedCollateralToken(address _tokenAddress)`

Adds an ERC-20 token to the list of approved collateral tokens for trade deals.

*   **Parameters**:
    *   `_tokenAddress` (address): The address of the ERC-20 token to add.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_tokenAddress` must not already be supported.
    *   `_tokenAddress` must be a valid ERC-20 token.
*   **Emits**: `SupportedCollateralTokenAdded(address indexed tokenAddress)`

### `removeSupportedCollateralToken(address _tokenAddress)`

Removes an ERC-20 token from the list of approved collateral tokens.

*   **Parameters**:
    *   `_tokenAddress` (address): The address of the ERC-20 token to remove.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_tokenAddress` must be currently supported.
*   **Emits**: `SupportedCollateralTokenRemoved(address indexed tokenAddress)`

### `isSupportedCollateralToken(address _tokenAddress)`

Checks if a given token address is an approved collateral token.

*   **Parameters**:
    *   `_tokenAddress` (address): The address of the token to check.
*   **Returns**: (bool) `true` if the token is supported, `false` otherwise.

## Events

### `TradeDealParametersSet(uint256 defaultSettlementPeriod, uint256 gracePeriod, address disputeResolutionService)`

Emitted when the global trade deal parameters are set or updated.

*   **Parameters**:
    *   `defaultSettlementPeriod` (uint256): The new default settlement period.
    *   `gracePeriod` (uint256): The new grace period.
    *   `disputeResolutionService` (address): The new dispute resolution service address.

### `SupportedCollateralTokenAdded(address indexed tokenAddress)`

Emitted when a new token is added to the list of supported collateral tokens.

*   **Parameters**:
    *   `tokenAddress` (address): The address of the token that was added.

### `SupportedCollateralTokenRemoved(address indexed tokenAddress)`

Emitted when a token is removed from the list of supported collateral tokens.

*   **Parameters**:
    *   `tokenAddress` (address): The address of the token that was removed.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Set global trade deal parameters
uint256 defaultSettlement = 7 days;
uint256 gracePeriod = 2 days;
address disputeService = 0xSomeDisputeResolutionContract; // Or address(0)

IDiamond(diamond).setTradeDealParameters(defaultSettlement, gracePeriod, disputeService);

// Add a new supported collateral token (e.g., DAI)
address daiAddress = 0x6B175474E89094C44Da98b954EedeAC495271d0F; // Example DAI address
IDiamond(diamond).addSupportedCollateralToken(daiAddress);

// Check if a token is supported
bool isDaiSupported = IDiamond(diamond).isSupportedCollateralToken(daiAddress);

// Remove the supported collateral token
IDiamond(diamond).removeSupportedCollateralToken(daiAddress);