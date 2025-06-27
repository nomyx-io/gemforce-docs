# ERC721AEnumerationLib Library

## Overview

[`ERC721AEnumerationLib`](/Users/sschepis/Development/gem-base/contracts/libraries/ERC721AEnumerationLib.sol) provides enumeration functionality for ERC721A tokens, implementing the ERC721Enumerable extension. This library enables efficient querying and iteration over token collections, allowing applications to discover tokens by index and enumerate tokens owned by specific addresses.

## Key Features

- **Token Enumeration**: Query tokens by global index
- **Owner Token Listing**: Enumerate tokens owned by specific addresses
- **Gas-Optimized Operations**: Efficient O(1) add/remove operations with swap-and-pop pattern
- **ERC721Enumerable Compliance**: Full compatibility with the ERC721Enumerable standard
- **Index Tracking**: Maintains bidirectional mappings between tokens and indices

## Core Functions

### Public Query Functions

#### `tokenOfOwnerByIndex()`
```solidity
function tokenOfOwnerByIndex(
    ERC721EnumerableContract storage self, 
    address owner, 
    uint256 index
) internal view returns (uint256)
```

Returns the token ID at the specified index in the owner's token list.

**Parameters:**
- `owner`: Address of the token owner
- `index`: Index in the owner's token array (0-based)

**Returns:**
- Token ID at the specified index

**Usage:**
```solidity
// Get the first token owned by an address
uint256 firstToken = ERC721AEnumerationLib.tokenOfOwnerByIndex(enumStorage, owner, 0);
```

#### `totalSupply()`
```solidity
function totalSupply(ERC721EnumerableContract storage self) internal view returns (uint256)
```

Returns the total number of tokens in the collection.

**Returns:**
- Total number of tokens currently in existence

#### `tokenByIndex()`
```solidity
function tokenByIndex(
    ERC721EnumerableContract storage self, 
    uint256 index
) internal view returns (uint256)
```

Returns the token ID at the specified global index.

**Parameters:**
- `index`: Global index in the all tokens array (0-based)

**Returns:**
- Token ID at the specified global index

**Usage:**
```solidity
// Get the 10th token in the collection
uint256 token = ERC721AEnumerationLib.tokenByIndex(enumStorage, 9);
```

### Internal Management Functions

#### `_addTokenToOwnerEnumeration()`
```solidity
function _addTokenToOwnerEnumeration(
    ERC721EnumerableContract storage self, 
    address to, 
    uint256 tokenId
) internal
```

Adds a token to the owner's enumeration tracking when minted or transferred.

**Parameters:**
- `to`: Address receiving the token
- `tokenId`: ID of the token being added

**Internal Logic:**
- Adds token to the end of owner's token array
- Records the token's position in the owner's array
- Updates bidirectional mapping for efficient lookups

#### `_addTokenToAllTokensEnumeration()`
```solidity
function _addTokenToAllTokensEnumeration(
    ERC721EnumerableContract storage self, 
    uint256 tokenId
) internal
```

Adds a token to the global enumeration tracking when minted.

**Parameters:**
- `tokenId`: ID of the token being added

**Internal Logic:**
- Adds token to the end of global tokens array
- Records the token's position in the global array
- Maintains global token index mapping

#### `_removeTokenFromOwnerEnumeration()`
```solidity
function _removeTokenFromOwnerEnumeration(
    ERC721EnumerableContract storage self, 
    address from, 
    uint256 tokenId
) internal
```

Removes a token from the owner's enumeration tracking when transferred or burned.

**Parameters:**
- `from`: Address losing the token
- `tokenId`: ID of the token being removed

**Gas Optimization Features:**
- **Swap-and-Pop Pattern**: Moves last token to deleted position to avoid gaps
- **O(1) Complexity**: Constant time removal regardless of collection size
- **Index Preservation**: Maintains array compactness without shifting elements

#### `_removeTokenFromAllTokensEnumeration()`
```solidity
function _removeTokenFromAllTokensEnumeration(
    ERC721EnumerableContract storage self, 
    uint256 tokenId
) internal
```

Removes a token from the global enumeration tracking when burned.

**Parameters:**
- `tokenId`: ID of the token being removed

**Gas Optimization Features:**
- **Swap-and-Pop Pattern**: Maintains array compactness
- **O(1) Complexity**: Efficient removal from global tracking
- **Index Updates**: Properly updates moved token's index

## Storage Structure

The enumeration library uses the following storage mappings:

```solidity
struct ERC721EnumerableContract {
    // Mapping from owner to list of owned token IDs
    mapping(address => mapping(uint256 => uint256)) _ownedTokens;
    
    // Mapping from token ID to index of the owner tokens list
    mapping(uint256 => uint256) _ownedTokensIndex;
    
    // Array with all token ids, used for enumeration
    uint256[] _allTokens;
    
    // Mapping from token id to position in the allTokens array
    mapping(uint256 => uint256) _allTokensIndex;
}
```

## Gas Optimization Techniques

### Swap-and-Pop Pattern

The library uses an efficient swap-and-pop pattern for removals:

1. **Identify Position**: Find the index of the token to remove
2. **Swap with Last**: Move the last token to the position being vacated
3. **Update Index**: Update the moved token's index mapping
4. **Pop Last**: Remove the last element (now duplicate)

This approach:
- Maintains array compactness without gaps
- Achieves O(1) time complexity for removals
- Minimizes gas costs by avoiding array shifts

### Bidirectional Mappings

