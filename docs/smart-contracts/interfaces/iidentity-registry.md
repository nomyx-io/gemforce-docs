# IIdentityRegistry Interface

The `IIdentityRegistry` interface defines the standard for a registry of identity contracts within the Gemforce ecosystem. This registry acts as a central lookup and management system for various identity contracts, allowing for consistent identity resolution and verification across the platform.

## Overview

`IIdentityRegistry` provides:

- **Identity Resolution**: A method to query and retrieve registered identity contract addresses.
- **Registration Management**: Functions to add, update, and remove identity entries in the registry.
- **Status Tracking**: Keep track of the status (e.g., active, suspended) of registered identities.
- **Event Logging**: Comprehensive event tracking for all registry operations.

## Key Features

### Identity Registration & Query
- **Register Identity**: Link an identity contract address to a user or entity.
- **Query Identity**: Retrieve an identity contract address associated with a given identifier.
- **Batch Registration**: Register multiple identities efficiently.

### Status and Access Control
- **Identity Status**: Manage and query the active status of registered identities.
- **Ownership/Admin Control**: Restrict registration and modification functions to authorized entities.
- **Verification Levels**: Potentially integrate with identity verification levels.

### Decentralized Identifiers (DIDs) Compatibility (Conceptual)
- Can be designed to store DID-like identifiers or map to them for broader interoperability.

## Interface Definition

```solidity
interface IIdentityRegistry {
    // Events
    event IdentityRegistered(
        bytes32 indexed identityHash,
        address indexed identityAddress,
        address indexed registrant,
        string metadataURI
    );
    event IdentityUpdated(
        bytes32 indexed identityHash,
        address indexed identityAddress,
        address indexed updater,
        string oldMetadataURI,
        string newMetadataURI
    );
    event IdentityRemoved(
        bytes32 indexed identityHash,
        address indexed identityAddress,
        address indexed remover
    );
    event IdentityStatusChanged(
        bytes32 indexed identityHash,
        address indexed identityAddress,
        bool newStatus,
        address indexed changer
    );

    // Structs
    struct IdentityEntry {
        address identityAddress;
        address registrant;
        string metadataURI;
        bool active;
        uint256 registeredAt;
        uint256 lastUpdated;
    }

    // Core Functions
    function registerIdentity(
        address _identityAddress,
        string calldata _metadataURI
    ) external returns (bytes32 identityHash);

    function registerIdentities(
        address[] calldata _identityAddresses,
        string[] calldata _metadataURIs
    ) external returns (bytes32[] memory identityHashes);

    function updateIdentity(
        bytes32 _identityHash,
        string calldata _newMetadataURI
    ) external;

    function removeIdentity(bytes32 _identityHash) external;

    function setIdentityStatus(
        bytes32 _identityHash,
        bool _newStatus
    ) external;

    // View Functions
    function getIdentityAddress(bytes32 _identityHash)
        external
        view
        returns (address);

    function getIdentityEntry(bytes32 _identityHash)
        external
        view
        returns (IdentityEntry memory);

    function isIdentityActive(bytes32 _identityHash)
        external
        view
        returns (bool);

    function getIdentityHash(address _identityAddress)
        external
        view
        returns (bytes32); // Reverse lookup for the hash

    function getRegisteredIdentityCount() external view returns (uint256);
    function getIdentityHashByIndex(uint256 index) external view returns (bytes32);
}
```

## Core Functions

### registerIdentity()
Registers a new identity contract address with an associated metadata URI. A unique `identityHash` is generated for lookup.

**Parameters:**
- `_identityAddress`: The address of the identity smart contract to register.
- `_metadataURI`: A URI pointing to additional off-chain metadata about the identity.

**Returns:**
- `bytes32`: A unique hash that identifies this registered identity.

**Usage:**
```solidity
// Register a new identity contract
bytes32 myIdentityHash = identityRegistry.registerIdentity(
    _newlyDeployedIdentityAddress,
    "ipfs://my-identity-meta.json"
);
```

