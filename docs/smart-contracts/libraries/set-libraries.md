# Set Libraries (AddressSet, Bytes32Set, UInt256Set)

## Overview

The Gemforce platform includes three specialized set libraries that provide efficient, gas-optimized data structures for managing unique collections of different data types. These libraries implement unordered sets with O(1) operations for insertion, deletion, and existence checks, making them ideal for managing collections where uniqueness and fast lookups are required.

## Libraries Included

- [`AddressSet`](/Users/sschepis/Development/gem-base/contracts/libraries/AddressSet.sol) - Set of Ethereum addresses
- [`Bytes32Set`](/Users/sschepis/Development/gem-base/contracts/libraries/Bytes32Set.sol) - Set of bytes32 values
- [`UInt256Set`](/Users/sschepis/Development/gem-base/contracts/libraries/UInt256Set.sol) - Set of uint256 values

## Key Features

- **O(1) Operations**: All operations (insert, remove, exists) have constant time complexity
- **Enumeration Support**: Iterate through set members by index
- **Uniqueness Enforcement**: Duplicate values are automatically prevented
- **Gas Optimized**: Efficient storage patterns minimize gas costs
- **Swap-and-Pop Deletion**: Maintains array compactness without shifting elements
- **Type Safety**: Separate libraries for different data types ensure type safety

## Common Architecture

All three set libraries follow the same architectural pattern:

### Storage Structure

```solidity
struct Set {
    mapping(T => uint256) keyPointers;  // T = address, bytes32, or uint256
    T[] keyList;
}
```

**Components:**
- `keyPointers`: Maps each key to its position in the keyList array
- `keyList`: Dynamic array containing all set members for enumeration

### Design Pattern

The libraries use a **bidirectional mapping** pattern:
1. **Key → Index**: `keyPointers` mapping provides O(1) existence checks and position lookup
2. **Index → Key**: `keyList` array enables enumeration and direct access by index

## Core Functions

All three libraries implement the same interface with type-specific parameters:

### `insert(Set storage self, T key)`

Adds a new key to the set.

**Parameters:**
- `self`: Storage pointer to the Set
- `key`: Value to insert (address, bytes32, or uint256)

**Requirements:**
- Key must not already exist in the set
- Reverts with error message if duplicate key is inserted

**Gas Cost:** ~20,000-25,000 gas

**Example:**
```solidity
AddressSet.Set storage addresses;
AddressSet.insert(addresses, 0x1234567890123456789012345678901234567890);
```

### `remove(Set storage self, T key)`

Removes a key from the set using swap-and-pop optimization.

**Parameters:**
- `self`: Storage pointer to the Set
- `key`: Value to remove

**Algorithm:**
1. Check if key exists (returns early if not)
2. Get position of key to remove
3. If not the last element, swap with last element
4. Update pointer for moved element
5. Delete key pointer and pop last element

**Gas Cost:** ~15,000-20,000 gas

**Example:**
```solidity
UInt256Set.remove(tokenIds, 12345);
```

### `exists(Set storage self, T key)`

Checks if a key exists in the set.

**Parameters:**
- `self`: Storage pointer to the Set
- `key`: Value to check

**Returns:**
- `true` if key exists, `false` otherwise

**Gas Cost:** ~500-1,000 gas

**Algorithm:**
```solidity
if (self.keyList.length == 0) return false;
return self.keyList[self.keyPointers[key]] == key;
```

### `count(Set storage self)`

Returns the number of elements in the set.

**Returns:**
- Number of elements in the set

**Gas Cost:** ~200-300 gas

### `keyAtIndex(Set storage self, uint256 index)`

Retrieves a key by its index position.

**Parameters:**
- `index`: Position in the set (0-based)

**Returns:**
- Key at the specified index

**Requirements:**
- Index must be less than `count()`

**Gas Cost:** ~300-500 gas

## Usage Examples

### AddressSet - Managing Whitelisted Addresses

