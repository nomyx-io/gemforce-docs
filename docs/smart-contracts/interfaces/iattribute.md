# IAttribute Interface

The `IAttribute` interface defines the standard for managing dynamic attributes associated with tokens or other entities within the Gemforce ecosystem. This interface provides a flexible and extensible way to store, retrieve, and update various properties or characteristics without modifying the core token contract logic.

## Overview

`IAttribute` provides:

- **Dynamic Attribute Management**: Store and retrieve custom attributes for any entity (e.g., token IDs, identity hashes).
- **Attribute Types**: Support for different data types (e.g., string, uint, bool, bytes).
- **Versioning (Optional)**: Can support attribute versioning for historical data.
- **Access Control**: Define permissions for setting and updating attributes.
- **Event Logging**: Comprehensive event tracking for attribute changes.

## Key Features

### Attribute Assignment & Retrieval
- **`setAttribute()`**: Assign a specific value to an attribute key for an entity.
- **`getAttribute()`**: Retrieve the value of an attribute for an entity.
- **Type-Specific Getters**: Convenience functions for specific data types (e.g., `getStringAttribute`, `getUintAttribute`).

### Attribute Ownership & Permissions
- **Entity Identification**: Attributes are linked to a unique entity identifier (e.g., `tokenId` or `identityHash`).
- **Authorization**: Control who can set and modify attributes (e.g., only the entity owner, specific roles).

### Data Structure Flexibility
- **Key-Value Pairs**: Attributes are typically stored as key-value pairs.
- **Bytes for Flexibility**: Using `bytes` for raw storage offers maximum flexibility for encoding complex data.

## Interface Definition

```solidity
interface IAttribute {
    // Events
    event AttributeSet(
        bytes32 indexed entityId,
        string indexed key,
        bytes oldValue,
        bytes newValue
    );
    event AttributeRemoved(
        bytes32 indexed entityId,
        string indexed key,
        bytes removedValue
    );

    // Structs
    struct AttributeEntry {
        bytes value;
        address setter;
        uint256 timestamp;
    }

    // Core Functions to Set/Get Attributes
    function setAttribute(bytes32 entityId, string calldata key, bytes calldata value) external;
    function getAttribute(bytes32 entityId, string calldata key) external view returns (bytes memory);

    // Type-specific Getters (for convenience, can be implemented as internal decoding)
    function getBoolAttribute(bytes32 entityId, string calldata key) external view returns (bool);
    function getUintAttribute(bytes32 entityId, string calldata key) external view returns (uint256);
    function getStringAttribute(bytes32 entityId, string calldata key) external view returns (string memory);
    function getAddressAttribute(bytes32 entityId, string calldata key) external view returns (address);
    function getBytes32Attribute(bytes32 entityId, string calldata key) external view returns (bytes32);

    // Query Functions
    function hasAttribute(bytes32 entityId, string calldata key) external view returns (bool);
    function getAttributeSetter(bytes32 entityId, string calldata key) external view returns (address);
    function getAttributeTimestamp(bytes32 entityId, string calldata key) external view returns (uint256);
}
```

## Core Functions

### setAttribute()
Sets or updates an attribute for a given entity. If the attribute already exists, its value is overwritten.

**Parameters:**
- `entityId`: A unique identifier for the entity (e.g., `keccak256(abi.encodePacked(tokenId))`).
- `key`: The name of the attribute (e.g., "color", "powerLevel", "status").
- `value`: The value of the attribute, encoded as `bytes`.

**Access Control:**
- This function should typically be restricted to the owner of the `entityId` or an authorized contract.

**Usage:**
```solidity
bytes32 myTokenIdHash = keccak256(abi.encodePacked(123)); // Hash of token ID 123
attributeContract.setAttribute(myTokenIdHash, "color", abi.encodePacked("red"));
attributeContract.setAttribute(myTokenIdHash, "rarity", abi.encodePacked(uint256(5)));
```

### getAttribute()
Retrieves the raw `bytes` value of an attribute for a specified entity and key.

**Parameters:**
- `entityId`: The unique identifier of the entity.
- `key`: The name of the attribute.

**Returns:**
- `bytes`: The raw `bytes` value of the attribute.

### Type-specific getters (e.g., `getUintAttribute()`)
These functions provide convenience by decoding the raw `bytes` value into a specific Solidity type. They internally call `getAttribute()` and then `abi.decode()`.