The library maintains bidirectional mappings between tokens and indices:
- **Token → Index**: Quick lookup of token position
- **Index → Token**: Direct access to token by position
- **Efficient Updates**: Both mappings updated atomically

## Integration Patterns

### With ERC721A Library

```solidity
import { ERC721ALib } from "./ERC721ALib.sol";
import { ERC721AEnumerationLib } from "./ERC721AEnumerationLib.sol";

contract NFTFacet {
    function mint(address to, uint256 quantity) external {
        // Mint tokens using ERC721A
        ERC721ALib._mint(erc721Storage, msg.sender, to, quantity, "", true);
        
        // Update enumeration for each token
        uint256 startTokenId = ERC721ALib.currentIndex(erc721Storage) - quantity;
        for (uint256 i = 0; i < quantity; i++) {
            ERC721AEnumerationLib._addTokenToOwnerEnumeration(
                enumStorage, 
                to, 
                startTokenId + i
            );
            ERC721AEnumerationLib._addTokenToAllTokensEnumeration(
                enumStorage, 
                startTokenId + i
            );
        }
    }
}
```

### Hook Integration

```solidity
// In ERC721A hooks
function _beforeTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal override {
    super._beforeTokenTransfers(from, to, startTokenId, quantity);
    
    for (uint256 i = 0; i < quantity; i++) {
        uint256 tokenId = startTokenId + i;
        
        if (from != address(0)) {
            ERC721AEnumerationLib._removeTokenFromOwnerEnumeration(enumStorage, from, tokenId);
        }
        
        if (to != address(0)) {
            ERC721AEnumerationLib._addTokenToOwnerEnumeration(enumStorage, to, tokenId);
        } else {
            ERC721AEnumerationLib._removeTokenFromAllTokensEnumeration(enumStorage, tokenId);
        }
        
        if (from == address(0)) {
            ERC721AEnumerationLib._addTokenToAllTokensEnumeration(enumStorage, tokenId);
        }
    }
}
```

## Usage Examples

### Querying Owner Tokens

```solidity
function getOwnerTokens(address owner) external view returns (uint256[] memory) {
    uint256 balance = IERC721(address(this)).balanceOf(owner);
    uint256[] memory tokens = new uint256[](balance);
    
    for (uint256 i = 0; i < balance; i++) {
        tokens[i] = ERC721AEnumerationLib.tokenOfOwnerByIndex(enumStorage, owner, i);
    }
    
    return tokens;
}
```

### Paginated Token Listing

```solidity
function getTokensPaginated(
    uint256 offset, 
    uint256 limit
) external view returns (uint256[] memory) {
    uint256 total = ERC721AEnumerationLib.totalSupply(enumStorage);
    require(offset < total, "Offset out of bounds");
    
    uint256 end = offset + limit;
    if (end > total) {
        end = total;
    }
    
    uint256[] memory tokens = new uint256[](end - offset);
    for (uint256 i = offset; i < end; i++) {
        tokens[i - offset] = ERC721AEnumerationLib.tokenByIndex(enumStorage, i);
    }
    
    return tokens;
}
```

### Random Token Selection

```solidity
function getRandomToken(uint256 seed) external view returns (uint256) {
    uint256 total = ERC721AEnumerationLib.totalSupply(enumStorage);
    require(total > 0, "No tokens exist");
    
    uint256 randomIndex = seed % total;
    return ERC721AEnumerationLib.tokenByIndex(enumStorage, randomIndex);
}
```

## Performance Characteristics

### Time Complexity
- **Query Operations**: O(1) for all query functions
- **Add Operations**: O(1) for adding tokens to enumeration
- **Remove Operations**: O(1) for removing tokens from enumeration

### Gas Costs (Approximate)
- **Add to Owner Enumeration**: ~5,000 gas
- **Remove from Owner Enumeration**: ~8,000 gas
- **Add to Global Enumeration**: ~5,000 gas
- **Remove from Global Enumeration**: ~8,000 gas
- **Query Operations**: ~500-1,000 gas

### Storage Overhead
- **Per Token**: 2 storage slots (owner index + global index)
- **Per Owner**: Dynamic array of owned tokens
- **Global**: Single array of all tokens

## Security Considerations

### Index Bounds Checking
- All query functions validate index bounds
- Prevents out-of-bounds array access
- Returns appropriate error messages

### State Consistency
- Enumeration state must be kept in sync with token transfers
- Proper integration with transfer hooks is essential
- Bidirectional mappings must be updated atomically

### Gas Limits
- Large collections may hit gas limits for batch operations
- Consider pagination for operations over many tokens
- Monitor gas usage in enumeration updates

## Best Practices

1. **Hook Integration**: Always integrate enumeration updates with transfer hooks
2. **Batch Operations**: Consider gas costs when processing multiple tokens
3. **Index Validation**: Always validate indices before array access
4. **State Synchronization**: Ensure enumeration state stays consistent with token state
5. **Gas Monitoring**: Monitor gas usage for large collections

## Related Documentation

- [ERC721A Library](erc721a-lib.md) - Core ERC721A implementation
- [ERC721Enumerable Standard](https://eips.ethereum.org/EIPS/eip-721) - ERC721 enumeration extension
- [Diamond Library](diamond-lib.md) - Diamond pattern integration
- [Gemforce Minter Facet](../facets/gemforce-minter-facet.md) - Minting interface

## Migration Notes

When upgrading from standard ERC721 enumeration:
- Storage layout is compatible with OpenZeppelin's implementation
- Gas costs are significantly optimized
- API remains fully compatible
- Integration requires proper hook setup