### getIdentityAddress()
Retrieves the identity contract address associated with a given `identityHash`.

**Parameters:**
- `_identityHash`: The unique hash of the registered identity.

**Returns:**
- `address`: The address of the registered identity contract. Returns `address(0)` if not found.

### setIdentityStatus()
Changes the active status of a registered identity. This can be used to suspend or reactivate an identity.

**Parameters:**
- `_identityHash`: The hash of the identity whose status is to be changed.
- `_newStatus`: `true` for active, `false` for suspended.

## Implementation Example

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract IdentityRegistry is IIdentityRegistry, Ownable {
    // Mapping from identity hash to its entry details
    mapping(bytes32 => IdentityEntry) private _identityEntries;
    // Mapping from identity address to its hash for reverse lookup
    mapping(address => bytes32) private _identityAddressToHash;
    // Counter for total registered identities (and to generate unique hashes if needed)
    uint256 private _registeredIdentityCount;
    // Array to store all hashes for enumeration
    bytes32[] private _allIdentityHashes;

    constructor() {
        // Owner is set to deployer by default due to Ownable
    }

    function registerIdentity(
        address _identityAddress,
        string calldata _metadataURI
    ) external override onlyOwner returns (bytes32 identityHash) {
        require(_identityAddress != address(0), "Identity address cannot be zero");
        require(_identityAddressToHash[_identityAddress] == bytes32(0), "Identity already registered");

        identityHash = keccak256(abi.encodePacked(_identityAddress, block.timestamp, msg.sender)); // Generate unique hash
        
        _identityEntries[identityHash] = IdentityEntry({
            identityAddress: _identityAddress,
            registrant: msg.sender,
            metadataURI: _metadataURI,
            active: true,
            registeredAt: block.timestamp,
            lastUpdated: block.timestamp
        });
        _identityAddressToHash[_identityAddress] = identityHash;
        _registeredIdentityCount++;
        _allIdentityHashes.push(identityHash);

        emit IdentityRegistered(identityHash, _identityAddress, msg.sender, _metadataURI);
    }

    function registerIdentities(
        address[] calldata _identityAddresses,
        string[] calldata _metadataURIs
    ) external override onlyOwner returns (bytes32[] memory identityHashes) {
        require(_identityAddresses.length == _metadataURIs.length, "Array length mismatch");
        identityHashes = new bytes32[](_identityAddresses.length);
        for (uint256 i = 0; i < _identityAddresses.length; i++) {
            identityHashes[i] = registerIdentity(_identityAddresses[i], _metadataURIs[i]);
        }
    }

    function updateIdentity(
        bytes32 _identityHash,
        string calldata _newMetadataURI
    ) external override onlyOwner {
        IdentityEntry storage entry = _identityEntries[_identityHash];
        require(entry.identityAddress != address(0), "Identity hash not found");

        string memory oldMetadataURI = entry.metadataURI;
        entry.metadataURI = _newMetadataURI;
        entry.lastUpdated = block.timestamp;

        emit IdentityUpdated(
            _identityHash,
            entry.identityAddress,
            msg.sender,
            oldMetadataURI,
            _newMetadataURI
        );
    }

    function removeIdentity(bytes32 _identityHash) external override onlyOwner {
        IdentityEntry storage entry = _identityEntries[_identityHash];
        require(entry.identityAddress != address(0), "Identity hash not found");

        address removedAddress = entry.identityAddress;

        delete _identityAddressToHash[removedAddress];
        delete _identityEntries[_identityHash]; // Clear storage for the hash
        _registeredIdentityCount--;

        // Remove from dynamic array, maintain order is not necessary
        for (uint256 i = 0; i < _allIdentityHashes.length; i++) {
            if (_allIdentityHashes[i] == _identityHash) {
                _allIdentityHashes[i] = _allIdentityHashes[_allIdentityHashes.length - 1]; // Swap with last
                _allIdentityHashes.pop(); // Pop last element
                break;
            }
        }

        emit IdentityRemoved(_identityHash, removedAddress, msg.sender);
    }

    function setIdentityStatus(
        bytes32 _identityHash,
        bool _newStatus
    ) external override onlyOwner {
        IdentityEntry storage entry = _identityEntries[_identityHash];
        require(entry.identityAddress != address(0), "Identity hash not found");
        require(entry.active != _newStatus, "Status is already the same");

        entry.active = _newStatus;
        entry.lastUpdated = block.timestamp;
        
        emit IdentityStatusChanged(_identityHash, entry.identityAddress, _newStatus, msg.sender);
    }

    function getIdentityAddress(bytes32 _identityHash)
        external
        view
        override
        returns (address)
    {
        return _identityEntries[_identityHash].identityAddress;
    }

    function getIdentityEntry(bytes32 _identityHash)
        external
        view
        override
        returns (IdentityEntry memory)
    {
        return _identityEntries[_identityHash];
    }

    function isIdentityActive(bytes32 _identityHash)
        external
        view
        override
        returns (bool)
    {
        return _identityEntries[_identityHash].active;
    }

    function getIdentityHash(address _identityAddress)
        external
        view
        override
        returns (bytes32)
    {
        return _identityAddressToHash[_identityAddress];
    }

    function getRegisteredIdentityCount() external view override returns (uint256) {
        return _registeredIdentityCount;
    }

    function getIdentityHashByIndex(uint256 index) external view override returns (bytes32) {
        require(index < _allIdentityHashes.length, "Index out of bounds");
        return _allIdentityHashes[index];
    }
}
```

## Security Considerations

### Access Control
- **`onlyOwner`**: All functions that modify the registry (`registerIdentity`, `updateIdentity`, `removeIdentity`, `setIdentityStatus`) are restricted to the contract owner to maintain integrity and prevent unauthorized modifications.
- **Role-Based Access Control**: For more complex scenarios, consider implementing a role-based access control (RBAC) system instead of a single owner.

### Data Integrity
- **Zero Address Check**: Prevent registration of `address(0)` as an identity.
- **Duplicate Registration**: Ensure that an identity address can only be registered once.
- **Hash Collisions**: While `keccak256` is highly collision-resistant, rely on the _generated_ hash to be unique within the contract's scope, not solely on external inputs.

### Gas Efficiency
- **Array Iteration**: `removeIdentity` iterates through `_allIdentityHashes` to remove an element. For a very large number of identities, this operation could become gas-expensive. Consider alternative data structures or a flag-based removal if enumeration isn't strictly necessary or if the number of removals is high.

## Best Practices

### Identity Hashing
1. **Deterministic Hashing**: The `identityHash` generated by the contract (e.g., using `keccak256(abi.encodePacked(_identityAddress, block.timestamp, msg.sender))`) should include entropy to make it unique and hard to guess, while still allowing for a unique identifier for each registered identity.

### Events
1. **Comprehensive Events**: Emit events for all state-changing operations to enable off-chain indexing and monitoring.

### Modularity
1. **Separation of Concerns**: The registry focuses solely on mapping identifiers to identity contracts. Specific identity logic (e.g., claims, keys) should reside within the `IIdentity` contracts linked by this registry.

## Integration Examples

### Frontend Integration (TypeScript via Ethers.js)

```typescript
import { ethers, Contract } from 'ethers';
import IdentityRegistryABI from './IdentityRegistry.json'; // ABI of the IIdentityRegistry