```solidity
import { AddressSet } from "./libraries/AddressSet.sol";

contract Whitelist {
    using AddressSet for AddressSet.Set;
    
    AddressSet.Set private whitelistedAddresses;
    
    function addToWhitelist(address user) external onlyOwner {
        whitelistedAddresses.insert(user);
    }
    
    function removeFromWhitelist(address user) external onlyOwner {
        whitelistedAddresses.remove(user);
    }
    
    function isWhitelisted(address user) external view returns (bool) {
        return whitelistedAddresses.exists(user);
    }
    
    function getWhitelistSize() external view returns (uint256) {
        return whitelistedAddresses.count();
    }
    
    function getWhitelistedAddress(uint256 index) external view returns (address) {
        return whitelistedAddresses.keyAtIndex(index);
    }
}
```

### Bytes32Set - Managing Content Hashes

```solidity
import { Bytes32Set } from "./libraries/Bytes32Set.sol";

contract ContentRegistry {
    using Bytes32Set for Bytes32Set.Set;
    
    Bytes32Set.Set private contentHashes;
    
    function registerContent(bytes32 contentHash) external {
        require(!contentHashes.exists(contentHash), "Content already registered");
        contentHashes.insert(contentHash);
    }
    
    function removeContent(bytes32 contentHash) external onlyOwner {
        contentHashes.remove(contentHash);
    }
    
    function isContentRegistered(bytes32 contentHash) external view returns (bool) {
        return contentHashes.exists(contentHash);
    }
    
    function getAllContentHashes() external view returns (bytes32[] memory) {
        uint256 length = contentHashes.count();
        bytes32[] memory hashes = new bytes32[](length);
        
        for (uint256 i = 0; i < length; i++) {
            hashes[i] = contentHashes.keyAtIndex(i);
        }
        
        return hashes;
    }
}
```

### UInt256Set - Managing Token Collections

```solidity
import { UInt256Set } from "./libraries/UInt256Set.sol";

contract TokenCollection {
    using UInt256Set for UInt256Set.Set;
    
    mapping(address => UInt256Set.Set) private userTokens;
    
    function addTokenToUser(address user, uint256 tokenId) external {
        userTokens[user].insert(tokenId);
    }
    
    function removeTokenFromUser(address user, uint256 tokenId) external {
        userTokens[user].remove(tokenId);
    }
    
    function userOwnsToken(address user, uint256 tokenId) external view returns (bool) {
        return userTokens[user].exists(tokenId);
    }
    
    function getUserTokenCount(address user) external view returns (uint256) {
        return userTokens[user].count();
    }
    
    function getUserTokenAtIndex(address user, uint256 index) external view returns (uint256) {
        return userTokens[user].keyAtIndex(index);
    }
    
    function getUserTokens(address user) external view returns (uint256[] memory) {
        UInt256Set.Set storage tokens = userTokens[user];
        uint256 length = tokens.count();
        uint256[] memory tokenIds = new uint256[](length);
        
        for (uint256 i = 0; i < length; i++) {
            tokenIds[i] = tokens.keyAtIndex(i);
        }
        
        return tokenIds;
    }
}
```

## Advanced Usage Patterns

### Set Operations

```solidity
// Union of two sets
function unionSets(
    UInt256Set.Set storage setA,
    UInt256Set.Set storage setB
) internal view returns (uint256[] memory) {
    UInt256Set.Set memory result;
    
    // Add all elements from setA
    for (uint256 i = 0; i < setA.count(); i++) {
        uint256 element = setA.keyAtIndex(i);
        if (!result.exists(element)) {
            result.insert(element);
        }
    }
    
    // Add all elements from setB
    for (uint256 i = 0; i < setB.count(); i++) {
        uint256 element = setB.keyAtIndex(i);
        if (!result.exists(element)) {
            result.insert(element);
        }
    }
    
    // Convert to array
    uint256[] memory unionArray = new uint256[](result.count());
    for (uint256 i = 0; i < result.count(); i++) {
        unionArray[i] = result.keyAtIndex(i);
    }
    
    return unionArray;
}
```

### Batch Operations

