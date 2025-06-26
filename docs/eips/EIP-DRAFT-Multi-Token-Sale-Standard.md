# EIP-DRAFT: Multi-Token Sale Standard with Proof-Based Purchases

## Simple Summary

A standardized interface for conducting token sales that supports multiple token types (ERC20, ERC721, ERC1155), multiple payment methods, and cryptographic proof-based purchases for allowlists and presales.

## Abstract

This EIP proposes a comprehensive standard for token sales that enables:
- Sales of ERC20, ERC721, and ERC1155 tokens through a unified interface
- Support for ETH and ERC20 token payments
- Cryptographic proof-based purchases (e.g., Merkle proofs for allowlists)
- Per-account purchase limits
- Variable pricing mechanisms
- Integration with identity systems for access control

## Motivation

Current token sale implementations are fragmented across different token types and lack standardization for advanced features like proof-based purchases and multi-payment support. This standard provides a unified approach that can handle various token sale scenarios while maintaining security and flexibility.

## Specification

### Core Interface

```solidity
interface IMultiTokenSale {
    enum TokenType { ERC20, ERC721, ERC1155 }
    enum PaymentMethod { ETH, ERC20 }
    
    struct PriceSettings {
        uint256 price;
        // Additional price configuration can be extended
    }
    
    struct MultiSaleSettings {
        address token;
        TokenType tokenType;
        address owner;
        PriceSettings price;
        PaymentMethod paymentMethod;
        address paymentToken;
        uint256 maxQuantityPerAccount;
        // Additional settings...
    }
    
    struct MultiSalePurchase {
        uint256 multiSaleId;
        address purchaser;
        address receiver;
        uint256 quantity;
    }
    
    struct MultiSaleProof {
        bytes proof;
        bytes data;
    }
    
    // Events
    event MultiSaleCreated(uint256 indexed tokenSaleId, MultiSaleSettings settings);
    event MultiSaleUpdated(uint256 indexed tokenSaleId, MultiSaleSettings settings);
    event MultiSaleSold(uint256 indexed multiSaleId, address indexed purchaser, uint256[] tokenIds, bytes data);
    
    // Core Functions
    function createTokenSale(MultiSaleSettings memory tokenSaleInit) external returns (uint256 tokenSaleId);
    function updateTokenSaleSettings(uint256 tokenSaleId, MultiSaleSettings memory settings) external;
    function purchaseProof(MultiSalePurchase memory purchaseInfo, MultiSaleProof memory purchaseProofParam) external payable returns (uint256[] memory ids);
    function purchase(uint256 multiSaleId, address purchaser, address receiver, uint256 quantity, bytes memory data) external payable returns (uint256[] memory ids);
    function getTokenSaleSettings(uint256 tokenSaleId) external view returns (MultiSaleSettings memory settings);
    function getTokenSaleIds() external view returns (uint256[] memory);
    function listTokenSales() external view returns (MultiSaleSettings[] memory);
}
```

### Key Features

#### 1. Multi-Token Support
- **ERC20**: Fungible token sales with quantity-based purchases
- **ERC721**: NFT sales with unique token ID generation
- **ERC1155**: Semi-fungible token sales with ID-based minting

#### 2. Proof-Based Purchases
- Support for cryptographic proofs (e.g., Merkle proofs)
- Enables allowlist functionality
- Presale and whitelist implementations
- Flexible proof validation system

#### 3. Payment Flexibility
- Native ETH payments with automatic refunds
- ERC20 token payments with approval-based transfers
- Per-sale payment method configuration

#### 4. Purchase Controls
- Per-account quantity limits
- Total sale quantity tracking
- Owner-based access control for sale management

### Proof System

The proof system enables sophisticated access control:

```solidity
// Example Merkle proof verification
function verifyMerkleProof(bytes32[] memory proof, bytes32 root, bytes32 leaf) internal pure returns (bool) {
    bytes32 computedHash = leaf;
    for (uint256 i = 0; i < proof.length; i++) {
        bytes32 proofElement = proof[i];
        if (computedHash <= proofElement) {
            computedHash = keccak256(abi.encodePacked(computedHash, proofElement));
        } else {
            computedHash = keccak256(abi.encodePacked(proofElement, computedHash));
        }
    }
    return computedHash == root;
}
```

## Rationale

### Unified Interface
A single interface for multiple token types reduces complexity and improves interoperability between different sale implementations.

### Proof-Based Access Control
Cryptographic proofs enable sophisticated access control without requiring on-chain storage of allowlists, reducing gas costs.

### Flexible Payment System
Supporting both ETH and ERC20 payments provides maximum flexibility for different use cases and user preferences.

### Modular Design
The standard is designed to be extensible, allowing implementations to add custom features while maintaining compatibility.

## Implementation Details

### Token Minting Strategy

```solidity
function _mintPurchasedTokens(uint256 multiSaleId, address recipient, uint256 amount, bytes memory data) internal returns (uint256[] memory tokenIds) {
    MultiSaleContract storage sale = _tokenSales[multiSaleId];
    
    if (sale.settings.tokenType == TokenType.ERC20) {
        IERC20Mint(sale.settings.token).mintTo(recipient, amount);
        return new uint256[](0); // No specific token IDs for ERC20
    } else if (sale.settings.tokenType == TokenType.ERC721) {
        tokenIds = new uint256[](amount);
        for (uint256 i = 0; i < amount; i++) {
            uint256 tokenId = _getNextTokenId();
            IERC721Mint(sale.settings.token).mintTo(recipient, 1, data);
            tokenIds[i] = tokenId;
        }
    } else if (sale.settings.tokenType == TokenType.ERC1155) {
        tokenIds = new uint256[](1);
        tokenIds[0] = sale.nonce++;
        IERC1155Mint(sale.settings.token).mintTo(recipient, tokenIds[0], amount, data);
    }
}
```

### Security Considerations

1. **Reentrancy Protection**: All purchase functions must use reentrancy guards
2. **Integer Overflow**: Use SafeMath or Solidity 0.8+ overflow protection
3. **Access Control**: Proper validation of sale ownership and permissions
4. **Payment Validation**: Verify payment amounts and handle excess payments
5. **Proof Validation**: Secure implementation of proof verification systems

### Gas Optimization

- Batch operations where possible
- Efficient storage patterns
- Minimal external calls during purchases
- Optimized proof verification algorithms

## Backwards Compatibility

This standard is designed to work with existing ERC20, ERC721, and ERC1155 token contracts that implement the respective minting interfaces.

## Test Cases

Comprehensive test cases should cover:
- Multi-token type sales
- Proof-based purchase validation
- Payment method handling
- Purchase limit enforcement
- Access control mechanisms
- Edge cases and error conditions

## Reference Implementation

The reference implementation includes:
- [`MultiSaleFacet.sol`](../contracts/facets/MultiSaleFacet.sol) - Core multi-sale functionality
- [`IMultiSale.sol`](../contracts/interfaces/IMultiSale.sol) - Interface definition
- [`MultiSaleLib.sol`](../contracts/libraries/MultiSaleLib.sol) - Supporting library functions

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).