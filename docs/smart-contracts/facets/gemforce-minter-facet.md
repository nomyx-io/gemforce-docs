# GemforceMinterFacet

The `GemforceMinterFacet` is a crucial component within the Gemforce Diamond smart contract system, responsible for the creation and emission of new ERC721A NFTs (Non-Fungible Tokens) based on a flexible and configurable minting process. This facet enables the platform to issue unique digital assets with various attributes and pricing models.

## Purpose

The primary purpose of the `GemforceMinterFacet` is to provide a robust and extensible framework for minting NFTs. It supports different pricing strategies, ensures proper attribute assignment, and integrates with the overall Gemforce ecosystem to allow for diverse token issuance scenarios, such as carbon credits, digital collectibles, or in-platform items.

## Key Features

*   **Configurable Minting Batches**: Allows defining parameters for different NFT collections or batches, including start/end times, total supply, price per token, and recipient limits.
*   **Flexible Pricing**: Supports various token pricing models or integrates with external pricing mechanisms.
*   **Presale/Public Sale Management**: Enables setting up distinct phases for NFT sales with different rules (e.g., whitelist-only presales).
*   **Royalty Support**: Integrates with royalty standards (e.g., ERC2981, EIP-2981) to ensure creators receive a percentage of secondary sales.
*   **Attribute Assignment**: Facilitates the assignment of unique attributes to minted NFTs, which can be stored on-chain or referenced via metadata URIs.
*   **Event Emission**: Emits detailed events for every minting operation, providing transparency and traceability.

## Functions

### `configureMintingBatch(uint256 _batchId, uint256 _pricePerToken, uint256 _maxSupply, uint256 _maxMintPerWallet, uint256 _mintStart, uint256 _mintEnd, bytes32 _merkleRoot)`

Configures the parameters for a specific minting batch. Requires the owner's permission.

*   **Parameters**:
    *   `_batchId` (uint256): A unique identifier for the minting batch.
    *   `_pricePerToken` (uint256): The amount of tokens required for each NFT within this batch.
    *   `_maxSupply` (uint256): The maximum total number of NFTs that can be minted in this batch.
    *   `_maxMintPerWallet` (uint256): The maximum number of NFTs a single wallet can mint from this batch.
    *   `_mintStart` (uint256): The timestamp when minting for this batch begins.
    *   `_mintEnd` (uint256): The timestamp when minting for this batch ends.
    *   `_merkleRoot` (bytes32): The Merkle root for a whitelist, if a presale is enabled for this batch (use `bytes32(0)` if not applicable).
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_batchId` must not be 0.
    *   `_maxSupply` must be greater than 0.
    *   `_pricePerToken` can be 0 for free mints.
*   **Emits**: `MintingBatchConfigured(uint256 batchId, uint256 pricePerToken, uint256 maxSupply, uint256 maxMintPerWallet, uint256 mintStart, uint256 mintEnd, bytes32 merkleRoot)`

### `mint(uint256 _batchId, uint256 _count, bytes32[] calldata _merkleProof)`

Allows a user to mint NFTs from a configured batch.

*   **Parameters**:
    *   `_batchId` (uint256): The ID of the minting batch to mint from.
    *   `_count` (uint256): The number of NFTs to mint in this transaction.
    *   `_merkleProof` (bytes32[] calldata): A Merkle proof if the batch requires whitelisting.
*   **Requirements**:
    *   The minting batch must be configured and active (within `mintStart` and `mintEnd`).
    *   The caller must send the correct amount of tokens (message value) based on `_pricePerToken` and `_count`.
    *   `_count` must be greater than 0.
    *   The total minted supply for the batch, plus `_count`, must not exceed `_maxSupply`.
    *   The caller's total minted count for the batch, plus `_count`, must not exceed `_maxMintPerWallet`.
    *   If a Merkle root is set, a valid Merkle proof must be provided for the caller's address.
*   **Emits**: `Minted(address indexed minter, uint256 indexed batchId, uint256 count)`

### `withdrawFunds()`

Allows the owner to withdraw collected funds from the facet.

*   **Requirements**:
    *   Only the Diamond owner can call this function.
*   **Emits**: `FundsWithdrawn(address indexed to, uint256 amount)`

## Events

### `MintingBatchConfigured(uint256 batchId, uint256 pricePerToken, uint256 maxSupply, uint256 maxMintPerWallet, uint256 mintStart, uint256 mintEnd, bytes32 merkleRoot)`

Emitted when a new minting batch is configured or an existing one is updated.

*   **Parameters**: Details the configuration of the minting batch.

### `Minted(address indexed minter, uint256 indexed batchId, uint256 count)`

Emitted after a successful NFT minting operation.

*   **Parameters**:
    *   `minter` (address): The address of the account that minted the NFTs.
    *   `batchId` (uint256): The ID of the batch from which NFTs were minted.
    *   `count` (uint256): The number of NFTs minted in this transaction.

### `FundsWithdrawn(address indexed to, uint256 amount)`

Emitted when funds are withdrawn from the facet by the owner.

*   **Parameters**:
    *   `to` (address): The address to which the funds were sent.
    *   `amount` (uint256): The amount of funds withdrawn.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Configure a new minting batch (e.g., Batch 1)
uint256 batchId = 1;
uint256 pricePerToken = 1 ether; // 1 ETH per token
uint256 maxSupply = 100;
uint256 maxMintPerWallet = 5;
uint256 mintStart = block.timestamp;
uint256 mintEnd = block.timestamp + 1 weeks;
bytes32 merkleRoot = bytes32(0); // No whitelist for this example

IDiamond(diamond).configureMintingBatch(
    batchId,
    pricePerToken,
    maxSupply,
    maxMintPerWallet,
    mintStart,
    mintEnd,
    merkleRoot
);

// A user mints 2 NFTs from Batch 1
uint256 numToMint = 2;
IDiamond(diamond).mint{value: pricePerToken * numToMint}(batchId, numToMint, new bytes32[](0));

// Owner withdraws funds
IDiamond(diamond).withdrawFunds();