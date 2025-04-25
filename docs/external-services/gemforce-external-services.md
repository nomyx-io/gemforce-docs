# Gemforce External Services Integration

This document provides detailed information about the external services and third-party integrations used by the Gemforce platform. Understanding these services is essential for developers working with the Gemforce API.

## DFNS Wallet-as-a-Service

DFNS provides a secure and user-friendly wallet management system that Gemforce uses for managing blockchain transactions.

### Overview

DFNS is a wallet-as-a-service platform that offers:
- Non-custodial wallet management
- WebAuthn-based authentication
- Transaction signing without exposing private keys
- Delegated transaction execution

### Integration Points

Gemforce integrates with DFNS via:
- DFNS API Client
- DFNS Delegated API Client

### Key Features Used

1. **User Registration & Authentication**
   - WebAuthn-based registration
   - Passwordless authentication
   - Recovery mechanisms

2. **Wallet Management**
   - Wallet creation
   - Asset listing
   - Transaction history

3. **Transaction Signing**
   - Challenge-based signing
   - Two-step transaction process (Init/Complete)
   - Delegated transaction execution

### API Flow

```
Client App <-> Gemforce Cloud Functions <-> DFNS API
```

1. Client initiates a request to Gemforce Cloud Functions
2. Gemforce creates a transaction payload and requests a challenge from DFNS
3. Challenge is passed to the client for signing with WebAuthn
4. Signed challenge is sent back to Gemforce
5. Gemforce completes the transaction with DFNS
6. DFNS broadcasts the transaction to the blockchain

### Configuration

DFNS requires the following environment variables:
- `DFNS_APP_ID`: Application ID for DFNS
- `DFNS_API_URL`: Base URL for DFNS API
- `DFNS_CRED_ID`: Credential ID for DFNS
- `DFNS_AUTH_TOKEN`: Authentication token (for server operations)

## Bridge API Integration

Bridge API provides financial services integration for Gemforce, enabling external account management, transfers, and KYC processes.

### Overview

Bridge API offers:
- External account management
- Transfer operations between traditional finance and crypto
- KYC verification
- Plaid integration for banking connections

### Integration Points

Gemforce integrates with Bridge API via RESTful HTTP endpoints.

### Key Features Used

1. **External Account Management**
   - Creation of external accounts
   - Multiple account types (US, IBAN)
   - Account listing and retrieval
   - Account deletion

2. **Transfer Operations**
   - Cross-currency transfers
   - Multiple payment rails (ACH, SEPA, Wire, Blockchain)
   - Transfer status tracking
   - Idempotent operations

3. **KYC Management**
   - KYC link generation
   - KYC status tracking
   - Individual and business verification

4. **Plaid Integration**
   - Link token generation
   - Token exchange
   - Account verification

### API Flow

```
Client App <-> Gemforce Cloud Functions <-> Bridge API
```

1. Client app calls Gemforce Cloud Functions
2. Gemforce validates and formats the request
3. Gemforce calls Bridge API with appropriate headers
4. Bridge API processes the request and returns a response
5. Gemforce processes and returns the response to the client

### Configuration

Bridge API requires the following environment variables:
- `BASE_BRIDGE_URL`: Base URL for Bridge API
- `BRIDGE_API_KEY`: API key for authentication

## Parse Server

Parse Server provides the backend infrastructure for Gemforce's cloud functions and data storage.

### Overview

Parse Server offers:
- User authentication and management
- Cloud functions
- Database operations
- File storage
- Push notifications

### Integration Points

Gemforce uses Parse Server as the primary backend platform.

### Key Features Used

1. **User Management**
   - Registration
   - Email verification
   - Password reset
   - Session management

2. **Cloud Functions**
   - Blockchain operations
   - DFNS integration
   - Bridge API integration
   - Business logic

3. **Data Storage**
   - User profiles
   - Blockchain data
   - Identity information
   - Transaction history

4. **Role-Based Access Control**
   - User roles
   - Permission management
   - Object-level ACLs

### Configuration

Parse Server requires the following environment variables:
- `APP_ID`: Parse application ID
- `MASTER_KEY`: Parse master key
- `DATABASE_URI`: MongoDB connection string
- `SERVER_URL`: Parse Server URL
- `PROJECT_WIZARD_URL`: URL for the project wizard

## Ethereum Blockchain Networks

Gemforce interacts with multiple Ethereum-compatible blockchain networks.

### Overview

Gemforce supports multiple blockchain networks including:
- Ethereum Mainnet
- BaseSepolia
- Other EVM-compatible networks

### Integration Points

Gemforce interacts with blockchain networks via:
- JSON-RPC providers
- WebSocket connections

### Key Features Used

1. **Smart Contract Deployment**
   - Diamond contract deployment
   - Facet deployment
   - Contract initialization

2. **Transaction Submission**
   - Method calls
   - Value transfers
   - Contract interactions

3. **Event Monitoring**
   - WebSocket subscriptions
   - Event filtering
   - Log parsing

