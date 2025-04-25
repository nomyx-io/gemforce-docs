# Gemforce API Documentation

This document provides a comprehensive overview of the Gemforce API, detailing both the smart contract interfaces and cloud functions that power the system.

## Table of Contents

1. [Introduction](#introduction)
2. [Smart Contract APIs](#smart-contract-apis)
   - [Diamond Pattern](#diamond-pattern)
   - [DiamondFactory](#diamondfactory)
   - [Identity Management](#identity-management)
   - [Token and Asset Management](#token-and-asset-management)
   - [Marketplace](#marketplace)
   - [Carbon Credits](#carbon-credits)
3. [Cloud Function APIs](#cloud-function-apis)
   - [Authentication Functions](#authentication-functions)
   - [Blockchain Management](#blockchain-management)
   - [Contract Interaction](#contract-interaction)
   - [DFNS Wallet Management](#dfns-wallet-management)
   - [Bridge API Integration](#bridge-api-integration)
   - [Project Management](#project-management)

## Introduction

Gemforce is a blockchain-powered platform that implements a Diamond pattern (EIP-2535) for upgradeable smart contracts, digital identity management, and token management. The platform combines on-chain smart contracts with off-chain cloud functions to provide a comprehensive solution for issuing, managing, and trading digital assets with verifiable claims.

## Smart Contract APIs

### Diamond Pattern

The Diamond pattern is the foundation of Gemforce's smart contract architecture. It provides a way to create upgradeable contracts that can be extended with new functionality without breaking existing functionality.

#### Diamond.sol

The Diamond contract is the main contract that implements the Diamond pattern.

**Key functions:**

* `initialize(address _owner, DiamondSettings memory params, IDiamondCut.FacetCut[] memory _facets, address diamondInit, bytes calldata _calldata)`: Initialize the diamond with owner, settings, and facets.
* `diamondCut(IDiamondCut.FacetCut[] calldata _diamondCut, address _init, bytes calldata _calldata)`: Add, replace, or remove functions from the diamond.
* `facets()`: Get all facets and their selectors.
* `facetFunctionSelectors(address _facet)`: Get all function selectors for a facet.
* `facetAddresses()`: Get all facet addresses used by a diamond.
* `facetAddress(bytes4 _functionSelector)`: Get the facet that supports a given selector.
* `transferOwnership(address _newOwner)`: Transfer ownership of the diamond.
* `owner()`: Get the owner of the diamond.

### DiamondFactory

The DiamondFactory contract is used to create new Diamond contracts.

**Key functions:**

* `initialize(DiamondFactoryInit memory initData)`: Initialize the factory with a set of facets.
* `getFacets(string memory facetSet)`: Get the facets for a facet set.
* `setFacet(string memory facetSet, uint256 idx, IDiamondCut.FacetCut memory facetAddress)`: Set a facet in a facet set.
* `setFacets(string memory facetSet, IDiamondCut.FacetCut[] memory facetAddress)`: Set multiple facets in a facet set.
* `removeFacets(string memory facetSet)`: Remove a facet set.
* `getDiamondAddress(string memory symbol)`: Get the address of a diamond by symbol.
* `create(DiamondSettings memory params, address diamondInit, bytes calldata _calldata, IDiamondCut.FacetCut[] memory facets)`: Create a new diamond with custom facets.
* `createFromSet(DiamondSettings memory params, address diamondInit, bytes calldata _calldata, string memory facets)`: Create a new diamond from a predefined facet set.
* `add(string memory symbol, address payable diamondAddress)`: Add an existing diamond to the factory.
* `remove(string memory symbol)`: Remove a diamond from the factory.
* `exists(string memory symbol)`: Check if a diamond exists.
* `symbols()`: Get all diamond symbols in the factory.

### Identity Management

Gemforce includes a digital identity system that allows for claims to be made about identities.

#### Identity.sol

The Identity contract represents a digital identity for a user.

**Key functions:**

* `initialize(address _owner, address _identityRegistry, address _trustedIssuerRegistry)`: Initialize the identity with an owner, identity registry, and trusted issuer registry.
* `getAttribute(string memory _key)`: Get an attribute for a token ID keyed by string.
* `setAttribute(string memory key, AttributeType attributeType, string memory value)`: Set an attribute for a token ID.
* `addKey(bytes32 _key, uint256 _purpose, uint256 _keyType)`: Add a key to the identity.
* `removeKey(bytes32 _key, uint256 _purpose)`: Remove a key from the identity.
* `getKey(bytes32 _key)`: Get a key from the identity.
* `addClaim(uint256 _topic, uint256 _scheme, address _issuer, bytes memory _signature, bytes memory _data, string memory _uri)`: Add a claim to the identity.
* `removeClaim(bytes32 _claimId)`: Remove a claim from the identity.
* `getClaim(bytes32 _claimId)`: Get a claim by ID.
* `getClaimIdsByTopic(uint256 _topic)`: Get claim IDs by topic.
* `getClaimTopics()`: Get all claim topics for the identity.
* `isVerified()`: Check if the identity is verified.

#### IdentityFactory.sol

The IdentityFactory contract creates new Identity contracts.

**Key functions:**

* `initialize(address identityRegistry, address trustedIssuerRegistry)`: Initialize the factory with identity and trusted issuer registries.
* `createIdentity(address ownerAddress)`: Create a new identity for an owner.
* `removeIdentity(address ownerAddress)`: Remove an identity for an owner.
* `getIdentity(address identityOwner)`: Get the identity of an owner.
* `getIdentityUsers()`: Get all identity users.

#### ClaimTopicsRegistryFacet.sol

Manages claim topics that can be used to make claims about identities.

**Key functions:**

* `addClaimTopic(uint256 _claimTopic)`: Add a claim topic to the registry.
* `removeClaimTopic(uint256 _claimTopic)`: Remove a claim topic from the registry.
* `getClaimTopics()`: Get all claim topics.

#### TrustedIssuersRegistryFacet.sol

Manages trusted issuers that can make claims about identities.

**Key functions:**

* `addTrustedIssuer(address _trustedIssuer, uint256[] calldata _claimTopics)`: Add a trusted issuer with claim topics.
* `removeTrustedIssuer(address _trustedIssuer)`: Remove a trusted issuer.
* `updateIssuerClaimTopics(address _trustedIssuer, uint256[] calldata _claimTopics)`: Update claim topics for a trusted issuer.
* `getTrustedIssuers()`: Get all trusted issuers.
* `getTrustedIssuerClaimTopics(address _trustedIssuer)`: Get claim topics for a trusted issuer.
* `isTrustedIssuer(address _issuer)`: Check if an address is a trusted issuer.
* `hasClaimTopic(address _trustedIssuer, uint256 _claimTopic)`: Check if a trusted issuer has a claim topic.

### Token and Asset Management

#### GemforceMinterFacet.sol

The GemforceMinterFacet contract is used to mint tokens and set attributes.

**Key functions:**

* `gemforceMint(Attribute[] memory metadata)`: Mint a new token with metadata.

### Marketplace

#### MarketplaceFacet.sol

The MarketplaceFacet contract provides functionality for buying and selling tokens.

**Key functions:**

* `purchaseItem(address contract, uint256 tokenId)`: Purchase a token.

### Carbon Credits

#### CarbonCreditFacet.sol

The CarbonCreditFacet contract provides functionality for managing carbon credits.

**Key functions:**

* `retireCarbonCredits(uint256 tokenId, uint256 amount)`: Retire carbon credits for a token.

## Cloud Function APIs

Gemforce provides a set of cloud functions built on Parse Server that bridge the gap between client applications and the blockchain.

### Authentication Functions

#### User Registration and Management

* `registerUser`: Register a new user with username, password, email, company, and name.
* `verifyEmail`: Verify a user's email with a token.
* `retrieveEmailFromToken`: Get the email associated with a token.
* `requestPasswordReset`: Request a password reset for a user.
* `resetPassword`: Reset a user's password with a token.
* `updateUserByEmail`: Update a user's profile by email.
* `isUserOnboarded`: Check if a user has completed onboarding.
* `getUsersWithIdentityWallets`: Get users who have identity wallets.

### Blockchain Management

#### Network and Provider Management

* `loadAllBlockchains`: Get all configured blockchains.
* `loadProviderUrl`: Get the RPC endpoint for a network ID.
* `loadProviderWebSocketUrl`: Get the WebSocket endpoint for a network ID.
* `loadAllProviderUrls`: Get all provider URLs.
* `loadBlockchainDataForNetwork`: Get blockchain data (signer, provider, wallet) for a network.

### Contract Interaction

#### Generic Contract Interaction

* `addDiamondFacet`: Add a facet to a diamond.
* `callMethod`: Call a method on a contract.
* `viewMethod`: View a method on a contract.
* `callContractMethod`: Call a method on a contract with custom parameters.
* `viewContractMethod`: View a method on a contract with custom parameters.
* `loadSmartContractForNetwork`: Load a smart contract for a network.
* `loadSmartContractsForNetwork`: Load all smart contracts for a network.

### DFNS Wallet Management

DFNS is a wallet-as-a-service solution integrated with Gemforce for managing user wallets.

#### User Registration and Authentication

* `registerInit`: Initialize DFNS user registration.
* `registerComplete`: Complete DFNS user registration.
* `login`: Login to DFNS.
* `recoverInit`: Initialize account recovery for DFNS.
* `recoverComplete`: Complete account recovery for DFNS.

#### Wallet Management

* `listWallets`: List DFNS wallets.
* `signaturesInit`: Initialize a signature.
* `signaturesComplete`: Complete a signature.
* `dfnsGetWallet`: Get a DFNS wallet by ID.
* `dfnsGetUSDC`: Get USDC balance for a wallet.

#### Transaction Management

* `dfnsInitApproval` / `dfnsCompleteApproval`: Approve tokens for spending.
* `dfnsInitTransferUSDC` / `dfnsCompleteTransferUSDC`: Transfer USDC.
* `dfnsInitWithdraw` / `dfnsCompleteWithdraw`: Withdraw tokens from the treasury.
* `dfnsInitiatePurchase` / `dfnsCompletePurchase`: Purchase an item from the marketplace.
* `dfnsInitRetireCredits` / `dfnsCompleteRetireCredits`: Retire carbon credits.

#### Identity Management

* `dfnsAddClaimTopicInit` / `dfnsAddClaimTopicComplete`: Add a claim topic.
* `dfnsAddTrustedIssuerInit` / `dfnsAddTrustedIssuerComplete`: Add a trusted issuer.
* `dfnsRemoveTrustedIssuerInit` / `dfnsRemoveTrustedIssuerComplete`: Remove a trusted issuer.
* `dfnsUpdateIssuerClaimTopicsInit` / `dfnsUpdateIssuerClaimTopicsComplete`: Update claim topics for an issuer.
* `dfnsCreateIdentityInit` / `dfnsCreateIdentityComplete`: Create a digital identity.
* `dfnsAddIdentityInit` / `dfnsAddIdentityComplete`: Add an identity to the registry.
* `dfnsRemoveIdentityInit` / `dfnsRemoveIdentityComplete`: Remove an identity from the factory.
* `dfnsUnregisterIdentityInit` / `dfnsUnregisterIdentityComplete`: Remove an identity from the registry.
* `dfnsSetClaimsInit` / `dfnsSetClaimsComplete`: Set claims for an identity.
* `dfnsAddClaimInit` / `dfnsAddClaimComplete`: Add a claim to an identity.
* `dfnsRemoveClaimInit` / `dfnsRemoveClaimComplete`: Remove a claim from an identity.
* `dfnsGetIdentity`: Get the identity for an owner address.

#### Asset Management

* `dfnsGemforceMintInit` / `dfnsGemforceMintComplete`: Mint a new token with metadata.

### Bridge API Integration

Gemforce integrates with a Bridge API for handling financial operations.

#### External Accounts

* `createExternalAccount`: Create an external account for a customer.
* `getExternalAccounts`: Get all external accounts for a customer.
* `getExternalAccount`: Get a specific external account for a customer.
* `deleteExternalAccount`: Delete an external account.

#### Transfers

* `createTransfer`: Create a transfer between accounts.
* `getCustomerTransfers`: Get all transfers for a customer.

#### KYC Management

* `generateKycLink`: Generate a KYC link for a user.
* `getKycLinkStatus`: Get the status of a KYC link.

#### Plaid Integration

* `getPlaidLinkToken`: Get a Plaid link token for a customer.
* `exchangePlaidPublicToken`: Exchange a Plaid public token for access.

### Project Management

* `getProjectMetadata`: Get metadata for a project.