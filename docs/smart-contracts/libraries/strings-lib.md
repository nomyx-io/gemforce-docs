# StringsLib Library

The `StringsLib` library provides common utility functions for string manipulation within Solidity smart contracts. While Solidity has limited built-in string capabilities, this library extends its functionality, enabling operations such as string conversion, concatenation, and more, which are essential for handling metadata, logging, and human-readable data on-chain.

## Overview

`StringsLib` provides:

- **String Conversion**: Convert various data types (integers, addresses) into their string representations.
- **String Manipulation**: Basic operations like concatenation and substring handling.
- **Utility Functions**: Helper functions for handling string data.

## Key Features

### Type-to-String Conversion
- **`toString(uint256)`**: Convert unsigned integers to strings.
- **`toHexString(uint256)`**: Convert unsigned integers to hexadecimal string representations.
- **`toHexString(address)`**: Convert addresses to hexadecimal string representations.

### String Concatenation
- Provide efficient ways to combine multiple strings.

### Hashing and Comparison
- Though not explicit in all `StringsLib` implementations, good string libraries often include `keccak256` hashing and byte-based comparison.

## Library Definition

```solidity
library StringsLib {
    // Convert a uint256 to its string representation
    function toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) {
            return "0";
        }
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits--;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        return string(buffer);
    }

    // Convert a bytes32 to its string representation (hexadecimal)
    function toHexString(bytes32 value) internal pure returns (string memory) {
        bytes memory alphabet = "0123456789abcdef";
        bytes memory str = new bytes(64);
        for (uint256 i = 0; i < 32; i++) {
            str[i * 2] = alphabet[uint8(value[i] >> 4)];
            str[i * 2 + 1] = alphabet[uint8(value[i] & 0x0f)];
        }
        return string(str);
    }

    // Convert address to its string representation (hexadecimal)
    function toHexString(address value) internal pure returns (string memory) {
        bytes memory alphabet = "0123456789abcdef";
        bytes memory str = new bytes(40);
        uint256 temp = uint256(value);
        for (uint256 i = 0; i < 20; i++) {
            str[39 - i * 2] = alphabet[uint8(temp & 0x0f)];
            temp >>= 4;
            str[38 - i * 2] = alphabet[uint8(temp & 0x0f)];
            temp >>= 4;
        }
        return string(str);
    }

    // Convert bytes to their string representation (hexadecimal)
    function toHexString(bytes memory value) internal pure returns (string memory) {
        bytes memory alphabet = "0123456789abcdef";
        bytes memory str = new bytes(value.length * 2);
        for (uint256 i = 0; i < value.length; i++) {
            str[i * 2] = alphabet[uint8(value[i] >> 4)];
            str[i * 2 + 1] = alphabet[uint8(value[i] & 0x0f)];
        }
        return string(str);
    }

    // Concatenate two strings
    function concat(string memory a, string memory b) internal pure returns (string memory) {
        return string(abi.encodePacked(a, b));
    }

    // Concatenate three strings
    function concat(string memory a, string memory b, string memory c) internal pure returns (string memory) {
        return string(abi.encodePacked(a, b, c));
    }

    // Concatenate four strings
    function concat(string memory a, string memory b, string memory c, string memory d) internal pure returns (string memory) {
        return string(abi.encodePacked(a, b, c, d));
    }

    // Concatenate five strings
    function concat(string memory a, string memory b, string memory c, string memory d, string memory e) internal pure returns (string memory) {
        return string(abi.encodePacked(a, b, c, d, e));
    }
}
```

## Core Functions

### toString(uint256 value)
Converts a `uint256` integer to its decimal string representation. This is commonly used for generating human-readable token IDs or amounts in metadata.

**Parameters:**
- `value`: The `uint256` integer to convert.

**Returns:**
- `string`: The decimal string representation of the integer.

**Usage:**
```solidity
uint256 tokenId = 12345;
string memory tokenIdStr = StringsLib.toString(tokenId); // "12345"
```

### toHexString(address value)
Converts an `address` into its hexadecimal string representation without the "0x" prefix. This is useful for including addresses directly into URIs or other data fields.

**Parameters:**
- `value`: The `address` to convert.

**Returns:**
- `string`: The hexadecimal string representation of the address (e.g., "abcdef1234567890abcdef1234567890abcdef12").

**Usage:**
```solidity
address ownerAddress = 0x742d35Cc6634C0539Ff34f. . .;
string memory ownerAddrHex = StringsLib.toHexString(ownerAddress);
```

### concat(string a, string b)
Concatenates two strings into a single string. Overloaded versions allow concatenating up to five strings.

**Parameters:**
- `a`, `b`, ...: The strings to concatenate.

**Returns:**
- `string`: The combined string.

**Usage:**
```solidity
string memory prefix = "https://example.com/nft/";
uint256 tokenId = 99;
string memory suffix = ".json";

string memory uri = StringsLib.concat(prefix, StringsLib.toString(tokenId), suffix); // "https://example.com/nft/99.json"
```

## Security Considerations

### Gas Costs
- **String Manipulation**: String operations in Solidity, especially concatenation and conversions, are very gas-expensive. This is due to the dynamic memory allocation and copying involved. Use these functions judiciously and preferably for off-chain metadata generation or infrequent on-chain operations.
- **Memory Limits**: Be mindful of the maximum string length you intend to generate. Extremely long strings can hit block gas limits.

### Data Encoding/Decoding
- **ABI Encoding**: The `concat` functions rely on `abi.encodePacked`. While generally safe for concatenating primitive types that are already strings, be aware of how `abi.encodePacked` handles different types if you were to extend this library with more complex concatenations (e.g., it packs tightly without padding).

## Best Practices

### Off-chain Handling
1. **Prefer Off-chain**: For complex string formatting, especially for NFT metadata URIs or extensive logging, perform string operations off-chain and then upload the final string (or its hash/URI) to the blockchain.
2. **IPFS for Metadata**: Store large metadata strings (like SVG content or detailed JSON) on IPFS and only store the IPFS CID on-chain.

### Concatenation Efficiency
1. **Reduce Calls**: When concatenating multiple strings, use the overloaded `concat` function with more arguments (e.g., `concat(a, b, c)`) instead of chaining binary concatenations (`concat(a, concat(b, c))`) to potentially save gas.

## Integration Examples

### Smart Contract Usage

```solidity
import "../libraries/StringsLib.sol"; // Adjust path as needed

contract MyNFTMetadata {
    using StringsLib for uint256;
    using StringsLib for address;
    using StringsLib for string; // For concat

    string public baseURI = "ipfs://QmbP123abcDEF/";
    string public constant JSON_SUFFIX = ".json";

    function tokenURI(uint256 tokenId) public view returns (string memory) {
        return StringsLib.concat(baseURI, tokenId.toString(), JSON_SUFFIX);
    }

    function createLogEntry(address user, uint256 amount) public view returns (string memory) {
        string memory userHex = user.toHexString();
        string memory amountStr = amount.toString();
        return StringsLib.concat("User ", userHex, " minted ", amountStr, " tokens.");
    }
}
```

## Related Documentation

- [Solidity ABI Encoding and Decoding](https://docs.soliditylang.org/en/latest/abi-spec.html)
- [Gemforce Minter Facet](../interfaces/igemforce-minter-facet.md) (uses `StringsLib` for URIs)
- [SVG Templates Library](./svg-templates-lib.md) (can use `StringsLib` for SVG composition)