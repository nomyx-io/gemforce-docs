# DFNS Cloud Functions

## Overview

The [`dfns.ts`](/Users/sschepis/Development/gem-base/src/cloud-functions/dfns.ts) module provides comprehensive integration with DFNS (Decentralized Finance Network Services) Wallet-as-a-Service platform. These cloud functions enable secure wallet management, user authentication, transaction signing, and smart contract interactions through DFNS's infrastructure.

## DFNS Integration Architecture

### Key Components
- **DfnsApiClient**: Administrative client for system-level operations
- **DfnsDelegatedApiClient**: User-delegated client for end-user operations
- **AsymmetricKeySigner**: Cryptographic signing for API authentication
- **Multi-Network Support**: BaseSepolia, OptimismSepolia, and local networks

### Security Model
- Server-side private key management
- Delegated authentication for end users
- Challenge-response authentication flow
- Secure transaction signing with user approval

## Configuration

### Environment Variables
```bash
# DFNS API Configuration
DFNS_APP_ID=your_dfns_app_id
DFNS_AUTH_TOKEN=your_dfns_auth_token
DFNS_API_URL=https://api.dfns.ninja
DFNS_CRED_ID=your_credential_id

# Network Configuration
CHAIN_ID=basesep  # or optsep, localhost
ETH_NODE_URI_BASESEP=https://base-sepolia.infura.io/v3/your-key
ETH_NODE_URI_OPTSEP=https://optimism-sepolia.infura.io/v3/your-key

# Private Key File
# dfns_private.key file in project root
```

### Network Mapping
```javascript
const DFNS_NETWORK = (() => {
  if (chainCode === "basesep") return "BaseSepolia";
  if (chainCode === "optsep") return "OptimismSepolia";
  return "local";
})();
```

## Core Functions

### User Registration

#### `registerInit`
```javascript
Parse.Cloud.run("registerInit", { username: "user@example.com" })
```

**Purpose**: Initiates the user registration process with DFNS.

**Parameters**:
- `username` (string): User's email address for registration

**Returns**: Registration challenge object
```javascript
{
  challenge: "base64-encoded-challenge",
  temporaryAuthenticationToken: "temp-token",
  // ... other challenge data
}
```

**Process**:
1. Validates username parameter
2. Creates DFNS API client with system credentials
3. Generates delegated registration challenge
4. Returns challenge for client-side signing

**Usage Example**:
```javascript
// Client-side registration initiation
const challenge = await Parse.Cloud.run("registerInit", {
  username: "user@example.com"
});

// Client signs the challenge using DFNS SDK
const signedChallenge = await dfnsClient.signChallenge(challenge);
```

**Error Conditions**:
- `VALIDATION_ERROR`: Missing username parameter
- `INTERNAL_SERVER_ERROR`: DFNS API communication failure

---

#### `registerComplete`
```javascript
Parse.Cloud.run("registerComplete", {
  signedChallenge: signedChallengeObject,
  temporaryAuthenticationToken: "temp-token"
})
```

**Purpose**: Completes user registration and creates DFNS wallet.

**Parameters**:
- `signedChallenge` (object): Client-signed challenge from registerInit
- `temporaryAuthenticationToken` (string): Temporary token from registerInit

**Returns**: Registration result with user token and wallet information
```javascript
{
  token: "user-auth-token",
  user: {
    id: "user-id",
    email: "user@example.com"
  },
  wallets: [
    {
      id: "wallet-id",
      network: "BaseSepolia",
      address: "0x..."
    }
  ]
}
```

**Process**:
1. Validates signed challenge and temporary token
2. Creates DFNS client with temporary authentication
3. Registers end user with wallet creation
4. Returns user authentication token and wallet details

### User Authentication

#### `login`
```javascript
Parse.Cloud.run("login", { username: "user@example.com" })
```

**Purpose**: Authenticates existing DFNS user and returns access token.

**Parameters**:
- `username` (string): User's registered email address

**Returns**: Login result with authentication token
```javascript
{
  username: "user@example.com",
  token: "user-auth-token"
}
```

**Process**:
1. Validates username parameter
2. Performs delegated login with DFNS
3. Returns authentication token for subsequent operations

**Usage Example**:
```javascript
// User login
const loginResult = await Parse.Cloud.run("login", {
  username: "user@example.com"
});

// Store token for subsequent API calls
const authToken = loginResult.token;
```

### Wallet Management

