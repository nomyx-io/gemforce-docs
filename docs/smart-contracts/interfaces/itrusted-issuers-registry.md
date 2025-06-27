# ITrustedIssuersRegistry Interface

The `ITrustedIssuersRegistry` interface defines the standard for a registry of trusted claim issuers within the Gemforce ecosystem. This registry allows smart contracts and off-chain systems to verify the authenticity and trustworthiness of entities that issue claims, attestations, or credentials. It is a critical component for identity and verification systems built on the platform.

## Overview

`ITrustedIssuersRegistry` provides:

- **Issuer Registration**: A mechanism to register and manage trusted issuer addresses.
- **Trust Verification**: Functions to check if a given address is a trusted issuer.
- **Status Management**: Ability to activate or deactivate issuers.
- **Metadata Association**: Link additional metadata or details to each registered issuer.
- **Event Logging**: Comprehensive event tracking for all registry operations.

## Key Features

### Issuer Management
- **Add Issuer**: Register new trusted issuers with their associated metadata.
- **Update Issuer Status**: Activate, deactivate, or suspend issuer privileges.
- **Remove Issuer**: Deregister an issuer from the list.

### Trust Verification
- **`isTrustedIssuer()`**: Public function to check the trust status of an address.
- **Role-Based Trust**: Potentially support different levels or categories of trust.

### Metadata and Information
- **Issuer Details**: Store information like name, URL, and other relevant data about the issuer.
- **Attestation Tracking**: Keep a record of who registered or last updated an issuer.

## Interface Definition

```solidity
interface ITrustedIssuersRegistry {
    // Events
    event IssuerRegistered(
        address indexed issuerAddress,
        string indexed issuerName,
        address indexed registrator,
        string metadataURI
    );
    event IssuerStatusChanged(
        address indexed issuerAddress,
        bool newStatus,
        address indexed changer
    );
    event IssuerMetadataUpdated(
        address indexed issuerAddress,
        string oldMetadataURI,
        string newMetadataURI,
        address indexed updater
    );
    event IssuerRemoved(
        address indexed issuerAddress,
        address indexed remover
    );

    // Structs
    struct IssuerEntry {
        address issuerAddress;
        string name;
        string metadataURI;
        bool active;
        uint256 registeredAt;
        uint256 lastUpdated;
        address registrator;
    }

    // Core Functions
    function registerIssuer(
        address _issuerAddress,
        string calldata _name,
        string calldata _metadataURI
    ) external;

    function updateIssuerStatus(
        address _issuerAddress,
        bool _newStatus
    ) external;

    function updateIssuerMetadata(
        address _issuerAddress,
        string calldata _newMetadataURI
    ) external;

    function removeIssuer(address _issuerAddress) external;

    // View Functions
    function isTrustedIssuer(address _address) external view returns (bool);
    function getIssuerEntry(address _issuerAddress) external view returns (IssuerEntry memory);
    function getTrustedIssuerCount() external view returns (uint256);
    function getTrustedIssuerAddress(uint256 index) external view returns (address);
}
```

## Core Functions

### registerIssuer()
Registers a new address as a trusted issuer in the registry.

**Parameters:**
- `_issuerAddress`: The address of the entity to be registered as a trusted issuer.
- `_name`: A descriptive name for the issuer.
- `_metadataURI`: A URI pointing to additional off-chain metadata about the issuer.

**Access Control:**
- Typically restricted to the contract owner or an authorized administrator.

**Usage:**
```solidity
// Register a new trusted issuer
trustedIssuersRegistry.registerIssuer(
    0xAbc123...,  // Address of the issuer
    "Example Verification Service",
    "ipfs://example.com/issuer_meta.json"
);
```

### isTrustedIssuer()
Checks if a given address is currently registered as an active, trusted issuer.

**Parameters:**
- `_address`: The address to check.

**Returns:**
- `bool`: `true` if the address is a trusted and active issuer, `false` otherwise.

### updateIssuerStatus()
Changes the active status of a registered issuer to `true` (active) or `false` (inactive/suspended).

**Parameters:**
- `_issuerAddress`: The address of the issuer whose status is to be updated.
- `_newStatus`: The new status (`true` for active, `false` for inactive).

