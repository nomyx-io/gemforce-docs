# ERC721ALib Library

## Overview

[`ERC721ALib`](/Users/sschepis/Development/gem-base/contracts/libraries/ERC721ALib.sol) is a gas-optimized implementation of the ERC721 Non-Fungible Token standard, based on the ERC721A specification by Chiru Labs. This library provides efficient batch minting capabilities and optimized storage patterns for NFT collections, making it ideal for large-scale NFT projects.

## Key Features

- **Gas-Optimized Minting**: Batch mint multiple tokens with minimal gas overhead
- **Efficient Storage**: Optimized storage layout reduces gas costs for transfers and queries
- **ERC721 Compatibility**: Full compatibility with the ERC721 standard
- **Burn Functionality**: Support for token burning with proper accounting
- **Auxiliary Data**: Additional per-address data storage (e.g., whitelist usage)
- **Safe Transfer Support**: Built-in safe transfer checks for contract recipients

## Storage Structure

### ERC721AStorage

The main storage structure for ERC721A functionality:

```solidity
struct ERC721AStorage {
    ERC721EnumerableContract enumerations;
    ERC721AContract erc721Contract;
}
```

### ERC721AContract

Core contract storage containing all NFT data:
- Token ownership mappings
- Approval mappings
- Address data (balance, minted count, burned count, auxiliary data)
- Current index and burn counter

## Core Functions

### Supply and Balance Queries

#### `totalSupply()`
```solidity
function totalSupply(ERC721AContract storage self) internal view returns (uint256)
```
Returns the total number of tokens in circulation (minted - burned).

#### `balanceOf()`
```solidity
function balanceOf(ERC721AContract storage self, address owner) internal view returns (uint256)
```
Returns the number of tokens owned by a specific address.

### Minting Statistics

#### `_numberMinted()`
```solidity
function _numberMinted(ERC721AContract storage self, address owner) internal view returns (uint256)
```
Returns the total number of tokens minted by an address.

#### `_numberBurned()`
```solidity
function _numberBurned(ERC721AContract storage self, address owner) internal view returns (uint256)
```
Returns the total number of tokens burned by or on behalf of an address.

### Auxiliary Data Management

#### `_getAux()`
```solidity
function _getAux(ERC721AContract storage self, address owner) internal view returns (uint64)
```
Returns auxiliary data for an address (e.g., whitelist mint slots used).

#### `_setAux()`
```solidity
function _setAux(ERC721AContract storage self, address owner, uint64 aux) internal
```
Sets auxiliary data for an address.

### Token Ownership

#### `ownershipOf()`
```solidity
function ownershipOf(ERC721AContract storage self, uint256 tokenId) internal view returns (TokenOwnership memory)
```
Returns detailed ownership information for a token, including:
- Owner address
- Start timestamp
- Burned status

#### `_ownerOf()`
```solidity
function _ownerOf(ERC721AContract storage self, uint256 tokenId) internal view returns (address)
```
Returns the owner address of a specific token.

### Token Existence and Approval

#### `_exists()`
```solidity
function _exists(ERC721AContract storage self, uint256 tokenId) internal view returns (bool)
```
Checks if a token exists (has been minted and not burned).

#### `getApproved()`
```solidity
function getApproved(ERC721AContract storage self, uint256 tokenId) internal view returns (address)
```
Returns the approved address for a specific token.

#### `setApprovalForAll()`
```solidity
function setApprovalForAll(ERC721AContract storage self, address sender, address operator, bool approved) internal
```
Sets or unsets approval for all tokens owned by sender.

#### `isApprovedForAll()`
```solidity
function isApprovedForAll(ERC721AContract storage self, address owner, address operator) internal view returns (bool)
```
Checks if an operator is approved for all tokens of an owner.

### Minting Operations

#### `_mint()`
```solidity
function _mint(
    ERC721AContract storage self,
    address msgSender,
    address to,
    uint256 quantity,
    bytes memory _data,
    bool safe
) internal
```

Mints `quantity` tokens to the `to` address with gas optimization for batch operations.

**Parameters:**
- `msgSender`: The address initiating the mint
- `to`: Recipient address
- `quantity`: Number of tokens to mint
- `_data`: Additional data for safe transfer check
- `safe`: Whether to perform safe transfer checks

**Gas Optimization:**
- Only stores ownership data for the first token in a batch
- Subsequent tokens inherit ownership through lookup algorithm
- Significantly reduces gas costs for batch minting

### Transfer Operations

#### `_transfer()`
```solidity
function _transfer(
    ERC721AContract storage self,
    address msgSender,
    address from,
    address to,
    uint256 tokenId,
    bool _force
) internal
```

Transfers a token from one address to another with proper ownership updates.

**Parameters:**
- `_force`: Bypasses approval checks when true (for internal operations)

**Features:**
- Automatic approval clearing
- Ownership slot optimization for gas efficiency
- Proper balance updates

### Burning Operations

#### `_burn()`
```solidity
function _burn(ERC721AContract storage self, uint256 tokenId) internal
```

