# IERC721A Interface

The IERC721A interface extends the standard ERC721 interface with gas-optimized batch minting capabilities. This interface is designed for projects that need to mint large quantities of NFTs efficiently while maintaining full ERC721 compatibility.

## Overview

IERC721A provides:

- **Gas-Optimized Minting**: Significantly reduced gas costs for batch minting
- **ERC721 Compatibility**: Full compatibility with existing ERC721 infrastructure
- **Sequential Token IDs**: Automatic sequential token ID assignment
- **Batch Operations**: Efficient batch minting and transfers
- **Enumeration Support**: Optional enumeration capabilities

## Key Features

### Gas Optimization
- **Batch Minting**: Mint multiple tokens in a single transaction
- **Storage Optimization**: Optimized storage layout for reduced gas costs
- **Lazy Initialization**: Deferred initialization of token data
- **Packed Storage**: Efficient packing of token information

### ERC721 Compatibility
- **Standard Compliance**: Full ERC721 standard compliance
- **Marketplace Support**: Compatible with all major NFT marketplaces
- **Wallet Support**: Works with all ERC721-compatible wallets
- **Tool Integration**: Compatible with existing NFT tools and services

## Interface Definition

```solidity
interface IERC721A {
    // ERC721 Standard Events
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    // ERC721A Specific Events
    event ConsecutiveTransfer(uint256 indexed fromTokenId, uint256 toTokenId, address indexed from, address indexed to);
    
    // ERC721 Standard Functions
    function balanceOf(address owner) external view returns (uint256 balance);
    function ownerOf(uint256 tokenId) external view returns (address owner);
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address operator);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    
    // ERC721Metadata
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function tokenURI(uint256 tokenId) external view returns (string memory);
    
    // ERC721A Specific Functions
    function totalSupply() external view returns (uint256);
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
    
    // Batch Operations
    function safeMint(address to, uint256 quantity) external;
    function safeMint(address to, uint256 quantity, bytes memory data) external;
    
    // Advanced Queries
    function numberMinted(address owner) external view returns (uint256);
    function numberBurned(address owner) external view returns (uint256);
    function getAux(address owner) external view returns (uint64);
    function setAux(address owner, uint64 aux) external;
    
    // Token Existence
    function exists(uint256 tokenId) external view returns (bool);
    
    // Burning
    function burn(uint256 tokenId) external;
}
```

## Core Functions

### Standard ERC721 Functions

#### balanceOf()
Returns the number of tokens owned by an address.

**Parameters:**
- `owner`: Address to query the balance of

**Returns:**
- `uint256`: Number of tokens owned

#### ownerOf()
Returns the owner of a specific token.

**Parameters:**
- `tokenId`: Token ID to query

**Returns:**
- `address`: Owner of the token

#### transferFrom()
Transfers a token from one address to another.

**Parameters:**
- `from`: Current owner of the token
- `to`: Address to transfer the token to
- `tokenId`: Token ID to transfer

### ERC721A Specific Functions

#### safeMint()
Mints tokens to a specified address with gas optimization.

**Parameters:**
- `to`: Address to mint tokens to
- `quantity`: Number of tokens to mint
- `data`: Optional data to pass to receiver (if contract)

**Gas Optimization:**
- Batch minting reduces gas cost per token significantly
- Sequential token IDs eliminate need for individual storage

#### numberMinted()
Returns the total number of tokens minted by an address.

**Parameters:**
- `owner`: Address to query

**Returns:**
- `uint256`: Total number of tokens minted

#### numberBurned()
Returns the total number of tokens burned by an address.

**Parameters:**
- `owner`: Address to query

**Returns:**
- `uint256`: Total number of tokens burned

#### getAux() / setAux()
Get/set auxiliary data for an address (64 bits of custom data).

**Parameters:**
- `owner`: Address to query/modify
- `aux`: Auxiliary data to set (setAux only)

**Returns:**
- `uint64`: Auxiliary data (getAux only)

## Implementation Example

### Basic ERC721A Contract

