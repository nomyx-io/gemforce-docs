# CollateralTokenFactoryFacet

The `CollateralTokenFactoryFacet` is a specialized component within the Gemforce Diamond smart contract system, responsible for the creation and management of new collateral token contracts. This facet enables the dynamic deployment of custom ERC-20 tokens designed specifically for use as collateral within the platform's Trade Deal module or other financial instruments.

## Purpose

The primary purpose of the `CollateralTokenFactoryFacet` is to streamline the process of issuing compliant and standardized collateral tokens. By providing a secure and configurable factory, it ensures that tokens created adhere to necessary parameters (e.g., supply caps, decimal places, name, symbol) and are automatically recognized and whitelisted for use within the Gemforce ecosystem. This facilitates rapid asset onboarding and expands the range of collateral options available for trade deals.

## Key Features

*   **Custom ERC-20 Deployment**: Allows authorized entities to deploy new ERC-20 token contracts with specified parameters.
*   **Whitelisting Integration**: Automatically registers newly created tokens as supported collateral within the `TradeDealAdminFacet`.
*   **Controlled Creation**: Only authorized accounts (Diamond owner) can initiate the creation of new collateral tokens.
*   **Parameter Configuration**: Provides options to define the initial supply, name, symbol, and decimals of the newly minted tokens.
*   **Ownership Assignment**: Assigns ownership of the newly created token to a specified address.
*   **Event Emission**: Emits events for every token creation, providing transparency and traceability of new collateral assets.

## Functions

### `createCollateralToken(string calldata _name, string calldata _symbol, uint8 _decimals, uint256 _initialSupply, address _owner)`

Creates and deploys a new ERC-20 token contract configured as a collateral token.

*   **Parameters**:
    *   `_name` (string calldata): The name of the new ERC-20 token (e.g., "Gemforce Collateral USD").
    *   `_symbol` (string calldata): The symbol of the new ERC-20 token (e.g., "GUSD").
    *   `_decimals` (uint8): The number of decimal places for the new token (e.g., 18).
    *   `_initialSupply` (uint256): The initial total supply of the new token.
    *   `_owner` (address): The address that will initially own all the `_initialSupply` of tokens.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_name` and `_symbol` must not be empty.
    *   `_initialSupply` must be greater than 0.
    *   `_owner` must not be the zero address.
*   **Emits**: `CollateralTokenCreated(address indexed tokenAddress, string name, string symbol, uint8 decimals, uint256 initialSupply, address owner)`

## Events

### `CollateralTokenCreated(address indexed tokenAddress, string name, string symbol, uint8 decimals, uint256 initialSupply, address owner)`

Emitted when a new collateral token contract is successfully created and deployed.

*   **Parameters**:
    *   `tokenAddress` (address): The address of the newly deployed ERC-20 token contract.
    *   `name` (string): The name of the token.
    *   `symbol` (string): The symbol of the token.
    *   `decimals` (uint8): The number of decimals of the token.
    *   `initialSupply` (uint256): The initial total supply of the token.
    *   `owner` (address): The address assigned as the owner of the initial supply.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'deployer' is the diamond owner

// Example: Create a new stablecoin-like collateral token
string memory tokenName = "Gemforce Stable Dollar";
string memory tokenSymbol = "GSD";
uint8 tokenDecimals = 18;
uint256 initialTokenSupply = 1_000_000 * (10 ** 18); // 1,000,000 GSD
address initialOwner = 0xYourPlatformWalletAddress;

IDiamond(diamond).createCollateralToken(
    tokenName,
    tokenSymbol,
    tokenDecimals,
    initialTokenSupply,
    initialOwner
);

// After creation, the new token address can be retrieved from the emitted event.
// This new token will automatically be added to the supported collateral list
// by the TradeDealAdminFacet due to internal integration logic.