## Implementation Example

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract TrustedIssuersRegistry is ITrustedIssuersRegistry, Ownable {
    // Mapping from issuer address to its IssuerEntry details
    mapping(address => IssuerEntry) private _issuerEntries;
    // Array to maintain order and iterate over active issuers
    address[] private _activeIssuers;
    // Count of active issuers
    uint256 private _activeIssuerCount;

    constructor() {
        // Owner set to deployer by default due to Ownable
    }

    function registerIssuer(
        address _issuerAddress,
        string calldata _name,
        string calldata _metadataURI
    ) external override onlyOwner {
        require(_issuerAddress != address(0), "Issuer address cannot be zero");
        require(_issuerEntries[_issuerAddress].issuerAddress == address(0), "Issuer already registered");

        _issuerEntries[_issuerAddress] = IssuerEntry({
            issuerAddress: _issuerAddress,
            name: _name,
            metadataURI: _metadataURI,
            active: true,
            registeredAt: block.timestamp,
            lastUpdated: block.timestamp,
            registrator: msg.sender
        });
        _activeIssuers.push(_issuerAddress);
        _activeIssuerCount++;

        emit IssuerRegistered(_issuerAddress, _name, msg.sender, _metadataURI);
    }

    function updateIssuerStatus(
        address _issuerAddress,
        bool _newStatus
    ) external override onlyOwner {
        IssuerEntry storage entry = _issuerEntries[_issuerAddress];
        require(entry.issuerAddress != address(0), "Issuer not found");
        require(entry.active != _newStatus, "Status is already the same");

        entry.active = _newStatus;
        entry.lastUpdated = block.timestamp;
        
        if (_newStatus) {
            // Add to active_issuers if not already there
            bool found = false;
            for (uint i = 0; i < _activeIssuers.length; i++) {
                if (_activeIssuers[i] == _issuerAddress) {
                    found = true;
                    break;
                }
            }
            if (!found) {
                _activeIssuers.push(_issuerAddress);
                _activeIssuerCount++;
            }
        } else {
            // Remove from active_issuers
            for (uint256 i = 0; i < _activeIssuers.length; i++) {
                if (_activeIssuers[i] == _issuerAddress) {
                    _activeIssuers[i] = _activeIssuers[_activeIssuers.length - 1]; // Swap with last
                    _activeIssuers.pop(); // Pop last element
                    _activeIssuerCount--;
                    break;
                }
            }
        }
        emit IssuerStatusChanged(_issuerAddress, _newStatus, msg.sender);
    }

    function updateIssuerMetadata(
        address _issuerAddress,
        string calldata _newMetadataURI
    ) external override onlyOwner {
        IssuerEntry storage entry = _issuerEntries[_issuerAddress];
        require(entry.issuerAddress != address(0), "Issuer not found");

        string memory oldMetadataURI = entry.metadataURI;
        entry.metadataURI = _newMetadataURI;
        entry.lastUpdated = block.timestamp;

        emit IssuerMetadataUpdated(_issuerAddress, oldMetadataURI, _newMetadataURI, msg.sender);
    }

    function removeIssuer(address _issuerAddress) external override onlyOwner {
        IssuerEntry storage entry = _issuerEntries[_issuerAddress];
        require(entry.issuerAddress != address(0), "Issuer not found");

        // First, set status to inactive and remove from active list if it was active
        if (entry.active) {
            updateIssuerStatus(_issuerAddress, false); // This also handles removing from _activeIssuers
        }

        delete _issuerEntries[_issuerAddress]; // Clear from mapping
        emit IssuerRemoved(_issuerAddress, msg.sender);
    }

    function isTrustedIssuer(address _address) external view override returns (bool) {
        return _issuerEntries[_address].active;
    }

    function getIssuerEntry(address _issuerAddress) external view override returns (IssuerEntry memory) {
        return _issuerEntries[_issuerAddress];
    }

    function getTrustedIssuerCount() external view override returns (uint256) {
        return _activeIssuerCount;
    }

    function getTrustedIssuerAddress(uint256 index) external view override returns (address) {
        require(index < _activeIssuers.length, "Index out of bounds");
        return _activeIssuers[index];
    }
}
```

## Security Considerations

### Access Control
- **`onlyOwner`**: All functions that modify the registry's state (`registerIssuer`, `updateIssuerStatus`, `updateIssuerMetadata`, `removeIssuer`) are restricted to the contract owner. This ensures that only authorized entities can manage the list of trusted issuers.
- **Role-Based Access Control**: For multi-admin environments, `Ownable` can be replaced with a more granular RBAC system (e.g., OpenZeppelin's `AccessControl`).

### Data Integrity
- **Zero Address Check**: Prevent registration of `address(0)`.
- **Duplicate Registration**: Ensure an issuer address can only be registered once.
- **Status Consistency**: Logic in `updateIssuerStatus` and `removeIssuer` manages the `_activeIssuers` array to keep it consistent with the `active` flag in `IssuerEntry`.

### Gas Efficiency
- **Array Management**: Adding and removing from `_activeIssuers` array uses basic swap-and-pop, which is gas-efficient for array elements but doesn't maintain order. If order is critical or high performance is needed for very large lists, a more sophisticated data structure might be considered, though it adds complexity.

## Best Practices

### Metadata
1. **URI Standards**: Use IPFS or other decentralized storage for `metadataURI` to ensure immutability and availability of issuer details.
2. **Off-chain Information**: The on-chain registry should only store essential information (address, name, status, metadata URI); detailed information should reside off-chain.

### Events
1. **Comprehensive Events**: Emit events for all state-changing operations to enable off-chain indexing, auditing, and real-time monitoring of issuer changes.

### Flexibility
1. **Extensibility**: Design with future needs in mind. If different levels of trust or categories of issuers are required, extend the `IssuerEntry` struct and relevant functions.

## Integration Examples

### Frontend Integration (TypeScript via Ethers.js)

```typescript
import { ethers, Contract } from 'ethers';
import TrustedIssuersRegistryABI from './TrustedIssuersRegistry.json'; // ABI for ITrustedIssuersRegistry