4. **State Reading**
   - View function calls
   - State synchronization

### Configuration

Blockchain integration requires the following environment variables:
- `ETH_NODE_URI_[NETWORK]`: JSON-RPC endpoint for each network
- `CHAIN_ID`: ID of the default chain
- `METADATA_BASE_URI`: Base URI for token metadata

## SendGrid Email Service

SendGrid is used for transactional email communications.

### Overview

SendGrid provides email delivery services for:
- User verification
- Password reset
- Notifications
- Other transactional emails

### Integration Points

Gemforce uses SendGrid's Node.js SDK.

### Key Features Used

1. **Email Templates**
   - EJS templating
   - HTML email content
   - Dynamic content insertion

2. **Email Sending**
   - Transactional emails
   - HTML content
   - Attachments

### Configuration

SendGrid requires the following environment variables:
- SendGrid API key (configured through environment variables)
- From email address

## Plaid (via Bridge API)

Plaid is integrated through Bridge API to provide banking connection services.

### Overview

Plaid offers:
- Bank account verification
- Account linking
- Transaction history
- Balance information

### Integration Points

Gemforce interacts with Plaid through Bridge API.

### Key Features Used

1. **Link Token Creation**
   - Generated for each user session
   - Configured for specific use cases

2. **Public Token Exchange**
   - Convert public tokens to access tokens
   - Securely store access tokens

### API Flow

```
Client App <-> Plaid Link <-> Client App <-> Gemforce <-> Bridge API <-> Plaid
```

1. Client requests a Plaid link token from Gemforce
2. Gemforce obtains the token through Bridge API
3. Client uses the token with Plaid Link
4. Plaid Link provides a public token to the client
5. Client sends the public token to Gemforce
6. Gemforce exchanges it via Bridge API
7. Bridge API handles Plaid API communication

## Integration Diagram

```
┌─────────────────────┐
│                     │
│    Client App       │
│                     │
└─────────┬───────────┘
          │
          │
┌─────────▼───────────┐         ┌─────────────────────┐
│                     │         │                     │
│    Parse Server     ├─────────►      DFNS           │
│    (Cloud Functions)│         │  Wallet-as-Service  │
│                     │         │                     │
└─────────┬───────────┘         └─────────────────────┘
          │
          │
┌─────────┼───────────┐         ┌─────────────────────┐
│         │           │         │                     │
│  ┌──────▼───────┐   │         │     Blockchain      │
│  │              │   │         │     Networks        │
│  │  Bridge API  │   │         │                     │
│  │              │   │         │  - Ethereum Mainnet │
│  └──────┬───────┘   │         │  - BaseSepolia      │
│         │           │         │  - Others           │
│         │           │         │                     │
│  ┌──────▼───────┐   │         └──────────┬──────────┘
│  │              │   │                    │
│  │    Plaid     │   │                    │
│  │              │   │                    │
│  └──────────────┘   │                    │
│                     │                    │
└─────────────────────┘                    │
          ▲                                │
          │                                │
          │                                │
┌─────────┴────────────────────────────────▼──────┐
│                                                 │
│               Smart Contracts                   │
│                                                 │
│  - Diamond                                      │
│  - Identity System                              │
│  - Asset Management                             │
│  - Carbon Credits                               │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Authentication Flows

### User Registration & Login

1. User registers through Parse Server
2. Email verification is sent via SendGrid
3. User verifies email
4. User logs in with username/password
5. Parse Server creates a session token

### DFNS Registration

1. User initiates DFNS registration
2. Gemforce requests a registration challenge from DFNS
3. User completes WebAuthn registration on client
4. Signed challenge is sent to Gemforce
5. Gemforce completes registration with DFNS
6. DFNS creates a wallet for the user

### Bridge API Authentication

Bridge API authentication is handled server-side with API keys. The client never interacts directly with Bridge API credentials.

## Security Considerations

### API Keys & Secrets

- All API keys and secrets are stored as environment variables
- API keys are never exposed to clients
- All external API calls are made server-side

### Delegated Authentication

- DFNS uses delegated authentication
- Client never has access to private keys
- WebAuthn provides phishing-resistant authentication

### Idempotency

- Bridge API calls use idempotency keys
- Prevents duplicate transactions
- Allows safe retries

## Rate Limiting

### DFNS

- DFNS imposes rate limits on API calls
- Gemforce implements exponential backoff for retries

### Bridge API

- Bridge API has rate limits based on API key
- Gemforce handles rate limit errors

## Error Handling

### DFNS Errors

- Challenge-related errors
- WebAuthn errors
- Transaction errors

### Bridge API Errors

- Validation errors
- Processing errors
- External account errors
- KYC errors

### Blockchain Errors

- Gas-related errors
- Transaction failure
- Network congestion

## Monitoring & Logging

All external service interactions are logged for:
- Debugging
- Audit trails
- Performance monitoring
- Error tracking

## Conclusion

Gemforce integrates with several external services to provide a comprehensive platform. Understanding these integrations is crucial for effectively working with the Gemforce API.