**Parameters:**
- `entityId`: The unique identifier of the entity.
- `key`: The name of the attribute.

**Returns:**
- The decoded value in the specified type (e.g., `uint256` for `getUintAttribute`).

## Implementation Example

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

// This is a simplified Attribute storage contract.
// In a real Gemforce Diamond context, this could be a facet.
contract AttributeStorage is IAttribute, Ownable {
    // entityId => key => AttributeEntry
    mapping(bytes32 => mapping(string => AttributeEntry)) private _attributes;

    constructor() {
        // Owner set to deployer by default due to Ownable
    }

    // Modifier to check if the caller is authorized to set attributes for this entityID.
    // In a production Diamond, this would be highly customized based on entity ownership (e.g., tokenId belongs to msg.sender).
    // For this generic example, we'll just allow the contract owner to set anything.
    modifier onlyEntityOwnerOrApproved(bytes32 entityId) {
        // Example check: only contract owner can set.
        // In a real system, you might check if msg.sender owns the NFT for this entityId.
        // Or if it's an approved operator.
        require(msg.sender == owner(), "Not authorized to set attribute");
        _;
    }

    function setAttribute(
        bytes32 entityId,
        string calldata key,
        bytes calldata value
    ) external override onlyEntityOwnerOrApproved(entityId) {
        bytes memory oldValue = _attributes[entityId][key].value;
        _attributes[entityId][key] = AttributeEntry({
            value: value,
            setter: msg.sender,
            timestamp: block.timestamp
        });
        emit AttributeSet(entityId, key, oldValue, value);
    }

    function getAttribute(bytes32 entityId, string calldata key) external view override returns (bytes memory) {
        return _attributes[entityId][key].value;
    }

    function getBoolAttribute(bytes32 entityId, string calldata key) external view override returns (bool) {
        bytes memory val = _attributes[entityId][key].value;
        require(val.length == 1, "Invalid bool attribute data");
        return abi.decode(val, (bool));
    }

    function getUintAttribute(bytes32 entityId, string calldata key) external view override returns (uint256) {
        bytes memory val = _attributes[entityId][key].value;
        require(val.length <= 32, "Invalid uint attribute data"); // uint256 is 32 bytes
        return abi.decode(val, (uint256));
    }

    function getStringAttribute(bytes32 entityId, string calldata key) external view override returns (string memory) {
        bytes memory val = _attributes[entityId][key].value;
        return abi.decode(val, (string));
    }

    function getAddressAttribute(bytes32 entityId, string calldata key) external view override returns (address) {
        bytes memory val = _attributes[entityId][key].value;
        require(val.length == 20, "Invalid address attribute data");
        return abi.decode(val, (address));
    }

    function getBytes32Attribute(bytes32 entityId, string calldata key) external view override returns (bytes32) {
        bytes memory val = _attributes[entityId][key].value;
        require(val.length == 32, "Invalid bytes32 attribute data");
        return abi.decode(val, (bytes32));
    }

    function hasAttribute(bytes32 entityId, string calldata key) external view override returns (bool) {
        return _attributes[entityId][key].value.length > 0;
    }

    function getAttributeSetter(bytes32 entityId, string calldata key) external view override returns (address) {
        return _attributes[entityId][key].setter;
    }

    function getAttributeTimestamp(bytes32 entityId, string calldata key) external view override returns (uint256) {
        return _attributes[entityId][key].timestamp;
    }

    function removeAttribute(bytes32 entityId, string calldata key) external onlyEntityOwnerOrApproved(entityId) {
        bytes memory removedValue = _attributes[entityId][key].value;
        delete _attributes[entityId][key];
        emit AttributeRemoved(entityId, key, removedValue);
    }
}
```

## Security Considerations

### Access Control
- **Authorization for `setAttribute`**: This is paramount. Who can set or modify an attribute for a given `entityId`?
    - If `entityId` refers to an NFT (`tokenId`), typically only the NFT owner or an approved operator should be able to set its attributes.
    - If `entityId` refers to an identity, only the identity owner or its management keys should be authorized.
    - The provided example uses `onlyOwner`, which is a simplistic approach for a single contract owner. In a Diamond setup, this would be handled by the `OwnershipFacet` or a custom access control logic unique to the entity type.
- **Reentrancy**: Not directly applicable to this contract as it primarily stores and retrieves data. No external calls are made that could lead to reentrancy.

### Data Integrity
- **Encoding/Decoding**: Ensure consistency in `abi.encode` when setting attributes and `abi.decode` when getting them via type-specific functions. Mismatched types can lead to errors or unexpected behavior.
- **Input Validation**: Validate input `key` and `value` to prevent excessively long strings or malicious data.

## Best Practices

### Entity Identification (`entityId`)
1. **Hashing**: Use `bytes32` derived from `keccak256(abi.encodePacked(something))` for `entityId` to consistently refer to the entity whose attributes are being managed. This could be a `tokenId`, an `identityAddress`, a `projectHash`, etc.
2. **Clarity**: Clearly document what `entityId` represents in the context of your contract.

### Data Encoding
1. **`abi.encodePacked`**: For simple, fixed-length types (like `uint`, `address`, `bool`), `abi.encodePacked` can be more gas-efficient than `abi.encode`.
2. **Complex Data**: For complex data structures, encode them into `bytes` using `abi.encode` or a custom serialization (e.g., RLP) before storing.

### Gas Efficiency
1. **`bytes` vs. `string`**: Storing `bytes` is generally more gas-efficient than `string` if you control the encoding/decoding.
2. **Minimize Storage Writes**: Avoid unnecessary `setAttribute` calls; only update when truly needed.

## Integration Examples

### Frontend Integration (React with Ethers.js)

```typescript
import React, { useState } from 'react';
import { ethers, Contract } from 'ethers';
import AttributeABI from './AttributeStorage.json'; // ABI for IAttribute