Burns a token, marking it as destroyed while maintaining ownership history.

**Features:**
- Maintains burn counter for supply calculations
- Preserves ownership data with burned flag
- Updates address statistics

## Gas Optimization Features

### Batch Minting Efficiency
- **Single Storage Write**: Only the first token in a batch requires ownership storage
- **Lookup Algorithm**: Subsequent tokens find ownership through backward iteration
- **Reduced Gas Costs**: Significant savings for large batch mints

### Storage Layout Optimization
- **Packed Structs**: Address data packed into single storage slots
- **Minimal Writes**: Only necessary storage updates during operations
- **Efficient Mappings**: Optimized mapping structures for common operations

### Transfer Optimization
- **Ownership Slot Management**: Efficient handling of ownership slots during transfers
- **Balance Updates**: Minimal storage writes for balance changes
- **Approval Clearing**: Gas-efficient approval management

## Error Handling

The library defines comprehensive custom errors for gas-efficient reverts:

```solidity
error ApprovalCallerNotOwnerNorApproved();
error ApprovalQueryForNonexistentToken();
error MintToZeroAddress();
error MintZeroQuantity();
error OwnerQueryForNonexistentToken();
error TransferCallerNotOwnerNorApproved();
error TransferFromIncorrectOwner();
error TransferToNonERC721ReceiverImplementer();
```

## Hook Functions

### `_beforeTokenTransfers()`
```solidity
function _beforeTokenTransfers(
    ERC721AContract storage self,
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity,
    bool force
) internal
```

Hook called before token transfers, minting, or burning. Can be overridden for custom logic.

### `_afterTokenTransfers()`
```solidity
function _afterTokenTransfers(
    ERC721AContract storage self,
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal
```

Hook called after token transfers, minting, or burning. Can be overridden for custom logic.

## Usage Examples

### Basic Minting
```solidity
// Mint 5 tokens to an address
ERC721ALib._mint(
    erc721Storage,
    msg.sender,
    recipient,
    5,
    "",
    true
);
```

### Batch Operations
```solidity
// Check total supply
uint256 supply = ERC721ALib.totalSupply(erc721Storage);

// Check user's minted count
uint256 minted = ERC721ALib._numberMinted(erc721Storage, user);

// Set auxiliary data (e.g., whitelist usage)
ERC721ALib._setAux(erc721Storage, user, 3);
```

### Transfer Operations
```solidity
// Transfer token
ERC721ALib._transfer(
    erc721Storage,
    msg.sender,
    from,
    to,
    tokenId,
    false
);

// Burn token
ERC721ALib._burn(erc721Storage, tokenId);
```

## Integration with Diamond Pattern

The library is designed to work within the Diamond pattern:

```solidity
import { ERC721ALib } from "./libraries/ERC721ALib.sol";

contract NFTFacet {
    function mint(address to, uint256 quantity) external {
        ERC721AStorage storage s = erc721AStorage();
        ERC721ALib._mint(s.erc721Contract, msg.sender, to, quantity, "", true);
    }
}
```

## Security Considerations

### Access Control
- **Minting Permissions**: Implement proper access controls for minting functions
- **Transfer Restrictions**: Consider implementing transfer restrictions if needed
- **Burn Authorization**: Ensure only authorized parties can burn tokens

### Validation
- **Address Validation**: All functions validate against zero addresses
- **Quantity Limits**: Implement reasonable limits for batch operations
- **Overflow Protection**: Built-in protection against arithmetic overflows

### Safe Transfers
- **Contract Recipients**: Automatic checks for contract recipients
- **Callback Validation**: Proper handling of ERC721Receiver callbacks
- **Reentrancy Protection**: Consider reentrancy guards for external calls

## Performance Characteristics

### Gas Costs (Approximate)
- **Single Mint**: ~50,000 gas
- **Batch Mint (10 tokens)**: ~55,000 gas (5,500 per additional token)
- **Transfer**: ~30,000 gas
- **Burn**: ~25,000 gas

### Storage Efficiency
- **Ownership Storage**: Only first token in batch requires storage
- **Address Data**: Packed into single storage slots
- **Minimal Overhead**: Efficient data structures minimize storage costs

## Related Documentation

- [ERC721A Enumeration Library](erc721a-enumeration-lib.md) - Enumeration extension
- [Attribute Library](attribute-lib.md) - Token attribute management
- [Diamond Library](diamond-lib.md) - Diamond pattern integration
- [Gemforce Minter Facet](../facets/gemforce-minter-facet.md) - Minting interface
- [ERC721A Standard](https://eips.ethereum.org/EIPS/eip-721) - ERC721 specification

## Best Practices

1. **Batch Minting**: Use batch minting for gas efficiency when minting multiple tokens
2. **Auxiliary Data**: Leverage auxiliary data for additional per-address information
3. **Hook Implementation**: Use hooks for custom logic without modifying core functions
4. **Error Handling**: Use custom errors for gas-efficient error reporting
5. **Access Control**: Implement proper permissions for all minting and administrative functions