const REGISTRY_ADDRESS = "0x..."; // Your deployed IdentityRegistry contract address

const getSigner = () => new ethers.providers.Web3Provider(window.ethereum).getSigner();
const getIdentityRegistryContract = () => new Contract(REGISTRY_ADDRESS, IdentityRegistryABI, getSigner());

async function registerNewIdentity(identityAddress: string, metadataURI: string): Promise<string> {
    try {
        const registry = getIdentityRegistryContract();
        const tx = await registry.registerIdentity(identityAddress, metadataURI);
        const receipt = await tx.wait();
        const event = receipt.events?.find((e: any) => e.event === 'IdentityRegistered');
        return event?.args?.identityHash;
    } catch (error) {
        console.error("Error registering identity:", error);
        throw error;
    }
}

async function getIdentityAddressByHash(identityHash: string): Promise<string> {
    try {
        const registry = getIdentityRegistryContract();
        const address = await registry.getIdentityAddress(identityHash);
        return address;
    } catch (error) {
        console.error("Error getting identity address:", error);
        throw error;
    }
}

async function checkIdentityStatus(identityHash: string): Promise<boolean> {
    try {
        const registry = getIdentityRegistryContract();
        const isActive = await registry.isIdentityActive(identityHash);
        return isActive;
    } catch (error) {
        console.error("Error checking identity status:", error);
        throw error;
    }
}