```solidity
contract MyNFTCollection is IERC721A {
    string private _name;
    string private _symbol;
    string private _baseTokenURI;
    uint256 private _currentIndex;
    uint256 private _burnCounter;
    
    // Mapping from token ID to ownership details
    mapping(uint256 => TokenOwnership) private _ownerships;
    
    // Mapping owner address to address data
    mapping(address => AddressData) private _addressData;
    
    // Mapping from token ID to approved address
    mapping(uint256 => address) private _tokenApprovals;
    
    // Mapping from owner to operator approvals
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    
    struct TokenOwnership {
        address addr;
        uint64 startTimestamp;
        bool burned;
        uint24 extraData;
    }
    
    struct AddressData {
        uint64 balance;
        uint64 numberMinted;
        uint64 numberBurned;
        uint64 aux;
    }
    
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
        _currentIndex = 1; // Start token IDs at 1
    }
    
    function safeMint(address to, uint256 quantity) external override {
        require(to != address(0), "ERC721A: mint to zero address");
        require(quantity > 0, "ERC721A: quantity must be greater than 0");
        
        uint256 startTokenId = _currentIndex;
        _currentIndex += quantity;
        
        // Update balance and minted count
        _addressData[to].balance += uint64(quantity);
        _addressData[to].numberMinted += uint64(quantity);
        
        // Set ownership for the first token in the batch
        _ownerships[startTokenId] = TokenOwnership(
            to,
            uint64(block.timestamp),
            false,
            0
        );
        
        // Emit Transfer events
        for (uint256 i = 0; i < quantity; i++) {
            emit Transfer(address(0), to, startTokenId + i);
        }
        
        // Emit ConsecutiveTransfer event for gas optimization
        if (quantity > 1) {
            emit ConsecutiveTransfer(startTokenId, startTokenId + quantity - 1, address(0), to);
        }
    }
    
    function ownerOf(uint256 tokenId) public view override returns (address) {
        require(_exists(tokenId), "ERC721A: owner query for nonexistent token");
        
        uint256 curr = tokenId;
        while (true) {
            TokenOwnership memory ownership = _ownerships[curr];
            if (ownership.addr != address(0)) {
                return ownership.addr;
            }
            curr--;
        }
        
        revert("ERC721A: unable to determine the owner of token");
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        return tokenId > 0 && tokenId < _currentIndex && !_ownerships[tokenId].burned;
    }
}
```

### Advanced Batch Minting

```solidity
contract AdvancedERC721A is IERC721A {
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant MAX_BATCH_SIZE = 20;
    uint256 public mintPrice = 0.01 ether;
    
    mapping(address => bool) public whitelist;
    bool public whitelistActive = true;
    
    function publicMint(uint256 quantity) external payable {
        require(!whitelistActive, "Whitelist phase active");
        require(quantity <= MAX_BATCH_SIZE, "Exceeds max batch size");
        require(totalSupply() + quantity <= MAX_SUPPLY, "Exceeds max supply");
        require(msg.value >= mintPrice * quantity, "Insufficient payment");
        
        _safeMint(msg.sender, quantity);
    }
    
    function whitelistMint(uint256 quantity) external payable {
        require(whitelistActive, "Whitelist phase not active");
        require(whitelist[msg.sender], "Not whitelisted");
        require(quantity <= MAX_BATCH_SIZE, "Exceeds max batch size");
        require(totalSupply() + quantity <= MAX_SUPPLY, "Exceeds max supply");
        require(msg.value >= mintPrice * quantity, "Insufficient payment");
        
        _safeMint(msg.sender, quantity);
    }
    
    function airdrop(address[] calldata recipients, uint256[] calldata quantities) external onlyOwner {
        require(recipients.length == quantities.length, "Arrays length mismatch");
        
        for (uint256 i = 0; i < recipients.length; i++) {
            require(totalSupply() + quantities[i] <= MAX_SUPPLY, "Exceeds max supply");
            _safeMint(recipients[i], quantities[i]);
        }
    }
}
```

## Gas Optimization Benefits

### Comparison with Standard ERC721

| Operation | ERC721 Gas Cost | ERC721A Gas Cost | Savings |
|-----------|----------------|------------------|---------|
| Mint 1 NFT | ~51,000 | ~51,000 | 0% |
| Mint 5 NFTs | ~255,000 | ~56,000 | 78% |
| Mint 10 NFTs | ~510,000 | ~61,000 | 88% |
| Mint 20 NFTs | ~1,020,000 | ~71,000 | 93% |

### Storage Optimization