const REGISTRY_ADDRESS = "0x..."; // Your deployed TrustedIssuersRegistry contract address

const getProvider = () => new ethers.providers.Web3Provider(window.ethereum);
const getSigner = () => getProvider().getSigner();
const getTrustedIssuersRegistryContract = () => new Contract(REGISTRY_ADDRESS, TrustedIssuersRegistryABI, getSigner());

async function registerNewIssuer(issuerAddress: string, name: string, metadataURI: string): Promise<void> {
    try {
        const registry = getTrustedIssuersRegistryContract();
        const tx = await registry.registerIssuer(issuerAddress, name, metadataURI);
        await tx.wait();
        alert("Issuer registered successfully!");
    } catch (error) {
        console.error("Error registering issuer:", error);
        alert("Failed to register issuer.");
    }
}

async function checkIssuerTrust(address: string): Promise<boolean> {
    try {
        const registry = getTrustedIssuersRegistryContract();
        const isTrusted = await registry.isTrustedIssuer(address);
        return isTrusted;
    } catch (error) {
        console.error("Error checking issuer trust:", error);
        throw error;
    }
}

async function updateIssuerStatus(issuerAddress: string, newStatus: boolean): Promise<void> {
    try {
        const registry = getTrustedIssuersRegistryContract();
        const tx = await registry.updateIssuerStatus(issuerAddress, newStatus);
        await tx.wait();
        alert("Issuer status updated successfully!");
    } catch (error) {
        console.error("Error updating issuer status:", error);
        alert("Failed to update issuer status.");
    }
}

// Example usage
// registerNewIssuer("0xSomeIssuerAddress", "Credential Service Inc.", "https://credentials.com/meta.json");
// const trusted = await checkIssuerTrust("0xSomeIssuerAddress"); // true or false
// updateIssuerStatus("0xSomeIssuerAddress", false); // Deactivate issuer
```

### Backend Integration (Node.js for a claim validation service)

```javascript
const Web3 = require('web3');
const TrustedIssuersRegistryABI = require('./TrustedIssuersRegistry.json').abi;

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const registryAddress = '0x...'; // Address of your deployed TrustedIssuersRegistry contract

const trustedIssuersRegistryContract = new web3.eth.Contract(TrustedIssuersRegistryABI, registryAddress);

async function validateClaimIssuer(claimIssuerAddress) {
    try {
        const isTrusted = await trustedIssuersRegistryContract.methods.isTrustedIssuer(claimIssuerAddress).call();
        if (isTrusted) {
            console.log(`Issuer ${claimIssuerAddress} is trusted.`);
            // Proceed with claim verification logic
            return true;
        } else {
            console.log(`Issuer ${claimIssuerAddress} is NOT trusted.`);
            // Reject claim or flag for manual review
            return false;
        }
    } catch (error) {
        console.error("Error validating claim issuer:", error);
        throw error;
    }
}

async function getIssuerDetails(issuerAddress) {
    try {
        const issuerEntry = await trustedIssuersRegistryContract.methods.getIssuerEntry(issuerAddress).call();
        console.log("Issuer Details:", issuerEntry);
        return issuerEntry;
    } catch (error) {
        console.error("Error fetching issuer details:", error);
        throw error;
    }
}

// Example usage in a backend service
// validateClaimIssuer("0xAnotherIssuerAddress");
// getIssuerDetails("0xSomeIssuerAddress");
```

## Related Documentation

- [IIdentity Interface](./iidentity.md)
- [ERC735 Claim Holder Interface](./ierc735.md)
- [Trusted Issuer Library](../libraries/trusted-issuer-lib.md)
- [Ownable Contract](https://docs.openzeppelin.com/contracts/5.x/api/access#Ownable)

## Standards Compliance

- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.