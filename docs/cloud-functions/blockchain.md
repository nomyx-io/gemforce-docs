# Blockchain Cloud Functions

## Overview

The [`blockchain.ts`](/Users/sschepis/Development/gem-base/src/cloud-functions/blockchain.ts) module provides cloud functions for managing blockchain network connections, provider configurations, and blockchain data access. These functions serve as the foundation for all blockchain interactions within the Gemforce platform.

## Functions

### Network Configuration Functions

#### `loadAllBlockchains`
```javascript
Parse.Cloud.run("loadAllBlockchains")
```

**Purpose**: Retrieves configuration data for all supported blockchain networks.

**Parameters**: None

**Returns**: Array of blockchain configuration objects
```javascript
[
  {
    providerUrl: "https://sepolia.infura.io/v3/your-key",
    networkId: 11155111
  },
  {
    providerUrl: "https://base-sepolia.infura.io/v3/your-key", 
    networkId: 84532
  }
  // ... more networks
]
```

**Usage Example**:
```javascript
const blockchains = await Parse.Cloud.run("loadAllBlockchains");
console.log("Available networks:", blockchains);

// Use in network selection UI
blockchains.forEach(blockchain => {
  console.log(`Network ${blockchain.networkId}: ${blockchain.providerUrl}`);
});
```

**Error Conditions**:
- Returns empty array if no blockchain configurations exist
- Database connection errors

---

#### `loadProviderUrl`
```javascript
Parse.Cloud.run("loadProviderUrl", { networkId: 1337 })
```

**Purpose**: Retrieves the RPC endpoint URL for a specific blockchain network.

**Parameters**:
- `networkId` (number): The network ID of the blockchain

**Returns**: String - RPC provider URL

**Usage Example**:
```javascript
try {
  const providerUrl = await Parse.Cloud.run("loadProviderUrl", {
    networkId: 11155111 // Sepolia
  });
  console.log("Sepolia RPC URL:", providerUrl);
  
  // Use with ethers.js
  const provider = new ethers.providers.JsonRpcProvider(providerUrl);
} catch (error) {
  console.error("Network not found:", error.message);
}
```

**Supported Network IDs**:
- `1337` - Localhost (development)
- `11155111` - Sepolia Ethereum testnet
- `84532` - Base Sepolia testnet
- `11155420` - OP Sepolia testnet

**Error Conditions**:
- `"No blockchain found for networkId: {networkId}"` - Invalid or unsupported network ID

---

#### `loadProviderWebSocketUrl`
```javascript
Parse.Cloud.run("loadProviderWebSocketUrl", { networkId: 1337 })
```

**Purpose**: Retrieves the WebSocket endpoint URL for real-time blockchain event monitoring.

**Parameters**:
- `networkId` (number): The network ID of the blockchain

**Returns**: String - WebSocket provider URL

**Usage Example**:
```javascript
const wsUrl = await Parse.Cloud.run("loadProviderWebSocketUrl", {
  networkId: 11155111
});

// Use for event listening
const wsProvider = new ethers.providers.WebSocketProvider(wsUrl);
wsProvider.on("block", (blockNumber) => {
  console.log("New block:", blockNumber);
});
```

**Error Conditions**:
- `"No blockchain found for networkId: {networkId}"` - Invalid network ID
- WebSocket URL may be null for some networks

---

#### `loadAllProviderUrls`
```javascript
Parse.Cloud.run("loadAllProviderUrls")
```

**Purpose**: Retrieves WebSocket URLs for all configured blockchain networks.

**Parameters**: None

**Returns**: Array of WebSocket URLs

**Usage Example**:
```javascript
const wsUrls = await Parse.Cloud.run("loadAllProviderUrls");

// Set up multi-network monitoring
wsUrls.forEach((url, index) => {
  if (url) {
    const provider = new ethers.providers.WebSocketProvider(url);
    provider.on("block", (blockNumber) => {
      console.log(`Network ${index} - New block: ${blockNumber}`);
    });
  }
});
```

**Error Conditions**:
- `"No blockchain found"` - No blockchain configurations exist
- Array may contain null values for networks without WebSocket support

---

### Blockchain Data Access

#### `loadBlockchainDataForNetwork`
```javascript
Parse.Cloud.run("loadBlockchainDataForNetwork", { networkId: 1337 })
```

**Purpose**: Creates a complete blockchain connection setup including provider, wallet, and signer for a specific network.

**Parameters**:
- `networkId` (number): The network ID of the blockchain

**Returns**: Object containing blockchain connection components
```javascript
{
  signer: ethers.Signer,    // Connected signer for transactions
  provider: ethers.Provider, // Network provider for queries
  wallet: ethers.Wallet     // Wallet instance
}
```

**Usage Example**:
```javascript
// Get blockchain connection for Sepolia
const { signer, provider, wallet } = await Parse.Cloud.run(
  "loadBlockchainDataForNetwork", 
  { networkId: 11155111 }
);

// Check wallet balance
const balance = await provider.getBalance(wallet.address);
console.log("Wallet balance:", ethers.utils.formatEther(balance));

// Send transaction
const tx = await signer.sendTransaction({
  to: "0x...",
  value: ethers.utils.parseEther("0.1")
});
console.log("Transaction hash:", tx.hash);
```

**Security Considerations**:
- Uses server-side private key management
- Private key is retrieved securely via `getPrivateKey()`
- Signer is created server-side to prevent key exposure

