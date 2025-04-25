# Gemforce API Quick Reference

This document provides a concise reference for developers who need to interact with the Gemforce API, including smart contract calls and cloud function endpoints.

## Smart Contract Interactions

### Diamond Pattern

The Diamond pattern allows for modular and upgradeable smart contracts. All functionality is implemented through facets.

```solidity
// Get the diamond address by symbol
address diamondAddress = diamondFactory.getDiamondAddress("GEM");

// Check if a diamond exists
bool exists = diamondFactory.exists("GEM");

// Create a new diamond
DiamondSettings memory settings = DiamondSettings({
    name: "Gemforce",
    symbol: "GEM",
    // Other settings...
});
address newDiamond = diamondFactory.createFromSet(
    settings,
    initializer,
    initData,
    "defaultFacetSet"
);
```

### Identity Management

```solidity
// Create a new identity
identityFactory.createIdentity(userAddress);

// Get an identity
address identityAddress = identityFactory.getIdentity(userAddress);

// Add a claim topic
claimTopicsRegistry.addClaimTopic(1); // e.g., KYC claim

// Add a trusted issuer
uint256[] memory topics = new uint256[](1);
topics[0] = 1; // KYC claim topic
trustedIssuersRegistry.addTrustedIssuer(issuerAddress, topics);

// Add a claim
identity.addClaim(
    1, // topic
    1, // scheme
    issuerAddress,
    signature,
    data,
    "https://example.com/claim"
);

// Check if an identity has a claim
bool hasKYC = identity.getClaim(claimId) != 0;
```

### Token Management

```solidity
// Mint a new token
Attribute[] memory attributes = new Attribute[](2);
attributes[0] = Attribute("name", AttributeType.String, "Carbon Credit");
attributes[1] = Attribute("amount", AttributeType.Number, "100");
uint256 tokenId = gemforceMinter.gemforceMint(attributes);

// Purchase a token
marketplace.purchaseItem(diamondAddress, tokenId);

// Retire carbon credits
carbonCredits.retireCarbonCredits(tokenId, 50);
```

## Cloud Function APIs

### Authentication

```javascript
// Register a user
Parse.Cloud.run("registerUser", {
  username: "user@example.com",
  password: "password123",
  email: "user@example.com",
  company: "Example Corp",
  firstName: "John",
  lastName: "Doe"
}).then(result => {
  console.log("User registered:", result);
});

// Login (using Parse SDK)
Parse.User.logIn("username", "password").then(user => {
  console.log("Logged in:", user);
});
```

### DFNS Wallet Management

```javascript
// Register with DFNS
Parse.Cloud.run("registerInit", {
  username: "user@example.com"
}).then(challenge => {
  // Handle challenge with WebAuthn
  // Then complete registration
  return Parse.Cloud.run("registerComplete", {
    signedChallenge: signedChallenge,
    temporaryAuthenticationToken: tempToken
  });
}).then(result => {
  console.log("DFNS registration complete:", result);
});

// Login to DFNS
Parse.Cloud.run("login", {
  username: "user@example.com"
}).then(result => {
  const dfnsToken = result.token;
  // Store DFNS token for future operations
});

// List wallets
Parse.Cloud.run("listWallets", {
  authToken: dfnsToken
}).then(wallets => {
  console.log("User wallets:", wallets);
});
```

### Contract Interactions via DFNS

This pattern is used for all blockchain interactions through DFNS:
1. Initialize the transaction
2. Sign the challenge client-side
3. Complete the transaction with the signed challenge

```javascript
// Example: Purchase an item
Parse.Cloud.run("dfnsInitiatePurchase", {
  tokenId: "123",
  walletId: "wallet_123",
  dfns_token: dfnsToken
}).then(async ({ challenge, requestBody }) => {
  // Sign the challenge client-side using DFNS SDK
  const signedChallenge = await signChallenge(challenge);
  
  // Complete the purchase
  return Parse.Cloud.run("dfnsCompletePurchase", {
    walletId: "wallet_123",
    dfns_token: dfnsToken,
    signedChallenge: signedChallenge,
    requestBody: requestBody
  });
}).then(result => {
  console.log("Purchase complete:", result);
});
```

### Identity Management via DFNS