#### `listWallets`
```javascript
Parse.Cloud.run("listWallets", { authToken: "user-auth-token" })
```

**Purpose**: Retrieves all wallets associated with the authenticated user.

**Parameters**:
- `authToken` (string): User's authentication token from login

**Returns**: Array of user's wallets
```javascript
{
  items: [
    {
      id: "wallet-id-1",
      network: "BaseSepolia",
      address: "0x1234...",
      status: "Active",
      dateCreated: "2024-01-01T00:00:00Z"
    },
    {
      id: "wallet-id-2", 
      network: "OptimismSepolia",
      address: "0x5678...",
      status: "Active",
      dateCreated: "2024-01-01T00:00:00Z"
    }
  ]
}
```

**Usage Example**:
```javascript
// List user's wallets
const wallets = await Parse.Cloud.run("listWallets", {
  authToken: userAuthToken
});

// Find wallet for specific network
const baseWallet = wallets.items.find(w => w.network === "BaseSepolia");
```

### Transaction Signing

#### `signaturesInit`
```javascript
Parse.Cloud.run("signaturesInit", {
  authToken: "user-auth-token",
  walletId: "wallet-id",
  message: "Hello, World!"
})
```

**Purpose**: Initiates message signing process for the specified wallet.

**Parameters**:
- `authToken` (string): User's authentication token
- `walletId` (string): ID of the wallet to use for signing
- `message` (string): Message to be signed

**Returns**: Signature initialization data
```javascript
{
  requestBody: {
    kind: "Message",
    message: "48656c6c6f2c20576f726c6421" // hex-encoded message
  },
  challenge: {
    challenge: "base64-challenge",
    // ... other challenge data
  }
}
```

**Process**:
1. Validates authentication token, wallet ID, and message
2. Creates delegated DFNS client
3. Converts message to hex format
4. Generates signature initialization challenge
5. Returns challenge for client-side signing

---

#### `signaturesComplete`
```javascript
Parse.Cloud.run("signaturesComplete", {
  authToken: "user-auth-token",
  walletId: "wallet-id",
  requestBody: requestBodyFromInit,
  signedChallenge: clientSignedChallenge
})
```

**Purpose**: Completes the message signing process with user-signed challenge.

**Parameters**:
- `authToken` (string): User's authentication token
- `walletId` (string): Wallet ID used for signing
- `requestBody` (object): Request body from signaturesInit
- `signedChallenge` (object): Client-signed challenge

**Returns**: Completed signature
```javascript
{
  signature: "0x1234567890abcdef...",
  recoveryId: 0,
  // ... other signature data
}
```

**Usage Example**:
```javascript
// Complete signing workflow
const initResult = await Parse.Cloud.run("signaturesInit", {
  authToken: userToken,
  walletId: walletId,
  message: "Transaction approval"
});

// Client signs the challenge
const signedChallenge = await dfnsClient.signChallenge(initResult.challenge);

// Complete the signature
const signature = await Parse.Cloud.run("signaturesComplete", {
  authToken: userToken,
  walletId: walletId,
  requestBody: initResult.requestBody,
  signedChallenge: signedChallenge
});
```

### Smart Contract Interactions

#### `dfnsInitiatePurchase`
```javascript
Parse.Cloud.run("dfnsInitiatePurchase", {
  tokenId: 123,
  walletId: "wallet-id",
  dfns_token: "user-auth-token"
})
```

**Purpose**: Initiates an NFT purchase transaction through DFNS wallet.

**Parameters**:
- `tokenId` (number): ID of the NFT to purchase
- `walletId` (string): DFNS wallet ID for the transaction
- `dfns_token` (string): User's DFNS authentication token

**Returns**: Transaction initialization data for marketplace purchase

**Process**:
1. Validates all required parameters
2. Creates delegated DFNS client
3. Encodes marketplace purchase transaction data
4. Prepares EVM transaction body
5. Returns transaction data for user approval

**Integration**: Works with MarketplaceFacet for NFT purchases

## Client Integration Patterns

### Complete User Onboarding Flow
```javascript
// 1. Register new user
const registrationChallenge = await Parse.Cloud.run("registerInit", {
  username: "user@example.com"
});

// 2. Client signs challenge (using DFNS SDK)
const signedChallenge = await dfnsClient.signChallenge(registrationChallenge);

// 3. Complete registration
const registrationResult = await Parse.Cloud.run("registerComplete", {
  signedChallenge: signedChallenge,
  temporaryAuthenticationToken: registrationChallenge.temporaryAuthenticationToken
});

// 4. Store user token
const userToken = registrationResult.token;
```