```solidity
// Standard ERC721 - stores owner for each token
mapping(uint256 => address) private _owners; // 20k gas per token

// ERC721A - stores owner only for first token in batch
mapping(uint256 => TokenOwnership) private _ownerships; // 20k gas per batch
```

## Integration Patterns

### Marketplace Integration

```solidity
contract MarketplaceIntegration {
    IERC721A public nftContract;
    
    function listToken(uint256 tokenId, uint256 price) external {
        require(nftContract.ownerOf(tokenId) == msg.sender, "Not token owner");
        require(nftContract.getApproved(tokenId) == address(this) || 
                nftContract.isApprovedForAll(msg.sender, address(this)), "Not approved");
        
        // List token for sale
        _createListing(tokenId, price, msg.sender);
    }
    
    function buyToken(uint256 tokenId) external payable {
        Listing memory listing = listings[tokenId];
        require(listing.active, "Token not for sale");
        require(msg.value >= listing.price, "Insufficient payment");
        
        // Transfer token
        nftContract.safeTransferFrom(listing.seller, msg.sender, tokenId);
        
        // Handle payment
        _handlePayment(listing.seller, listing.price);
    }
}
```

### Staking Integration

```solidity
contract NFTStaking {
    IERC721A public nftContract;
    mapping(uint252 => StakeInfo) public stakes;
    
    struct StakeInfo {
        address owner;
        uint256 stakedAt;
        uint256 rewards;
    }
    
    function stake(uint256[] calldata tokenIds) external {
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 tokenId = tokenIds[i];
            require(nftContract.ownerOf(tokenId) == msg.sender, "Not token owner");
            
            // Transfer to staking contract
            nftContract.safeTransferFrom(msg.sender, address(this), tokenId);
            
            stakes[tokenId] = StakeInfo({
                owner: msg.sender,
                stakedAt: block.timestamp,
                rewards: 0
            });
        }
    }
    
    function unstake(uint256[] calldata tokenIds) external {
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 tokenId = tokenIds[i];
            StakeInfo storage stake = stakes[tokenId];
            require(stake.owner == msg.sender, "Not stake owner");
            
            // Calculate and distribute rewards
            uint256 rewards = calculateRewards(tokenId);
            _distributeRewards(msg.sender, rewards);
            
            // Return NFT
            nftContract.safeTransferFrom(address(this), msg.sender, tokenId);
            delete stakes[tokenId];
        }
    }
}
```

## Security Considerations

### Reentrancy Protection
- Use reentrancy guards for minting functions
- Validate external calls in safe transfer hooks
- Implement proper access controls

### Integer Overflow Protection
- Use SafeMath or Solidity 0.8+ built-in overflow protection
- Validate quantity parameters in minting functions
- Check total supply limits

### Access Control
- Implement proper role-based access control
- Validate ownership before transfers
- Secure administrative functions

## Best Practices

### Minting Guidelines
1. **Batch Size Limits**: Implement reasonable batch size limits
2. **Supply Caps**: Enforce maximum supply limits
3. **Payment Validation**: Validate payment amounts for paid mints
4. **Whitelist Management**: Secure whitelist functionality

### Gas Optimization
1. **Batch Operations**: Encourage batch minting over individual mints
2. **Storage Packing**: Pack related data into single storage slots
3. **Lazy Loading**: Defer expensive operations when possible
4. **Event Optimization**: Use ConsecutiveTransfer events for batch operations

### Integration Considerations
1. **Marketplace Compatibility**: Ensure compatibility with major marketplaces
2. **Wallet Support**: Test with popular wallet implementations
3. **Metadata Standards**: Follow metadata standards for better compatibility
4. **Enumeration Support**: Consider implementing enumeration for better tooling support

## Related Documentation

- [ERC721A Library](../libraries/erc721a-lib.md)
- [ERC721A Enumeration Library](../libraries/erc721a-enumeration-lib.md)
- [Metadata Library](../libraries/metadata-lib.md)
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)
- [Developer Guides: Performance Optimization](../../developer-guides/performance-optimization.md)

## Standards Compliance

- **ERC721**: Full ERC721 standard compliance
- **ERC721Metadata**: Metadata extension support
- **ERC721Enumerable**: Optional enumeration support
- **ERC165**: Interface detection support
- **Gas Optimization**: Optimized for batch operations