```javascript
// Create an identity
Parse.Cloud.run("dfnsCreateIdentityInit", {
  ownerAddress: "0x123...",
  walletId: "wallet_123",
  dfns_token: dfnsToken
}).then(async ({ challenge, requestBody }) => {
  // Sign the challenge client-side
  const signedChallenge = await signChallenge(challenge);
  
  // Complete the identity creation
  return Parse.Cloud.run("dfnsCreateIdentityComplete", {
    walletId: "wallet_123",
    dfns_token: dfnsToken,
    signedChallenge: signedChallenge,
    requestBody: requestBody
  });
}).then(result => {
  console.log("Identity created:", result);
});

// Add a claim topic
Parse.Cloud.run("dfnsAddClaimTopicInit", {
  claimTopic: 1, // e.g., KYC claim
  walletId: "wallet_123",
  dfns_token: dfnsToken
}).then(async ({ challenge, requestBody }) => {
  // Sign the challenge client-side
  const signedChallenge = await signChallenge(challenge);
  
  // Complete adding claim topic
  return Parse.Cloud.run("dfnsAddClaimTopicComplete", {
    walletId: "wallet_123",
    dfns_token: dfnsToken,
    signedChallenge: signedChallenge,
    requestBody: requestBody
  });
}).then(result => {
  console.log("Claim topic added:", result);
});
```

### Carbon Credit Management

```javascript
// Retire carbon credits
Parse.Cloud.run("dfnsInitRetireCredits", {
  tokenId: "123",
  amount: "50",
  walletId: "wallet_123",
  dfns_token: dfnsToken
}).then(async ({ challenge, requestBody }) => {
  // Sign the challenge client-side
  const signedChallenge = await signChallenge(challenge);
  
  // Complete retirement
  return Parse.Cloud.run("dfnsCompleteRetireCredits", {
    walletId: "wallet_123",
    dfns_token: dfnsToken,
    signedChallenge: signedChallenge,
    requestBody: requestBody
  });
}).then(result => {
  console.log("Credits retired:", result);
});
```

### Bridge API Integration

```javascript
// Create an external account
Parse.Cloud.run("createExternalAccount", {
  customer_id: "cust_123",
  currency: "usd",
  account_owner_name: "John Doe",
  account_type: "us",
  account: {
    account_number: "123456789",
    routing_number: "123456789",
    checking_or_savings: "checking"
  }
}).then(result => {
  console.log("External account created:", result);
});

// Create a transfer
Parse.Cloud.run("createTransfer", {
  amount: "100.00",
  on_behalf_of: "cust_123",
  source: {
    currency: "usd",
    payment_rail: "ach",
    external_account_id: "ext_acct_123"
  },
  destination: {
    currency: "usdc",
    payment_rail: "ethereum",
    to_address: "0x123..."
  }
}).then(result => {
  console.log("Transfer created:", result);
});
```

### Plaid Integration

```javascript
// Get a Plaid link token
Parse.Cloud.run("getPlaidLinkToken", {
  customerId: "cust_123"
}).then(result => {
  const linkToken = result.link_token;
  // Use linkToken with Plaid Link
});

// Exchange a Plaid public token
Parse.Cloud.run("exchangePlaidPublicToken", {
  customerId: "cust_123",
  linkToken: "link-123",
  publicToken: "public-123"
}).then(result => {
  console.log("Plaid token exchanged:", result);
});
```

## Common Patterns

### Two-Step Transaction Pattern

Most blockchain interactions follow this pattern:

1. **Initialization Step**:
   - Call the `*Init` function with required parameters
   - Receive a challenge and request body

2. **Completion Step**:
   - Sign the challenge client-side (typically with WebAuthn)
   - Call the `*Complete` function with the signed challenge and request body
   - Receive transaction result

### Error Handling

```javascript
Parse.Cloud.run("someFunction", params)
  .then(result => {
    // Handle success
  })
  .catch(error => {
    // Parse Server error codes
    if (error.code === Parse.Error.VALIDATION_ERROR) {
      console.error("Validation error:", error.message);
    } else if (error.code === Parse.Error.SCRIPT_FAILED) {
      console.error("Script error:", error.message);
    } else {
      console.error("Unknown error:", error);
    }
  });
```

## Important Considerations

1. **Authentication**: Always ensure users are authenticated before accessing protected endpoints.

2. **DFNS Token Management**: Store the DFNS token securely and refresh it when needed.

3. **Transaction Monitoring**: Monitor transaction status after submission as blockchain transactions can take time to confirm.

4. **Gas Management**: Be mindful of gas costs for transactions, especially for operations like minting tokens.

5. **Error Handling**: Implement robust error handling for both client-side and server-side errors.

6. **Idempotency**: Use idempotency keys for financial operations to prevent duplicate transactions.

7. **Wallet Address Validation**: Validate Ethereum addresses before sending transactions.

8. **Testing**: Test all interactions on test networks before moving to production.