### Wallet Operations Flow
```javascript
// 1. Login existing user
const loginResult = await Parse.Cloud.run("login", {
  username: "user@example.com"
});

// 2. List user's wallets
const wallets = await Parse.Cloud.run("listWallets", {
  authToken: loginResult.token
});

// 3. Select wallet for operations
const selectedWallet = wallets.items[0];
```

### Transaction Signing Flow
```javascript
// 1. Initiate signature
const signatureInit = await Parse.Cloud.run("signaturesInit", {
  authToken: userToken,
  walletId: selectedWallet.id,
  message: "Approve transaction"
});

// 2. Client signs challenge
const signedChallenge = await dfnsClient.signChallenge(signatureInit.challenge);

// 3. Complete signature
const signature = await Parse.Cloud.run("signaturesComplete", {
  authToken: userToken,
  walletId: selectedWallet.id,
  requestBody: signatureInit.requestBody,
  signedChallenge: signedChallenge
});
```

## Contract Integration

### Supported Contracts
The DFNS module automatically loads ABIs and addresses for:

- **Treasury Contract**: Financial operations
- **Marketplace Contract**: NFT trading
- **USDC Contract**: Stable coin payments
- **Carbon Credits Contract**: Environmental assets
- **Identity Contracts**: User verification
- **Trade Deal Contracts**: Financial instruments

### Dynamic Contract Loading
```javascript
// Contracts are loaded based on network configuration
const networkConfig = config.find(chain => chain.code === chainCode);
const artifactsPath = `../../deployments/${chainCode}`;

// ABIs and addresses loaded from deployment artifacts
if (fs.existsSync(marketplaceFilePath)) {
  const { address, abi } = JSON.parse(fs.readFileSync(marketplaceFilePath, "utf8"));
  MARKETPLACE_CONTRACT_ADDRESS = address;
  MARKETPLACE_CONTRACT_ABI = abi;
}
```

## Security Considerations

### Authentication Security
- Server-side private key storage for DFNS API authentication
- Delegated authentication prevents direct key exposure
- Challenge-response flow ensures user consent

### Transaction Security
- User approval required for all transactions
- Message signing for transaction authorization
- Network-specific wallet isolation

### API Security
- Input validation for all parameters
- Error handling prevents information leakage
- Rate limiting through Parse Server

## Error Handling

### Common Error Patterns
```javascript
// Validation errors
if (!username) {
  throw new Parse.Error(
    Parse.Error.VALIDATION_ERROR,
    "Username is required"
  );
}

// DFNS API errors
try {
  const result = await client.auth.delegatedLogin({ body: { username } });
  return result;
} catch (error) {
  throw new Parse.Error(
    Parse.Error.INTERNAL_SERVER_ERROR,
    `Error during login: ${error.message}`
  );
}
```

### Error Types
- **VALIDATION_ERROR**: Missing or invalid parameters
- **INTERNAL_SERVER_ERROR**: DFNS API communication failures
- **AUTHENTICATION_ERROR**: Invalid tokens or credentials

## Performance Considerations

### Client Caching
- Cache user authentication tokens
- Reuse wallet information
- Minimize API calls through batching

### Network Optimization
- Use appropriate network for operations
- Consider gas costs for different networks
- Implement retry logic for network failures

## Testing

### Unit Tests
```javascript
describe("DFNS Functions", () => {
  test("registerInit requires username", async () => {
    await expect(
      Parse.Cloud.run("registerInit", {})
    ).rejects.toThrow("Username is required");
  });
  
  test("login returns token", async () => {
    const result = await Parse.Cloud.run("login", {
      username: "test@example.com"
    });
    expect(result).toHaveProperty("token");
  });
});
```

### Integration Tests
- Test with DFNS sandbox environment
- Validate wallet creation and management
- Test transaction signing flows
- Verify contract interaction patterns

## Related Documentation

- [DFNS API Documentation](https://docs.dfns.co/)
- [Blockchain Functions](blockchain.md) - Network management
- [Contract Functions](contracts.md) - Smart contract interactions
- [Authentication Functions](auth-functions.md) - User authentication
- [Developer Setup Guide](../developer-setup-guide.md) - Environment configuration

---

*These functions provide secure wallet-as-a-service integration for the Gemforce platform through DFNS infrastructure.*