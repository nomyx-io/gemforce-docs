# Contract Cloud Functions

## Overview

The [`contracts.ts`](/Users/sschepis/Development/gem-base/src/cloud-functions/contracts.ts) module provides Parse Server cloud functions for interacting with smart contracts on various blockchain networks. These functions enable secure contract operations, diamond facet management, and flexible contract interaction patterns through a unified API.

## Module Details

- **File**: `src/cloud-functions/contracts.ts`
- **Framework**: Parse Server Cloud Functions
- **Language**: TypeScript
- **Dependencies**: Contract utilities, Diamond utilities

## Key Features

### 🔹 Diamond Contract Management
- Add facets to diamond contracts dynamically
- Support for EIP-2535 Diamond Standard operations
- Network-agnostic diamond management

### 🔹 Generic Contract Interaction
- Call state-changing contract methods
- Query read-only contract methods
- Support for custom ABIs and contract addresses

### 🔹 Smart Contract Loading
- Load contract configurations for specific networks
- Batch loading of multiple contracts
- Network-specific contract resolution

### 🔹 Flexible Parameter Handling
- Support for custom private keys
- Optional parameter validation
- Value transfer support for payable functions

## Cloud Functions

### Diamond Management Functions

#### `addDiamondFacet`
```typescript
Parse.Cloud.define("addDiamondFacet", async (request: any) => {
  const { networkId, diamondAddress, privateKey, facetName } = request.params;
  // Implementation details...
});
```

**Purpose**: Adds a new facet to an existing diamond contract.

**Parameters**:
- `networkId` (string): Target blockchain network identifier
- `diamondAddress` (string): Address of the diamond contract
- `privateKey` (string, optional): Private key for transaction signing
- `facetName` (string): Name of the facet to add

**Returns**: Transaction result with facet addition details

**Access Control**: Requires valid private key with diamond owner permissions

**Example Usage**:
```javascript
// Add marketplace facet to diamond
const result = await Parse.Cloud.run("addDiamondFacet", {
  networkId: "baseSepolia",
  diamondAddress: "0x1234567890123456789012345678901234567890",
  facetName: "MarketplaceFacet"
});

console.log("Facet added:", result.transactionHash);
```

**Process**:
1. Validates network and diamond address
2. Resolves private key (uses default if not provided)
3. Loads facet contract for the specified network
4. Executes diamond cut operation to add facet
5. Returns transaction details

**Error Conditions**:
- Invalid network ID
- Diamond contract not found
- Insufficient permissions
- Facet already exists
- Transaction failure

---

### Generic Contract Functions

#### `callMethod`
```typescript
Parse.Cloud.define("callMethod", async (request: any) => {
  const { methodName, params, privateKey } = request.params;
  // Implementation details...
});
```

**Purpose**: Calls a state-changing method on a pre-configured contract.

**Parameters**:
- `methodName` (string): Name of the contract method to call
- `params` (array): Array of parameters for the method
- `privateKey` (string, optional): Private key for transaction signing

**Returns**: Transaction result with method execution details

**Example Usage**:
```javascript
// Call mint function on NFT contract
const result = await Parse.Cloud.run("callMethod", {
  methodName: "mint",
  params: ["0xRecipientAddress", "tokenURI"],
  privateKey: "0x..." // Optional
});

console.log("Mint transaction:", result.transactionHash);
```

**Use Cases**:
- Token minting operations
- Marketplace listing creation
- Identity claim management
- Trade deal operations

---

#### `viewMethod`
```typescript
Parse.Cloud.define("viewMethod", async (request: any) => {
  const { methodName, params } = request.params;
  // Implementation details...
});
```

**Purpose**: Calls a read-only method on a pre-configured contract.

**Parameters**:
- `methodName` (string): Name of the view method to call
- `params` (array): Array of parameters for the method

**Returns**: Method return value(s)

**Example Usage**:
```javascript
// Get token balance
const balance = await Parse.Cloud.run("viewMethod", {
  methodName: "balanceOf",
  params: ["0xUserAddress"]
});

console.log("Token balance:", balance);
```

**Use Cases**:
- Balance queries
- Token metadata retrieval
- Contract state inspection
- Verification checks

---

### Custom Contract Functions

#### `callContractMethod`
```typescript
Parse.Cloud.define("callContractMethod", async (request: any) => {
  const { providerUrl, privateKey, abi, address, functionName, params, value } = request.params;
  // Implementation details...
});
```

**Purpose**: Calls a state-changing method on any contract with custom ABI.

**Parameters**:
- `providerUrl` (string): Blockchain RPC endpoint URL
- `privateKey` (string): Private key for transaction signing
- `abi` (array): Contract ABI definition
- `address` (string): Contract address
- `functionName` (string): Name of the function to call
- `params` (array): Function parameters
- `value` (string, optional): ETH value to send with transaction