const ATTRIBUTE_CONTRACT_ADDRESS = "0x..."; // Your deployed AttributeStorage contract or Diamond

const getSigner = () => new ethers.providers.Web3Provider(window.ethereum).getSigner();
const getAttributeContract = () => new Contract(ATTRIBUTE_CONTRACT_ADDRESS, AttributeABI, getSigner());

interface SetAttributeProps {
    entityId: string; // Hex string '0x...'
    key: string;
    value: string; // The value to set (e.g., "red", "5", "true")
    valueType: 'string' | 'uint' | 'bool' | 'address' | 'bytes32'; // Type hint for encoding
}

async function handleSetAttribute({ entityId, key, value, valueType }: SetAttributeProps) {
    try {
        const attributeContract = getAttributeContract();
        let encodedValue: Uint8Array;

        switch (valueType) {
            case 'string':
                encodedValue = ethers.utils.toUtf8Bytes(value);
                break;
            case 'uint':
                encodedValue = ethers.utils.arrayify(ethers.utils.hexlify(ethers.BigNumber.from(value)));
                break;
            case 'bool':
                encodedValue = ethers.utils.arrayify(value === 'true' ? '0x01' : '0x00');
                break;
            case 'address':
                encodedValue = ethers.utils.arrayify(ethers.utils.getAddress(value));
                break;
            case 'bytes32':
                encodedValue = ethers.utils.arrayify(value);
                break;
            default:
                throw new Error("Unsupported value type");
        }

        const tx = await attributeContract.setAttribute(
            entityId,
            key,
            encodedValue
        );
        await tx.wait();
        alert(`Attribute '${key}' set for entity '${entityId}' to '${value}' successfully!`);
    } catch (error) {
        console.error("Error setting attribute:", error);
        alert("Failed to set attribute. Check console for details.");
    }
}

interface GetAttributeProps {
    entityId: string;
    key: string;
    valueType: 'string' | 'uint' | 'bool' | 'address' | 'bytes32';
}

async function handleGetAttribute({ entityId, key, valueType }: GetAttributeProps) {
    try {
        const attributeContract = getAttributeContract();
        let result;
        switch (valueType) {
            case 'string':
                result = await attributeContract.getStringAttribute(entityId, key);
                break;
            case 'uint':
                result = (await attributeContract.getUintAttribute(entityId, key)).toString();
                break;
            case 'bool':
                result = await attributeContract.getBoolAttribute(entityId, key);
                break;
            case 'address':
                result = await attributeContract.getAddressAttribute(entityId, key);
                break;
            case 'bytes32':
                result = await attributeContract.getBytes32Attribute(entityId, key);
                break;
            default:
                throw new Error("Unsupported value type");
        }
        alert(`Attribute '${key}' for entity '${entityId}' is: ${result}`);
        console.log(`Attribute '${key}' for entity '${entityId}' (${valueType}):`, result);
    } catch (error) {
        console.error("Error getting attribute:", error);
        alert("Failed to get attribute. Check console for details.");
    }
}