```solidity
function batchInsert(AddressSet.Set storage set, address[] calldata addresses) external {
    for (uint256 i = 0; i < addresses.length; i++) {
        if (!set.exists(addresses[i])) {
            set.insert(addresses[i]);
        }
    }
}

function batchRemove(AddressSet.Set storage set, address[] calldata addresses) external {
    for (uint256 i = 0; i < addresses.length; i++) {
        if (set.exists(addresses[i])) {
            set.remove(addresses[i]);
        }
    }
}
```

## Performance Characteristics

### Time Complexity
- **Insert**: O(1) - Constant time regardless of set size
- **Remove**: O(1) - Swap-and-pop maintains constant time
- **Exists**: O(1) - Direct mapping lookup
- **Count**: O(1) - Array length property
- **KeyAtIndex**: O(1) - Direct array access
- **Enumeration**: O(n) - Linear in set size

### Gas Costs (Approximate)

| Operation | Cold Storage | Warm Storage |
|-----------|--------------|--------------|
| Insert | 22,000-25,000 | 5,000-8,000 |
| Remove | 15,000-20,000 | 3,000-5,000 |
| Exists | 800-2,100 | 100-800 |
| Count | 200-400 | 100-200 |
| KeyAtIndex | 300-800 | 100-300 |

### Storage Costs
- **Per Element**: 2 storage slots (1 for mapping, 1 for array)
- **Set Overhead**: Minimal (just array length)
- **Memory Efficiency**: Compact storage with no gaps

## Security Considerations

### Input Validation
- **Duplicate Prevention**: All libraries prevent duplicate insertions
- **Existence Checks**: Remove operations handle non-existent keys gracefully
- **Index Bounds**: KeyAtIndex should validate index bounds in calling code

### Gas Limit Considerations
- **Large Sets**: Enumeration operations may hit gas limits for very large sets
- **Batch Operations**: Consider gas limits when processing multiple elements
- **State Changes**: Multiple insertions/removals in single transaction

### Best Practices

```solidity
// Always check bounds when enumerating
function safeKeyAtIndex(UInt256Set.Set storage set, uint256 index) 
    internal view returns (uint256) {
    require(index < set.count(), "Index out of bounds");
    return set.keyAtIndex(index);
}

// Use exists() before remove() for safety
function safeRemove(AddressSet.Set storage set, address key) internal {
    if (set.exists(key)) {
        set.remove(key);
    }
}

// Paginate large enumerations
function getPaginatedKeys(
    UInt256Set.Set storage set,
    uint256 offset,
    uint256 limit
) internal view returns (uint256[] memory) {
    uint256 total = set.count();
    require(offset < total, "Offset out of bounds");
    
    uint256 end = offset + limit;
    if (end > total) {
        end = total;
    }
    
    uint256[] memory keys = new uint256[](end - offset);
    for (uint256 i = offset; i < end; i++) {
        keys[i - offset] = set.keyAtIndex(i);
    }
    
    return keys;
}
```

## Integration with Diamond Pattern

```solidity
// Storage struct for diamond pattern
struct SetStorage {
    AddressSet.Set admins;
    UInt256Set.Set activeTokens;
    Bytes32Set.Set validHashes;
}

// Storage position
bytes32 constant SET_STORAGE_POSITION = keccak256("diamond.gemforce.set.storage");

function setStorage() internal pure returns (SetStorage storage ds) {
    bytes32 position = SET_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
}
```

## Migration and Upgrades

### Data Migration
- Sets maintain their structure across upgrades
- New elements can be added without affecting existing data
- Consider gas costs when migrating large sets

### Version Compatibility
- Libraries are backward compatible
- Storage layout remains consistent
- Function signatures are stable

## Related Documentation

- [Diamond Library](diamond-lib.md) - Diamond pattern integration
- [Attribute Library](attribute-lib.md) - Token attribute management
- [Multi-Sale Library](multi-sale-lib.md) - Sale management with sets
- [Fee Distributor Library](fee-distributor-lib.md) - Revenue sharing with address sets

## Author Attribution

These set libraries are based on the work of **Rob Hitchens**, providing efficient and gas-optimized set implementations for Solidity smart contracts.