**Returns**: Transaction result with execution details

**Example Usage**:
```javascript
// Call custom contract function
const result = await Parse.Cloud.run("callContractMethod", {
  providerUrl: "https://sepolia.base.org",
  privateKey: "0x...",
  abi: contractABI,
  address: "0xContractAddress",
  functionName: "customFunction",
  params: ["param1", "param2"],
  value: "1000000000000000000" // 1 ETH in wei
});
```

**Use Cases**:
- Integration with external contracts
- Custom protocol interactions
- Multi-network operations
- Advanced contract testing

---

#### `viewContractMethod`
```typescript
Parse.Cloud.define("viewContractMethod", async (request: any) => {
  const { providerUrl, abi, address, functionName, params } = request.params;
  // Implementation details...
});
```

**Purpose**: Calls a read-only method on any contract with custom ABI.

**Parameters**:
- `providerUrl` (string): Blockchain RPC endpoint URL
- `abi` (array): Contract ABI definition
- `address` (string): Contract address
- `functionName` (string): Name of the view function to call
- `params` (array): Function parameters

**Returns**: Function return value(s)

**Example Usage**:
```javascript
// Query custom contract state
const result = await Parse.Cloud.run("viewContractMethod", {
  providerUrl: "https://sepolia.base.org",
  abi: contractABI,
  address: "0xContractAddress",
  functionName: "getCustomData",
  params: ["queryParam"]
});

console.log("Custom data:", result);
```

**Use Cases**:
- External contract queries
- Cross-protocol data retrieval
- Multi-network state inspection
- Integration testing

---

### Contract Loading Functions

#### `loadSmartContractForNetwork`
```typescript
Parse.Cloud.define("loadSmartContractForNetwork", async (request: any) => {
  const { smartContractName, networkId } = request.params;
  // Implementation details...
});
```

**Purpose**: Loads contract configuration for a specific contract on a specific network.

**Parameters**:
- `smartContractName` (string): Name of the smart contract
- `networkId` (string): Target network identifier

**Returns**: Contract configuration including address, ABI, and metadata

**Example Usage**:
```javascript
// Load diamond contract for Base Sepolia
const contract = await Parse.Cloud.run("loadSmartContractForNetwork", {
  smartContractName: "Diamond",
  networkId: "baseSepolia"
});

console.log("Diamond address:", contract.address);
console.log("Diamond ABI:", contract.abi);
```

**Use Cases**:
- Dynamic contract loading
- Network-specific configurations
- Contract address resolution
- ABI retrieval for frontend

---

#### `loadSmartContractsForNetwork`
```typescript
Parse.Cloud.define("loadSmartContractsForNetwork", async (request: any) => {
  const { networkId } = request.params;
  // Implementation details...
});
```

**Purpose**: Loads all available contract configurations for a specific network.

**Parameters**:
- `networkId` (string): Target network identifier

**Returns**: Object containing all contract configurations for the network

**Example Usage**:
```javascript
// Load all contracts for Base Sepolia
const contracts = await Parse.Cloud.run("loadSmartContractsForNetwork", {
  networkId: "baseSepolia"
});

console.log("Available contracts:", Object.keys(contracts));
console.log("Diamond address:", contracts.Diamond.address);
console.log("USDC address:", contracts.USDC.address);
```

**Use Cases**:
- Bulk contract loading
- Network initialization
- Frontend contract registry
- Development environment setup

## Integration Examples

### Diamond Facet Management
```javascript
// Complete diamond facet management workflow
class DiamondManager {
  async addFacetToDiamond(networkId, diamondAddress, facetName) {
    try {
      // Add the facet
      const result = await Parse.Cloud.run("addDiamondFacet", {
        networkId,
        diamondAddress,
        facetName
      });
      
      console.log(`${facetName} added to diamond:`, result.transactionHash);
      
      // Verify facet was added
      const diamond = await Parse.Cloud.run("loadSmartContractForNetwork", {
        smartContractName: "Diamond",
        networkId
      });
      
      // Check if facet functions are available
      const facetFunctions = await Parse.Cloud.run("viewContractMethod", {
        providerUrl: this.getProviderUrl(networkId),
        abi: diamond.abi,
        address: diamondAddress,
        functionName: "facetFunctionSelectors",
        params: [facetName]
      });
      
      console.log(`${facetName} functions:`, facetFunctions);
      return result;
      
    } catch (error) {
      console.error("Failed to add facet:", error);
      throw error;
    }
  }
}
```

