# MultiSaleFacet

## Overview

The [`MultiSaleFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/MultiSaleFacet.sol) provides comprehensive multi-token sale functionality within the Gemforce diamond system. This facet enables the creation and management of token sales supporting multiple token standards (ERC20, ERC721, ERC1155) with flexible payment methods, allowlist support, and advanced purchase controls.

## Contract Details

- **Contract Name**: `MultiSaleFacet`
- **Inheritance**: `IMultiSale`, `Modifiers`, `ReentrancyGuard`
- **License**: MIT
- **Solidity Version**: >=0.8.0

## Key Features

### 🔹 Multi-Token Standard Support
- ERC20 fungible token sales
- ERC721 NFT individual sales
- ERC1155 semi-fungible token sales
- Automatic token type detection and minting

### 🔹 Flexible Payment Methods
- ETH payments with automatic refund of excess
- ERC20 token payments
- Configurable payment tokens per sale
- Secure payment processing with reentrancy protection

### 🔹 Advanced Sale Controls
- Per-account purchase limits
- Allowlist support with cryptographic proofs
- Variable pricing mechanisms
- Owner and admin access controls

### 🔹 Comprehensive Sale Management
- Create, update, and query token sales
- Track purchase history and quantities
- Batch operations for efficiency
- Event-driven architecture for transparency

## Core Data Structures

### MultiSaleSettings
```solidity
struct MultiSaleSettings {
    address token;                    // Token contract address
    TokenType tokenType;             // ERC20, ERC721, or ERC1155
    address owner;                   // Sale owner (receives payments)
    VariablePriceContract price;     // Price configuration
    PaymentMethod paymentMethod;     // ETH or ERC20
    address paymentToken;            // ERC20 token for payments (if applicable)
    uint256 maxQuantityPerAccount;   // Purchase limit per account (0 = unlimited)
    // Additional configuration fields...
}
```

### MultiSalePurchase
```solidity
struct MultiSalePurchase {
    uint256 multiSaleId;    // ID of the token sale
    address purchaser;      // Address making the purchase
    address receiver;       // Address receiving the tokens
    uint256 quantity;       // Number of tokens to purchase
}
```

### MultiSaleProof
```solidity
struct MultiSaleProof {
    bytes proof;    // Cryptographic proof (e.g., Merkle proof)
    bytes data;     // Additional proof or purchase data
}
```

## Core Functions

### Sale Management Functions

#### `createTokenSale()`
```solidity
function createTokenSale(MultiSaleSettings memory tokenSaleInit) external virtual onlyOwner returns (uint256 tokenSaleId)
```

**Purpose**: Creates a new token sale with specified configuration.

**Parameters**:
- `tokenSaleInit` (MultiSaleSettings): Complete sale configuration

**Returns**:
- `tokenSaleId` (uint256): Unique identifier for the new sale

**Access Control**: Owner only

**Process**:
1. Generates unique token sale ID
2. Stores complete sale configuration
3. Adds sale ID to tracking array
4. Emits MultiSaleCreated event

**Events**: `MultiSaleCreated(tokenSaleId, settings)`

**Example Usage**:
```solidity
// Create NFT sale with ETH payment
MultiSaleSettings memory nftSale = MultiSaleSettings({
    token: nftContractAddress,
    tokenType: TokenType.ERC721,
    owner: saleOwner,
    price: VariablePriceContract({
        price: 0.1 ether,
        // Additional price configuration...
    }),
    paymentMethod: PaymentMethod.ETH,
    paymentToken: address(0),
    maxQuantityPerAccount: 5
    // Additional settings...
});

uint256 saleId = IMultiSale(diamond).createTokenSale(nftSale);
console.log("Created NFT sale with ID:", saleId);
```

---

#### `updateTokenSaleSettings()`
```solidity
function updateTokenSaleSettings(uint256 tokenSaleId, MultiSaleSettings memory settings) external
```

**Purpose**: Updates configuration for an existing token sale.

**Parameters**:
- `tokenSaleId` (uint256): ID of the sale to update
- `settings` (MultiSaleSettings): New configuration settings

**Access Control**: Sale owner or contract owner

**Process**:
1. Validates caller permissions
2. Replaces existing settings with new configuration
3. Emits MultiSaleUpdated event

**Events**: `MultiSaleUpdated(tokenSaleId, settings)`

**Example Usage**:
```solidity
// Update sale price and purchase limit
MultiSaleSettings memory updatedSettings = existingSettings;
updatedSettings.price.price = 0.15 ether;
updatedSettings.maxQuantityPerAccount = 3;

IMultiSale(diamond).updateTokenSaleSettings(saleId, updatedSettings);
```

### Purchase Functions

#### `purchase()`
```solidity
function purchase(
    uint256 multiSaleId,
    address purchaser,
    address receiver,
    uint256 quantity,
    bytes memory data
) external payable nonReentrant returns (uint256[] memory ids)
```

**Purpose**: Purchases tokens from a sale without requiring cryptographic proof.

**Parameters**:
- `multiSaleId` (uint256): ID of the token sale
- `purchaser` (address): Address making the purchase (defaults to msg.sender if zero)
- `receiver` (address): Address receiving tokens (defaults to msg.sender if zero)
- `quantity` (uint256): Number of tokens to purchase
- `data` (bytes): Additional data for minting process

**Returns**:
- `ids` (uint256[]): Array of minted token IDs

**Security**: Uses nonReentrant modifier for payment protection

**Process**:
1. Retrieves sale configuration and calculates total price
2. Validates and processes payment (ETH or ERC20)
3. Sets default addresses if not provided
4. Checks purchase limits and constraints
5. Processes purchase through MultiSaleLib
6. Mints tokens to receiver
7. Updates sale state and tracking
8. Emits purchase event

**Payment Handling**:
- **ETH**: Validates sufficient payment, refunds excess automatically
- **ERC20**: Validates no ETH sent, transfers payment tokens

**Example Usage**:
```solidity
// Purchase 3 NFTs with ETH
uint256[] memory tokenIds = IMultiSale(diamond).purchase{value: 0.3 ether}(
    saleId,
    address(0),  // Use msg.sender as purchaser
    address(0),  // Use msg.sender as receiver
    3,           // Purchase 3 tokens
    ""           // No additional data
);

console.log("Purchased", tokenIds.length, "NFTs");
```

**Error Conditions**:
- `"Insufficient ETH sent"` - Not enough ETH for purchase
- `"Do not send ETH for ERC20 payments"` - ETH sent for ERC20 sale
- `"maxperaccount"` - Exceeds per-account purchase limit
- `"ETH transfer failed"` - Refund transfer failure

---

#### `purchaseProof()`
```solidity
function purchaseProof(
    MultiSalePurchase memory purchaseInfo,
    MultiSaleProof memory purchaseProofParam
) external payable nonReentrant returns (uint256[] memory ids)
```

**Purpose**: Purchases tokens using cryptographic proof for allowlist or presale access.

**Parameters**:
- `purchaseInfo` (MultiSalePurchase): Purchase details
- `purchaseProofParam` (MultiSaleProof): Cryptographic proof and data

**Returns**:
- `ids` (uint256[]): Array of minted token IDs

**Security**: Uses nonReentrant modifier and proof validation

**Process**:
1. Retrieves sale configuration
2. Validates purchase limits against receiver address
3. Sets default addresses if not provided
4. Validates and processes payment
5. Validates cryptographic proof through MultiSaleLib
6. Mints tokens to receiver
7. Updates sale state and tracking
8. Emits purchase event

**Proof Types**:
- **Merkle Proofs**: For allowlist verification
- **Signature Proofs**: For presale authorization
- **Custom Proofs**: Application-specific verification

**Example Usage**:
```solidity
// Purchase with Merkle proof for allowlist
MultiSalePurchase memory purchase = MultiSalePurchase({
    multiSaleId: saleId,
    purchaser: msg.sender,
    receiver: msg.sender,
    quantity: 2
});

MultiSaleProof memory proof = MultiSaleProof({
    proof: merkleProof,
    data: abi.encode(maxAllowedQuantity)
});

uint256[] memory tokenIds = IMultiSale(diamond).purchaseProof{value: 0.2 ether}(
    purchase,
    proof
);
```

### Query Functions

#### `getTokenSaleSettings()`
```solidity
function getTokenSaleSettings(uint256 tokenSaleId) external view virtual returns (MultiSaleSettings memory settings)
```

**Purpose**: Retrieves complete configuration for a specific token sale.

**Parameters**:
- `tokenSaleId` (uint256): ID of the token sale

**Returns**:
- `settings` (MultiSaleSettings): Complete sale configuration

**Example Usage**:
```solidity
MultiSaleSettings memory saleConfig = IMultiSale(diamond).getTokenSaleSettings(saleId);
console.log("Sale price:", saleConfig.price.price);
console.log("Max per account:", saleConfig.maxQuantityPerAccount);
```

---

#### `getTokenSaleIds()`
```solidity
function getTokenSaleIds() external view returns (uint256[] memory)
```

**Purpose**: Returns all token sale IDs in the system.

**Returns**: Array of token sale IDs

**Example Usage**:
```solidity
uint256[] memory allSaleIds = IMultiSale(diamond).getTokenSaleIds();
console.log("Total sales:", allSaleIds.length);
```

---

#### `listTokenSales()`
```solidity
function listTokenSales() external view returns (MultiSaleSettings[] memory)
```

**Purpose**: Returns complete settings for all token sales.

**Returns**: Array of MultiSaleSettings for all sales

**Example Usage**:
```solidity
MultiSaleSettings[] memory allSales = IMultiSale(diamond).listTokenSales();

for (uint256 i = 0; i < allSales.length; i++) {
    console.log("Sale", i, "price:", allSales[i].price.price);
}
```

## Integration Examples

### NFT Collection Sale
```solidity
// Create and manage NFT collection sale
contract NFTCollectionSale {
    function createNFTSale(
        address nftContract,
        uint256 pricePerNFT,
        uint256 maxPerWallet
    ) external onlyOwner returns (uint256) {
        MultiSaleSettings memory settings = MultiSaleSettings({
            token: nftContract,
            tokenType: TokenType.ERC721,
            owner: msg.sender,
            price: VariablePriceContract({
                price: pricePerNFT,
                priceType: PriceType.FIXED
            }),
            paymentMethod: PaymentMethod.ETH,
            paymentToken: address(0),
            maxQuantityPerAccount: maxPerWallet
        });
        
        return IMultiSale(diamond).createTokenSale(settings);
    }
    
    function purchaseNFTs(uint256 saleId, uint256 quantity) external payable {
        uint256[] memory tokenIds = IMultiSale(diamond).purchase{value: msg.value}(
            saleId,
            msg.sender,
            msg.sender,
            quantity,
            ""
        );
        
        emit NFTsPurchased(msg.sender, tokenIds);
    }
}
```

### Allowlist Presale
```solidity
// Implement allowlist-based presale
contract AllowlistPresale {
    bytes32 public merkleRoot;
    
    function setAllowlist(bytes32 _merkleRoot) external onlyOwner {
        merkleRoot = _merkleRoot;
    }
    
    function purchaseWithAllowlist(
        uint256 saleId,
        uint256 quantity,
        uint256 maxAllowed,
        bytes32[] calldata merkleProof
    ) external payable {
        // Verify Merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, maxAllowed));
        require(MerkleProof.verify(merkleProof, merkleRoot, leaf), "Invalid proof");
        
        MultiSalePurchase memory purchase = MultiSalePurchase({
            multiSaleId: saleId,
            purchaser: msg.sender,
            receiver: msg.sender,
            quantity: quantity
        });
        
        MultiSaleProof memory proof = MultiSaleProof({
            proof: abi.encode(merkleProof),
            data: abi.encode(maxAllowed)
        });
        
        uint256[] memory tokenIds = IMultiSale(diamond).purchaseProof{value: msg.value}(
            purchase,
            proof
        );
        
        emit AllowlistPurchase(msg.sender, tokenIds, quantity);
    }
}
```

### Multi-Token Marketplace
```solidity
// Support multiple token types in one marketplace
contract MultiTokenMarketplace {
    function createERC20Sale(
        address tokenContract,
        uint256 tokensPerETH,
        address paymentToken
    ) external returns (uint256) {
        MultiSaleSettings memory settings = MultiSaleSettings({
            token: tokenContract,
            tokenType: TokenType.ERC20,
            owner: msg.sender,
            price: VariablePriceContract({
                price: 1 ether / tokensPerETH,
                priceType: PriceType.FIXED
            }),
            paymentMethod: PaymentMethod.ERC20,
            paymentToken: paymentToken,
            maxQuantityPerAccount: 0  // Unlimited
        });
        
        return IMultiSale(diamond).createTokenSale(settings);
    }
    
    function createERC1155Sale(
        address tokenContract,
        uint256 pricePerToken,
        uint256 maxPerAccount
    ) external returns (uint256) {
        MultiSaleSettings memory settings = MultiSaleSettings({
            token: tokenContract,
            tokenType: TokenType.ERC1155,
            owner: msg.sender,
            price: VariablePriceContract({
                price: pricePerToken,
                priceType: PriceType.FIXED
            }),
            paymentMethod: PaymentMethod.ETH,
            paymentToken: address(0),
            maxQuantityPerAccount: maxPerAccount
        });
        
        return IMultiSale(diamond).createTokenSale(settings);
    }
}
```

### Dynamic Pricing Sale
```solidity
// Implement dynamic pricing mechanisms
contract DynamicPricingSale {
    function createTimedSale(
        address tokenContract,
        uint256 startPrice,
        uint256 endPrice,
        uint256 duration
    ) external returns (uint256) {
        MultiSaleSettings memory settings = MultiSaleSettings({
            token: tokenContract,
            tokenType: TokenType.ERC721,
            owner: msg.sender,
            price: VariablePriceContract({
                price: startPrice,
                priceType: PriceType.LINEAR_DECREASE,
                startTime: block.timestamp,
                endTime: block.timestamp + duration,
                endPrice: endPrice
            }),
            paymentMethod: PaymentMethod.ETH,
            paymentToken: address(0),
            maxQuantityPerAccount: 10
        });
        
        return IMultiSale(diamond).createTokenSale(settings);
    }
    
    function getCurrentPrice(uint256 saleId) external view returns (uint256) {
        MultiSaleSettings memory settings = IMultiSale(diamond).getTokenSaleSettings(saleId);
        return VariablePriceLib.calculateCurrentPrice(settings.price);
    }
}
```

### Batch Purchase System
```solidity
// Enable batch purchases across multiple sales
contract BatchPurchaseSystem {
    struct BatchPurchase {
        uint256 saleId;
        uint256 quantity;
        bytes data;
    }
    
    function batchPurchase(BatchPurchase[] calldata purchases) external payable {
        uint256 totalValue = 0;
        
        for (uint256 i = 0; i < purchases.length; i++) {
            MultiSaleSettings memory settings = IMultiSale(diamond).getTokenSaleSettings(purchases[i].saleId);
            uint256 purchaseValue = settings.price.price * purchases[i].quantity;
            
            uint256[] memory tokenIds = IMultiSale(diamond).purchase{value: purchaseValue}(
                purchases[i].saleId,
                msg.sender,
                msg.sender,
                purchases[i].quantity,
                purchases[i].data
            );
            
            totalValue += purchaseValue;
            emit BatchPurchaseItem(purchases[i].saleId, tokenIds);
        }
        
        // Refund any excess ETH
        if (msg.value > totalValue) {
            (bool success, ) = payable(msg.sender).call{value: msg.value - totalValue}("");
            require(success, "Refund failed");
        }
    }
}
```

## Events

### Sale Management Events
```solidity
event MultiSaleCreated(uint256 indexed tokenSaleId, MultiSaleSettings settings);
event MultiSaleUpdated(uint256 indexed tokenSaleId, MultiSaleSettings settings);
```

### Purchase Events
```solidity
event MultiSaleSold(
    uint256 indexed multiSaleId,
    address indexed purchaser,
    uint256[] tokenIds,
    bytes data
);
```

## Security Considerations

### Reentrancy Protection
- All payment functions use `nonReentrant` modifier
- Secure ETH and ERC20 token handling
- Automatic excess ETH refunds

### Access Control
- Owner-only sale creation and updates
- Sale owner permissions for configuration changes
- Purchaser validation and limits

### Payment Security
- Precise payment amount validation
- Automatic excess payment refunds
- Secure token transfer mechanisms

### Proof Validation
- Cryptographic proof verification for allowlists
- Merkle proof support for presales
- Custom proof mechanisms for advanced use cases

## Gas Optimization

### Efficient Minting
- Batch minting for multiple tokens
- Optimized token ID generation
- Minimal storage operations

### Payment Processing
- Single transaction for purchase and minting
- Efficient excess ETH handling
- Optimized storage updates

## Error Handling

### Purchase Errors
- `"Insufficient ETH sent"` - Payment amount validation
- `"maxperaccount"` - Purchase limit enforcement
- `"Token not supported"` - Invalid token type

### Access Errors
- `"notowner"` - Unauthorized sale updates
- Proof validation failures
- Payment method mismatches

## Testing Considerations

### Unit Tests
- Sale creation and configuration
- Purchase flows for all token types
- Payment processing and refunds
- Access control validation

### Integration Tests
- Multi-token sale scenarios
- Allowlist and proof verification
- Dynamic pricing mechanisms
- Batch operation efficiency

## Related Documentation

- [IMultiSale](../interfaces/imulti-sale.md) - Multi-sale interface
- [MultiSaleLib](../libraries/multi-sale-lib.md) - Multi-sale utilities
- [VariablePriceLib](../libraries/variable-price-lib.md) - Pricing mechanisms
- [MarketplaceFacet](./marketplace-facet.md) - Marketplace integration
- [Token Sale Guide](../../guides/token-sales.md) - Implementation guide

---

*This facet provides comprehensive multi-token sale functionality within the Gemforce platform, supporting diverse token standards and advanced sale mechanisms for flexible marketplace operations.*