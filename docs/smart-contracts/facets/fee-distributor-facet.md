# FeeDistributorFacet

The `FeeDistributorFacet` is a core component of the Gemforce Diamond smart contract system, responsible for managing the distribution of fees collected from various operations within the platform. It provides functionalities for setting fee recipients, defining distribution weights, and initiating the fee distribution process.

## Purpose

The primary purpose of the `FeeDistributorFacet` is to enable a flexible and transparent mechanism for revenue sharing and incentive distribution. It allows the platform to collect fees (e.g., from marketplace transactions, NFT mints) and
distribute them to designated beneficiaries (e.g., development teams, liquidity providers, ecosystem contributors) based on predefined percentages.

## Key Features

*   **Configurable Fee Recipients**: Allows setting multiple addresses as fee recipients.
*   **Weighted Distribution**: Enables assigning a specific percentage (weight) to each recipient, ensuring a proportional distribution of collected fees.
*   **Ownership Control**: Only authorized administrators (Diamond owner) can modify fee distribution settings.
*   **Event Emission**: Emits events for every significant action, such as setting fee distributors or initiating fee distribution, providing transparency and traceability.

## Functions

### `setFeeDistributors(address[] calldata _recipients, uint256[] calldata _weights)`

This function allows the owner to set or update the list of fee recipients and their corresponding distribution weights.

*   **Parameters**:
    *   `_recipients` (address[] calldata): An array of addresses that will receive the fees.
    *   `_weights` (uint256[] calldata): An array of `uint256` values representing the distribution weight (percentage) for each recipient. The sum of all weights must equal 10,000 (representing 100%).
*   **Requirements**:
    *   Only the `Diamond` owner can call this function.
    *   The length of `_recipients` array must be equal to the length of `_weights` array.
    *   All weights must be non-zero.
    *   The sum of `_weights` must be exactly 10,000.
*   **Emits**: `FeeDistributorsSet(address[] recipients, uint256[] weights)`

### `distributeFees(address _tokenAddress, uint256 _amount)`

Initiates the distribution of a specified amount of a given token to the configured fee recipients based on their weights.

*   **Parameters**:
    *   `_tokenAddress` (address): The address of the ERC-20 token to be distributed.
    *   `_amount` (uint256): The total amount of the token to distribute.
*   **Requirements**:
    *   The `FeeDistributorFacet` must have sufficient balance of `_tokenAddress` to fulfill the distribution.
    *   `_amount` must be greater than 0.
    *   There must be at least one fee distributor configured.
*   **Emits**: `FeesDistributed(address tokenAddress, uint256 amount)`

## Events

### `FeeDistributorsSet(address[] recipients, uint256[] weights)`

Emitted when the fee recipients and their weights are set or updated.

*   **Parameters**:
    *   `recipients` (address[]): The new array of fee recipient addresses.
    *   `weights` (uint256[]): The new array of distribution weights for the recipients.

### `FeesDistributed(address tokenAddress, uint256 amount)`

Emitted when fees are successfully distributed to the recipients.

*   **Parameters**:
    *   `tokenAddress` (address): The address of the token that was distributed.
    *   `amount` (uint256): The total amount of the token that was distributed.

## Usage Example

```solidity
// Example of setting fee distributors
address[] memory recipients = new address[](2);
recipients[0] = 0xRecipient1Address;
recipients[1] = 0xRecipient2Address;

uint256[] memory weights = new uint256[](2);
weights[0] = 7000; // 70%
weights[1] = 3000; // 30%

// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner
IDiamond(diamond).setFeeDistributors(recipients, weights);

// Example of distributing fees
IERC20(tokenAddress).transfer(address(this), amountToDistribute); // Transfer tokens to the facet first
IDiamond(diamond).distributeFees(tokenAddress, amountToDistribute);