// Example usage in component:
// <button onClick={() => handleSetAttribute({ entityId: "0x...", key: "color", value: "blue", valueType: "string" })}>Set Color</button>
// <button onClick={() => handleGetAttribute({ entityId: "0x...", key: "color", valueType: "string" })}>Get Color</button>
```

### Backend Integration (Node.js with Web3.js)

```javascript
const Web3 = require('web3');
const AttributeABI = require('./AttributeStorage.json').abi; // ABI of IAttribute

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const attributeContractAddress = '0x...'; // Your deployed AttributeStorage contract

const attributeContract = new web3.eth.Contract(AttributeABI, attributeContractAddress);
const adminAccount = web3.eth.accounts.privateKeyToAccount('YOUR_ADMIN_PRIVATE_KEY');
web3.eth.accounts.wallet.add(adminAccount);

async function setTokenAttribute(tokenId, key, value, type) {
    try {
        const entityId = web3.utils.keccak256(web3.eth.abi.encodePacked(tokenId));
        let encodedValue;

        switch (type) {
            case 'string':
                encodedValue = web3.eth.abi.encodeParameter('string', value);
                break;
            case 'uint':
                encodedValue = web3.eth.abi.encodeParameter('uint256', value);
                break;
            case 'bool':
                encodedValue = web3.eth.abi.encodeParameter('bool', value);
                break;
            case 'address':
                encodedValue = web3.eth.abi.encodeParameter('address', value);
                break;
            case 'bytes32':
                encodedValue = web3.eth.abi.encodeParameter('bytes32', value);
                break;
            default:
                throw new Error("Unsupported type for encoding");
        }
        
        // Remove '0x' prefix for bytes if present, as setAttribute expects raw bytes
        encodedValue = encodedValue.startsWith('0x') ? encodedValue.substring(2) : encodedValue;
        encodedValue = web3.utils.hexToBytes('0x' + encodedValue); // Convert to bytes array

        const tx = attributeContract.methods.setAttribute(entityId, key, encodedValue);
        const gasLimit = await tx.estimateGas({ from: adminAccount.address });
        const receipt = await tx.send({ from: adminAccount.address, gas: gasLimit });

        console.log(`Attribute set for token ${tokenId}, key '${key}'. Tx Hash: ${receipt.transactionHash}`);
        return receipt;
    } catch (error) {
        console.error("Backend: Error setting attribute:", error);
        throw error;
    }
}

async function getTokenAttribute(tokenId, key, type) {
    try {
        const entityId = web3.utils.keccak256(web3.eth.abi.encodePacked(tokenId));
        let result;
        switch (type) {
            case 'string':
                result = await attributeContract.methods.getStringAttribute(entityId, key).call();
                break;
            case 'uint':
                result = await attributeContract.methods.getUintAttribute(entityId, key).call();
                break;
            case 'bool':
                result = await attributeContract.methods.getBoolAttribute(entityId, key).call();
                break;
            case 'address':
                result = await attributeContract.methods.getAddressAttribute(entityId, key).call();
                break;
            case 'bytes32':
                result = await attributeContract.methods.getBytes32Attribute(entityId, key).call();
                break;
            default:
                throw new Error("Unsupported type for decoding");
        }
        console.log(`Attribute for token ${tokenId}, key '${key}' is: ${result}`);
        return result;
    } catch (error) {
        console.error("Backend: Error getting attribute:", error);
        throw error;
    }
}

// Example usage
// setTokenAttribute(123, "background", "forest", "string");
// getTokenAttribute(123, "background", "string");
```

## Related Documentation

- [Attribute Library](../libraries/attribute-lib.md)
- [Ownable Contract](https://docs.openzeppelin.com/contracts/5.x/api/access#Ownable)
- [Solidity ABI Encoding and Decoding](https://docs.soliditylang.org/en/latest/abi-spec.html)

## Standards Compliance

- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.