### Multi-Network Contract Interaction
```javascript
// Interact with contracts across multiple networks
class MultiNetworkContractManager {
  async deployToAllNetworks(contractName, constructorParams) {
    const networks = ["baseSepolia", "optimismSepolia", "sepolia"];
    const deployments = {};
    
    for (const networkId of networks) {
      try {
        // Load network-specific configuration
        const contracts = await Parse.Cloud.run("loadSmartContractsForNetwork", {
          networkId
        });
        
        // Deploy contract
        const result = await Parse.Cloud.run("callContractMethod", {
          providerUrl: this.getProviderUrl(networkId),
          privateKey: this.getPrivateKey(networkId),
          abi: contracts[contractName].abi,
          address: contracts.ContractFactory.address,
          functionName: "deploy",
          params: constructorParams
        });
        
        deployments[networkId] = result;
        console.log(`Deployed to ${networkId}:`, result.contractAddress);
        
      } catch (error) {
        console.error(`Failed to deploy to ${networkId}:`, error);
        deployments[networkId] = { error: error.message };
      }
    }
    
    return deployments;
  }
}
```

### Custom Contract Integration
```javascript
// Integrate with external protocols
class ExternalProtocolIntegration {
  async interactWithUniswap(tokenA, tokenB, amount) {
    const uniswapABI = [
      // Uniswap V3 Router ABI
      "function exactInputSingle((address,address,uint24,address,uint256,uint256,uint256,uint160)) external payable returns (uint256)"
    ];
    
    const swapParams = {
      tokenIn: tokenA,
      tokenOut: tokenB,
      fee: 3000, // 0.3%
      recipient: this.userAddress,
      deadline: Math.floor(Date.now() / 1000) + 3600,
      amountIn: amount,
      amountOutMinimum: 0,
      sqrtPriceLimitX96: 0
    };
    
    const result = await Parse.Cloud.run("callContractMethod", {
      providerUrl: "https://mainnet.base.org",
      privateKey: this.privateKey,
      abi: uniswapABI,
      address: "0xUniswapV3RouterAddress",
      functionName: "exactInputSingle",
      params: [swapParams]
    });
    
    return result;
  }
}
```

### Contract State Monitoring
```javascript
// Monitor contract state changes
class ContractMonitor {
  async monitorDiamondState(networkId, diamondAddress) {
    const contracts = await Parse.Cloud.run("loadSmartContractsForNetwork", {
      networkId
    });
    
    const diamond = contracts.Diamond;
    
    // Get current facets
    const facets = await Parse.Cloud.run("viewContractMethod", {
      providerUrl: this.getProviderUrl(networkId),
      abi: diamond.abi,
      address: diamondAddress,
      functionName: "facets",
      params: []
    });
    
    console.log("Current facets:", facets);
    
    // Monitor for changes
    setInterval(async () => {
      const currentFacets = await Parse.Cloud.run("viewContractMethod", {
        providerUrl: this.getProviderUrl(networkId),
        abi: diamond.abi,
        address: diamondAddress,
        functionName: "facets",
        params: []
      });
      
      if (JSON.stringify(currentFacets) !== JSON.stringify(facets)) {
        console.log("Diamond facets changed:", currentFacets);
        this.handleFacetChange(currentFacets);
      }
    }, 30000); // Check every 30 seconds
  }
}
```

## Security Considerations

### Private Key Management
- Private keys are handled securely through utility functions
- Default private key fallback for authorized operations
- No private key logging or exposure in responses

### Access Control
- Functions validate network permissions
- Contract owner verification for sensitive operations
- Transaction signing validation

### Input Validation
- Parameter validation for all contract interactions
- ABI validation for custom contract calls
- Network ID validation against supported networks

## Error Handling

### Network Errors
- Invalid network ID validation
- RPC endpoint connectivity checks
- Transaction timeout handling

### Contract Errors
- Contract existence validation
- Function signature verification
- Parameter type checking

### Transaction Errors
- Gas estimation and limits
- Nonce management
- Transaction failure recovery

## Performance Optimization

### Caching
- Contract ABI caching for repeated calls
- Network configuration caching
- Provider connection pooling

### Batch Operations
- Multiple contract calls in single request
- Efficient parameter encoding
- Optimized gas usage

## Testing Considerations

### Unit Tests
- Individual function testing
- Parameter validation testing
- Error condition handling

### Integration Tests
- Multi-network deployment testing
- Diamond facet management workflows
- External contract integration

### Load Testing
- Concurrent request handling
- Network congestion scenarios
- Rate limiting validation

## Related Documentation

- [Blockchain Cloud Functions](./blockchain.md) - Network provider management
- [DFNS Cloud Functions](./dfns.md) - Wallet-as-a-Service integration
- [Contract Utilities](../libraries/contract-utilities.md) - Underlying contract utilities
- [Diamond Utilities](../libraries/diamond-utilities.md) - Diamond management utilities
- [Developer Setup Guide](../developer-setup-guide.md) - Environment configuration

---

*These cloud functions provide the backbone for secure, scalable smart contract interactions within the Gemforce platform, supporting both standard operations and custom integrations across multiple blockchain networks.*