**Error Conditions**:
- `"No blockchain found for networkId: {networkId}"` - Invalid network ID
- Private key retrieval errors
- Network connection failures

---

## Integration Patterns

### Multi-Network Operations
```javascript
// Load all networks and perform operations
const blockchains = await Parse.Cloud.run("loadAllBlockchains");

for (const blockchain of blockchains) {
  try {
    const { provider } = await Parse.Cloud.run("loadBlockchainDataForNetwork", {
      networkId: blockchain.networkId
    });
    
    const blockNumber = await provider.getBlockNumber();
    console.log(`Network ${blockchain.networkId} - Block: ${blockNumber}`);
  } catch (error) {
    console.error(`Network ${blockchain.networkId} error:`, error.message);
  }
}
```

### Event Monitoring Setup
```javascript
// Set up cross-network event monitoring
async function setupEventMonitoring() {
  const wsUrls = await Parse.Cloud.run("loadAllProviderUrls");
  
  wsUrls.forEach((url, index) => {
    if (url) {
      const provider = new ethers.providers.WebSocketProvider(url);
      
      // Monitor new blocks
      provider.on("block", (blockNumber) => {
        console.log(`Network ${index} - Block ${blockNumber}`);
      });
      
      // Monitor specific contract events
      const contract = new ethers.Contract(contractAddress, abi, provider);
      contract.on("Transfer", (from, to, value) => {
        console.log(`Transfer on network ${index}:`, { from, to, value });
      });
    }
  });
}
```

### Network-Specific Contract Interactions
```javascript
// Deploy contract to specific network
async function deployToNetwork(networkId, contractBytecode, constructorArgs) {
  const { signer } = await Parse.Cloud.run("loadBlockchainDataForNetwork", {
    networkId
  });
  
  const factory = new ethers.ContractFactory(abi, contractBytecode, signer);
  const contract = await factory.deploy(...constructorArgs);
  await contract.deployed();
  
  return contract.address;
}
```

## Database Schema

The blockchain functions rely on the `Blockchain` Parse class with the following schema:

```javascript
// Blockchain class fields
{
  networkId: Number,        // Unique network identifier
  name: String,            // Human-readable network name
  symbol: String,          // Network symbol (e.g., "ETH", "sETH")
  rpcEndpoint: String,     // HTTP RPC endpoint URL
  wssEndpoint: String,     // WebSocket endpoint URL (optional)
  code: String,            // Short network code (e.g., "sepolia")
  explorer: String         // Block explorer URL
}
```

### Sample Data
```javascript
// Sepolia network configuration
{
  networkId: 11155111,
  name: "Sepolia Ethereum",
  symbol: "sETH",
  rpcEndpoint: "https://sepolia.infura.io/v3/your-key",
  wssEndpoint: "wss://sepolia.infura.io/ws/v3/your-key",
  code: "sepolia",
  explorer: "https://sepolia.etherscan.io"
}
```

## Error Handling

### Common Error Patterns
```javascript
// Robust error handling for blockchain functions
async function safeBlockchainCall(networkId) {
  try {
    const result = await Parse.Cloud.run("loadBlockchainDataForNetwork", {
      networkId
    });
    return result;
  } catch (error) {
    if (error.message.includes("No blockchain found")) {
      console.error("Unsupported network:", networkId);
      // Handle unsupported network
    } else if (error.message.includes("connection")) {
      console.error("Network connection failed:", error.message);
      // Handle connection issues
    } else {
      console.error("Unexpected error:", error.message);
      // Handle other errors
    }
    throw error;
  }
}
```

## Performance Considerations

### Caching Strategies
```javascript
// Client-side caching of network configurations
let cachedBlockchains = null;
let cacheTimestamp = 0;
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

async function getCachedBlockchains() {
  const now = Date.now();
  if (!cachedBlockchains || (now - cacheTimestamp) > CACHE_DURATION) {
    cachedBlockchains = await Parse.Cloud.run("loadAllBlockchains");
    cacheTimestamp = now;
  }
  return cachedBlockchains;
}
```

### Connection Pooling
- Provider instances should be reused when possible
- WebSocket connections should be managed carefully to avoid memory leaks
- Implement connection retry logic for production use

## Security Best Practices

### Private Key Management
- Private keys are stored securely server-side
- Never expose private keys to client applications
- Use environment variables for sensitive configuration

### Network Validation
- Always validate network IDs before processing
- Implement rate limiting for blockchain function calls
- Monitor for unusual usage patterns

## Testing

### Unit Tests
```javascript
describe("Blockchain Functions", () => {
  test("loadAllBlockchains returns array", async () => {
    const result = await Parse.Cloud.run("loadAllBlockchains");
    expect(Array.isArray(result)).toBe(true);
  });
  
  test("loadProviderUrl returns valid URL", async () => {
    const url = await Parse.Cloud.run("loadProviderUrl", {
      networkId: 1337
    });
    expect(url).toMatch(/^https?:\/\//);
  });
});
```

### Integration Tests
- Test with real network connections
- Validate provider functionality
- Test error conditions with invalid network IDs

## Related Documentation

- [Contract Functions](contracts.md) - Smart contract interactions
- [DFNS Functions](dfns.md) - Wallet management
- [Deploy Functions](deploy.md) - Contract deployment
- [System Architecture](../system-architecture/gemforce-system-architecture.md) - Overall system design

---

*These functions provide the foundation for all blockchain interactions in the Gemforce platform.*