// Example usage
// const newIdHash = await registerNewIdentity("0xIdentityContractAddress", "http://my.identity/profile.json");
// const retrievedAddress = await getIdentityAddressByHash(newIdHash);
// const status = await checkIdentityStatus(newIdHash);
```

### Backend Integration (Node.js for a centralized identity service)

```javascript
const Web3 = require('web3');
const IdentityRegistryABI = require('./IdentityRegistry.json').abi;

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const registryAddress = '0x...'; // Address of your deployed IdentityRegistry contract
const ownerPrivateKey = 'YOUR_CONTRACT_OWNER_PRIVATE_KEY'; // Private key of the contract owner

const identityRegistryContract = new web3.eth.Contract(IdentityRegistryABI, registryAddress);
const ownerAccount = web3.eth.accounts.privateKeyToAccount(ownerPrivateKey);
web3.eth.accounts.wallet.add(ownerAccount);

async function registerNewIdentityBackend(identityAddress, metadataURI) {
    try {
        console.log(`Registering identity ${identityAddress} with URI ${metadataURI}...`);
        const tx = identityRegistryContract.methods.registerIdentity(identityAddress, metadataURI);
        const gasLimit = await tx.estimateGas({ from: ownerAccount.address });
        const receipt = await tx.send({ from: ownerAccount.address, gas: gasLimit });
        console.log(`Identity registered. Transaction hash: ${receipt.transactionHash}`);
        // Extract identity hash from event logs if necessary
        return receipt;
    } catch (error) {
        console.error("Backend: Error registering identity:", error);
        throw error;
    }
}

async function suspendIdentityBackend(identityHash) {
    try {
        console.log(`Suspending identity with hash ${identityHash}...`);
        const tx = identityRegistryContract.methods.setIdentityStatus(identityHash, false);
        const gasLimit = await tx.estimateGas({ from: ownerAccount.address });
        const receipt = await tx.send({ from: ownerAccount.address, gas: gasLimit });
        console.log(`Identity suspended. Transaction hash: ${receipt.transactionHash}`);
        return receipt;
    } catch (error) {
        console.error("Backend: Error suspending identity:", error);
        throw error;
    }
}

// Example usage
// registerNewIdentityBackend("0xAnotherIdentityContract", "https://api.example.com/identity/2");
// suspendIdentityBackend("0xabcdef1234567890..."); // Use a previously obtained identity hash
```

## Related Documentation

- [IIdentity Interface](./iidentity.md)
- [Identity Factory](../identity-factory.md)
- [Ownable Contract](https://docs.openzeppelin.com/contracts/5.x/api/access#Ownable)
- [Decentralized Identifiers (DIDs) (External)](https://www.w3.org/TR/did-core/)

## Standards Compliance

- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.