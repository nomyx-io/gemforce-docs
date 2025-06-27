# MultiSaleFacet

The `MultiSaleFacet` is an integral part of the Gemforce Diamond smart contract system, designed to enable the sale of multiple types of tokens (ERC-20, ERC-721, ERC-1155) in a single, flexible contract. It supports various sale configurations, allowing for diverse token distribution strategies.

## Purpose

The primary purpose of the `MultiSaleFacet` is to provide a versatile and robust platform for conducting token sales on the Gemforce ecosystem. It abstracts the complexities of managing different token standards and payment methods, allowing project creators to easily set up and manage sales for their digital assets, whether they are fungible tokens, unique NFTs, or semi-fungible items.

## Key Features

*   **Multi-Token Support**: Handles sales for ERC-20, ERC-721, and ERC-1155 tokens.
*   **Configurable Sale Parameters**: Allows setting up sale periods, token prices, minimum/maximum purchase limits, and payment token types.
*   **Whitelisting/Presale Functionality**: Supports restricted access sales via Merkle proofs for whitelisted addresses.
*   **Flexible Payment Options**: Accepts various ERC-20 tokens as payment, in addition to native blockchain currency (e.g., Ether).
*   **Royalty Management**: Can integrate with royalty standards to ensure creators receive their share from secondary sales.
*   **Refund Capabilities**: Provides mechanisms for users to request refunds under specified conditions (e.g., if a sale is canceled).
*   **Event Emission**: Emits detailed events for every sale operation, providing transparency and traceability.

## Functions

### `configureSale(uint256 _saleId, address _tokenForSale, uint256 _tokenStandard, uint256 _unitPrice, uint256 _maxSupply, uint256 _maxPurchasePerAddress, address _paymentToken, uint256 _startTime, uint256 _endTime, bytes32 _merkleRoot)`

Configures the parameters for a specific token sale. Callable only by the owner.

*   **Parameters**:
    *   `_saleId` (uint256): A unique identifier for the sale.
    *   `_tokenForSale` (address): The address of the token being sold (ERC-20, ERC-721, or ERC-1155).
    *   `_tokenStandard` (uint256): Indicates the token standard (e.g., 0 for ERC-20, 1 for ERC-721, 2 for ERC-1155).
    *   `_unitPrice` (uint256): The price of one unit of the token (or one NFT).
    *   `_maxSupply` (uint256): The maximum number of tokens/NFTs available for sale in this batch.
    *   `_maxPurchasePerAddress` (uint256): The maximum number of units an individual address can purchase.
    *   `_paymentToken` (address): The address of the ERC-20 token accepted as payment. Use `address(0)` for native currency.
    *   `_startTime` (uint256): Unix timestamp when the sale begins.
    *   `_endTime` (uint256): Unix timestamp when the sale ends.
    *   `_merkleRoot` (bytes32): The Merkle root for whitelisting, if applicable (use `bytes32(0)` for public sale).
*   **Requirements**:
    *   Only the Diamond owner can call.
    *   `_saleId` must be unique.
    *   `_tokenStandard` must be a valid and supported value.
*   **Emits**: `SaleConfigured(uint256 saleId, ...)`

### `buy(uint256 _saleId, uint256 _amount, bytes32[] calldata _merkleProof, uint256 _tokenId)`

Allows a user to purchase tokens from a configured sale.

*   **Parameters**:
    *   `_saleId` (uint256): The ID of the sale to purchase from.
    *   `_amount` (uint256): The number of tokens/NFTs to purchase.
    *   `_merkleProof` (bytes32[] calldata): Merkle proof if whitelisting is active.
    *   `_tokenId` (uint256): (For ERC-1155 sales) The specific ID of the ERC-1155 token to buy. Ignored for ERC-20/ERC-721.
*   **Requirements**:
    *   Sale must be active (`startTime` <= `block.timestamp` < `endTime`).
    *   Sufficient tokens available in the contract.
    *   Caller's purchase amount must not exceed `maxPurchasePerAddress`.
    *   Correct payment amount must be sent (either via `msg.value` or ERC-20 `transferFrom`).
    *   If Merkle root is set, a valid proof for `msg.sender` must be provided.
*   **Emits**: `TokensPurchased(uint256 indexed saleId, address indexed buyer, uint256 amount, address tokenForSale, address paymentToken)`

### `withdrawFunds(address _tokenAddress, address _to, uint256 _amount)`

Allows the owner to withdraw collected funds from the facet.

*   **Parameters**:
    *   `_tokenAddress` (address): The address of the token to withdraw.
    *   `_to` (address): The address to send the funds to.
    *   `_amount` (uint256): The amount to withdraw.
*   **Requirements**:
    *   Only the Diamond owner can call.
*   **Emits**: `FundsWithdrawn(address indexed tokenAddress, address indexed to, uint256 amount)`

## Events

### `SaleConfigured(uint256 indexed saleId, address tokenForSale, uint256 tokenStandard, uint256 unitPrice, uint256 maxSupply, uint256 maxPurchasePerAddress, address paymentToken, uint256 startTime, uint256 endTime, bytes32 merkleRoot)`

Emitted when a new sale is configured or an existing one is updated.

*   **Parameters**: Details the full configuration of the sale.

### `TokensPurchased(uint256 indexed saleId, address indexed buyer, uint256 amount, address tokenForSale, address paymentToken)`

Emitted after a successful token purchase.

*   **Parameters**:
    *   `saleId` (uint256): The ID of the sale from which tokens were purchased.
    *   `buyer` (address): The address of the buyer.
    *   `amount` (uint256): The number of tokens purchased.
    *   `tokenForSale` (address): The address of the token that was bought.
    *   `paymentToken` (address): The address of the token used for payment.

### `FundsWithdrawn(address indexed tokenAddress, address indexed to, uint256 amount)`

Emitted when funds are withdrawn from the facet.

*   **Parameters**:
    *   `tokenAddress` (address): The address of the token that was withdrawn.
    *   `to` (address): The recipient of the withdrawn funds.
    *   `amount` (uint256): The amount withdrawn.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Configure an ERC-721 sale (Sale ID 1)
uint256 saleId = 1;
address nftAddress = 0xYourERC721NFTContract;
uint256 tokenStandard = 1; // ERC-721
uint256 unitPrice = 0.05 ether; // 0.05 ETH per NFT
uint256 maxSupply = 100;
uint256 maxPurchasePerAddress = 2;
address paymentToken = address(0); // Native currency (ETH)
uint256 startTime = block.timestamp;
uint256 endTime = block.timestamp + 1 weeks;
bytes32 merkleRoot = bytes32(0); // No whitelist

IDiamond(diamond).configureSale(
    saleId,
    nftAddress,
    tokenStandard,
    unitPrice,
    maxSupply,
    maxPurchasePerAddress,
    paymentToken,
    startTime,
    endTime,
    merkleRoot
);

// A user buys 1 NFT from Sale ID 1
// First, approve the MultiSaleFacet to spend the NFTs if they are not already held by the facet
// (This example assumes the NFTs are already held by the facet or minted directly into it)
// For ERC-721, the facet needs to be approved or be the owner of the tokens before transfers for sale.
IDiamond(diamond).buy{value: unitPrice}(saleId, 1, new bytes32[](0), 0);

// Owner withdraws funds
IDiamond(diamond).withdrawFunds(address(0), ownerAddress, IDiamond(diamond).getEthBalance());