# CarbonCreditLib Library

## Overview

The [`CarbonCreditLib`](../../../contracts/libraries/CarbonCreditLib.sol) library provides core utilities and data structures for managing carbon credits associated with ERC721 tokens within the Gemforce platform. This library implements the Diamond Standard storage pattern and provides essential functions for carbon credit initialization, retirement, and status management.

## Key Features

- **Diamond Storage Pattern**: Secure storage isolation using Diamond Standard
- **Token Balance Management**: Track carbon credit balances per ERC721 token
- **Credit Retirement**: Permanent retirement of carbon credits to prevent double-counting
- **Status Tracking**: Monitor active vs. retired carbon credit status
- **Token Validation**: Ensure token existence before operations
- **Gas Optimization**: Efficient storage layout and operations

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

enum CarbonCreditStatus {
    ACTIVE,
    RETIRED
}

import "@openzeppelin/contracts/interfaces/IERC721.sol";

library CarbonCreditLib {
    struct CarbonCreditStorage {
        mapping(uint256 => uint256) tokenBalances;
    }

    bytes32 constant CARBON_CREDIT_STORAGE_POSITION = keccak256("diamond.standard.carbon.credit.storage");

    function carbonCreditStorage() internal pure returns (CarbonCreditStorage storage cs);
    function _tokenExists(uint256 tokenId) internal view returns (bool);
    function initializeBalance(CarbonCreditStorage storage self, uint256 tokenId, uint256 initialBalance) internal;
    function retireCredits(CarbonCreditStorage storage self, uint256 tokenId, uint256 amount) internal;
    function getBalance(CarbonCreditStorage storage self, uint256 tokenId) internal view returns (uint256);
    function getCarbonCreditStatus(CarbonCreditStorage storage self, uint256 tokenId) internal view returns (CarbonCreditStatus);
}
```

## Data Structures

### CarbonCreditStatus Enum
```solidity
enum CarbonCreditStatus {
    ACTIVE,    // Token has remaining carbon credits
    RETIRED    // All carbon credits have been retired
}
```

**Purpose**: Represents the current status of carbon credits for a token.

**Values**:
- `ACTIVE` (0): Token has remaining carbon credits available
- `RETIRED` (1): All carbon credits have been permanently retired

### CarbonCreditStorage Struct
```solidity
struct CarbonCreditStorage {
    mapping(uint256 => uint256) tokenBalances;
}
```

**Purpose**: Diamond storage structure for carbon credit data.

**Fields**:
- `tokenBalances`: Mapping from token ID to carbon credit balance

## Storage Management

### `carbonCreditStorage()`
```solidity
function carbonCreditStorage() internal pure returns (CarbonCreditStorage storage cs)
```

**Purpose**: Access the Diamond storage slot for carbon credit data.

**Returns**: Storage reference to the carbon credit storage structure

**Implementation**:
```solidity
function carbonCreditStorage() internal pure returns (CarbonCreditStorage storage cs) {
    bytes32 position = CARBON_CREDIT_STORAGE_POSITION;
    assembly {
        cs.slot := position
    }
}
```

**Storage Position**: `keccak256("diamond.standard.carbon.credit.storage")`

**Usage Example**:
```solidity
// Access carbon credit storage in a facet
CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
uint256 balance = cs.tokenBalances[tokenId];
```

## Core Functions

### Token Validation

#### `_tokenExists()`
```solidity
function _tokenExists(uint256 tokenId) internal view returns (bool)
```

**Purpose**: Verify that an ERC721 token exists by checking if it has an owner.

**Parameters**:
- `tokenId` (uint256): The ID of the token to check

**Returns**: Boolean indicating whether the token exists

**Implementation Details**:
- Uses try-catch to safely call `ownerOf()`
- Returns `false` if the call reverts (token doesn't exist)
- Returns `true` if owner is not zero address

**Example Usage**:
```solidity
// Check if token exists before operations
uint256 tokenId = 1;
bool exists = CarbonCreditLib._tokenExists(tokenId);
require(exists, "Token does not exist");
```

### Balance Management

#### `initializeBalance()`
```solidity
function initializeBalance(CarbonCreditStorage storage self, uint256 tokenId, uint256 initialBalance) internal
```

**Purpose**: Initialize carbon credit balance for a specific ERC721 token.

**Parameters**:
- `self` (CarbonCreditStorage storage): Storage reference
- `tokenId` (uint256): The ID of the ERC721 token
- `initialBalance` (uint256): The initial carbon credit balance

**Requirements**:
- Token must not already have carbon credits initialized
- Token must exist (have a valid owner)
- Initial balance must be greater than zero

**Validation Logic**:
```solidity
require(self.tokenBalances[tokenId] == 0, "Carbon credits already initialized");
require(_tokenExists(tokenId), "Token does not exist");
require(initialBalance > 0, "Initial balance must be greater than zero");
```

**Example Usage**:
```solidity
// Initialize carbon credits for a newly minted environmental NFT
CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
uint256 tokenId = 1;
uint256 initialCredits = 1000;

CarbonCreditLib.initializeBalance(cs, tokenId, initialCredits);
```

#### `retireCredits()`
```solidity
function retireCredits(CarbonCreditStorage storage self, uint256 tokenId, uint256 amount) internal
```

**Purpose**: Permanently retire a specified amount of carbon credits from a token.

**Parameters**:
- `self` (CarbonCreditStorage storage): Storage reference
- `tokenId` (uint256): The ID of the ERC721 token
- `amount` (uint256): The amount of carbon credits to retire

**Requirements**:
- Amount must be greater than zero
- Token must have sufficient carbon credit balance
- Retirement is permanent and irreversible

**Validation Logic**:
```solidity
require(amount > 0, "Amount must be greater than zero");
require(self.tokenBalances[tokenId] >= amount, "Insufficient balance");
```

**Example Usage**:
```solidity
// Retire carbon credits to offset emissions
CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
uint256 tokenId = 1;
uint256 retireAmount = 250;

CarbonCreditLib.retireCredits(cs, tokenId, retireAmount);
```

### Query Functions

#### `getBalance()`
```solidity
function getBalance(CarbonCreditStorage storage self, uint256 tokenId) internal view returns (uint256)
```

**Purpose**: Get the current carbon credit balance for a specific token.

**Parameters**:
- `self` (CarbonCreditStorage storage): Storage reference
- `tokenId` (uint256): The ID of the ERC721 token

**Returns**: Current carbon credit balance for the token

**Example Usage**:
```solidity
// Check current carbon credit balance
CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
uint256 tokenId = 1;
uint256 balance = CarbonCreditLib.getBalance(cs, tokenId);
```

#### `getCarbonCreditStatus()`
```solidity
function getCarbonCreditStatus(CarbonCreditStorage storage self, uint256 tokenId) internal view returns (CarbonCreditStatus)
```

**Purpose**: Get the current status of carbon credits for a token.

**Parameters**:
- `self` (CarbonCreditStorage storage): Storage reference
- `tokenId` (uint256): The ID of the ERC721 token

**Returns**: [`CarbonCreditStatus`](carbon-credit-lib.md:4) enum value

**Logic**:
- Returns `RETIRED` if balance is zero
- Returns `ACTIVE` if balance is greater than zero

**Example Usage**:
```solidity
// Check carbon credit status
CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
uint256 tokenId = 1;
CarbonCreditStatus status = CarbonCreditLib.getCarbonCreditStatus(cs, tokenId);

if (status == CarbonCreditStatus.ACTIVE) {
    // Token has active carbon credits
} else {
    // All credits have been retired
}
```

## Integration Examples

### Environmental NFT Contract
```solidity
// Environmental NFT contract using CarbonCreditLib
contract EnvironmentalNFT is ERC721, IDiamondCut {
    using CarbonCreditLib for CarbonCreditLib.CarbonCreditStorage;
    
    event CarbonCreditsInitialized(uint256 indexed tokenId, uint256 initialBalance);
    event CarbonCreditsRetired(uint256 indexed tokenId, uint256 amount, uint256 remainingBalance);
    
    function mintWithCarbonCredits(
        address to,
        uint256 tokenId,
        uint256 carbonCredits,
        string memory uri
    ) external onlyMinter {
        // Mint the NFT
        _mint(to, tokenId);
        _setTokenURI(tokenId, uri);
        
        // Initialize carbon credits
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        CarbonCreditLib.initializeBalance(cs, tokenId, carbonCredits);
        
        emit CarbonCreditsInitialized(tokenId, carbonCredits);
    }
    
    function retireCarbonCredits(uint256 tokenId, uint256 amount) external {
        require(ownerOf(tokenId) == msg.sender, "Not token owner");
        
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        uint256 balanceBefore = CarbonCreditLib.getBalance(cs, tokenId);
        
        CarbonCreditLib.retireCredits(cs, tokenId, amount);
        
        uint256 balanceAfter = CarbonCreditLib.getBalance(cs, tokenId);
        emit CarbonCreditsRetired(tokenId, amount, balanceAfter);
    }
    
    function getCarbonCreditInfo(uint256 tokenId) external view returns (
        uint256 balance,
        CarbonCreditStatus status
    ) {
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        balance = CarbonCreditLib.getBalance(cs, tokenId);
        status = CarbonCreditLib.getCarbonCreditStatus(cs, tokenId);
    }
    
    modifier onlyMinter() {
        // Implementation would check for minter role
        _;
    }
}
```

### Carbon Credit Marketplace
```solidity
// Marketplace for trading environmental NFTs with carbon credits
contract CarbonCreditMarketplace {
    using CarbonCreditLib for CarbonCreditLib.CarbonCreditStorage;
    
    struct Listing {
        uint256 tokenId;
        address seller;
        uint256 price;
        uint256 carbonCredits;
        bool active;
    }
    
    mapping(uint256 => Listing) public listings;
    uint256 public nextListingId;
    
    event NFTListed(uint256 indexed listingId, uint256 indexed tokenId, uint256 carbonCredits, uint256 price);
    event NFTPurchased(uint256 indexed listingId, address indexed buyer, uint256 creditsRetired);
    
    function listNFT(
        uint256 tokenId,
        uint256 price,
        address nftContract
    ) external returns (uint256 listingId) {
        require(IERC721(nftContract).ownerOf(tokenId) == msg.sender, "Not token owner");
        
        // Get carbon credit information
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        uint256 carbonCredits = CarbonCreditLib.getBalance(cs, tokenId);
        require(carbonCredits > 0, "No carbon credits associated");
        
        listingId = nextListingId++;
        listings[listingId] = Listing({
            tokenId: tokenId,
            seller: msg.sender,
            price: price,
            carbonCredits: carbonCredits,
            active: true
        });
        
        emit NFTListed(listingId, tokenId, carbonCredits, price);
    }
    
    function purchaseNFT(
        uint256 listingId,
        uint256 creditsToRetire,
        address nftContract
    ) external payable {
        Listing storage listing = listings[listingId];
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        uint256 currentCredits = CarbonCreditLib.getBalance(cs, listing.tokenId);
        require(creditsToRetire <= currentCredits, "Cannot retire more credits than available");
        
        // Transfer NFT
        IERC721(nftContract).safeTransferFrom(listing.seller, msg.sender, listing.tokenId);
        
        // Retire carbon credits if requested
        if (creditsToRetire > 0) {
            CarbonCreditLib.retireCredits(cs, listing.tokenId, creditsToRetire);
        }
        
        // Transfer payment
        payable(listing.seller).transfer(msg.value);
        
        listing.active = false;
        emit NFTPurchased(listingId, msg.sender, creditsToRetire);
    }
    
    function getListingCarbonInfo(uint256 listingId) external view returns (
        uint256 currentBalance,
        CarbonCreditStatus status,
        uint256 listedCredits
    ) {
        Listing memory listing = listings[listingId];
        
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        currentBalance = CarbonCreditLib.getBalance(cs, listing.tokenId);
        status = CarbonCreditLib.getCarbonCreditStatus(cs, listing.tokenId);
        listedCredits = listing.carbonCredits;
    }
}
```

### Carbon Offset Tracking System
```solidity
// System for tracking carbon offset activities
contract CarbonOffsetTracker {
    using CarbonCreditLib for CarbonCreditLib.CarbonCreditStorage;
    
    struct OffsetRecord {
        address offsetter;
        uint256 tokenId;
        uint256 amount;
        uint256 timestamp;
        string purpose;
    }
    
    mapping(uint256 => OffsetRecord) public offsetRecords;
    mapping(address => uint256[]) public userOffsets;
    uint256 public nextRecordId;
    
    event CarbonOffset(
        uint256 indexed recordId,
        address indexed offsetter,
        uint256 indexed tokenId,
        uint256 amount,
        string purpose
    );
    
    function recordCarbonOffset(
        uint256 tokenId,
        uint256 amount,
        string memory purpose,
        address nftContract
    ) external returns (uint256 recordId) {
        require(IERC721(nftContract).ownerOf(tokenId) == msg.sender, "Not token owner");
        
        CarbonCreditLib.CarbonCreditStorage storage cs = CarbonCreditLib.carbonCreditStorage();
        
        // Verify sufficient balance
        uint256 currentBalance = CarbonCreditLib.getBalance(cs, tokenId);
        require(currentBalance >= amount, "Insufficient carbon credits");
        
        // Retire the credits
        CarbonCreditLib.retireCredits(cs, tokenId, amount);
        
        // Record the offset
        recordId = nextRecordId++;
        offsetRecords[recordId] = OffsetRecord({
            offsetter: msg.sender,
            tokenId: tokenId,
            amount: amount,
            timestamp: block.timestamp,
            purpose: purpose
        });
        
        userOffsets[msg.sender].push(recordId);
        
        emit CarbonOffset(recordId, msg.sender, tokenId, amount, purpose);
    }
    
    function getUserOffsetSummary(address user) external view returns (
        uint256 totalOffsetsCount,
        uint256 totalCreditsRetired,
        uint256[] memory recordIds
    ) {
        recordIds = userOffsets[user];
        totalOffsetsCount = recordIds.length;
        
        for (uint256 i = 0; i < recordIds.length; i++) {
            totalCreditsRetired += offsetRecords[recordIds[i]].amount;
        }
    }
    
    function getTokenOffsetHistory(uint256 tokenId) external view returns (
        uint256[] memory recordIds,
        uint256 totalRetired
    ) {
        // Find all offset records for this token
        uint256 count = 0;
        for (uint256 i = 0; i < nextRecordId; i++) {
            if (offsetRecords[i].tokenId == tokenId) {
                count++;
            }
        }
        
        recordIds = new uint256[](count);
        uint256 index = 0;
        
        for (uint256 i = 0; i < nextRecordId; i++) {
            if (offsetRecords[i].tokenId == tokenId) {
                recordIds[index] = i;
                totalRetired += offsetRecords[i].amount;
                index++;
            }
        }
    }
}
```

## Security Considerations

### Storage Security
- Uses Diamond Standard storage pattern for isolation
- Prevents storage collisions with unique storage position
- Secure access control through internal functions only

### Validation Security
- Token existence validation before operations
- Balance validation before retirement
- Prevention of double initialization
- Overflow protection in balance calculations

### Operational Security
- Immutable retirement operations
- Consistent state management
- Safe external contract interactions
- Error handling for edge cases

## Gas Optimization

### Storage Efficiency
- Minimal storage footprint with single mapping
- Efficient storage slot usage
- Optimized for Diamond Standard pattern

### Function Efficiency
- Internal functions for gas savings
- Minimal external calls
- Efficient validation logic
- Optimized assembly for storage access

## Error Handling

### Common Errors
- "Carbon credits already initialized" - Attempting to initialize twice
- "Token does not exist" - Operating on non-existent token
- "Initial balance must be greater than zero" - Invalid initialization
- "Amount must be greater than zero" - Invalid retirement amount
- "Insufficient balance" - Attempting to retire more than available

### Best Practices
- Always validate token existence before operations
- Check balances before retirement operations
- Handle initialization state properly
- Use appropriate error messages for debugging

## Testing Considerations

### Unit Tests
- Storage pattern implementation
- Token existence validation
- Balance initialization and management
- Credit retirement functionality
- Status determination logic

### Integration Tests
- Integration with ERC721 contracts
- Diamond facet integration
- Multi-token operations
- Edge case handling
- Gas usage optimization

## Related Documentation

- [ICarbonCredit Interface](../interfaces/icarbon-credit.md) - Carbon credit interface definition
- [CarbonCreditFacet](../facets/carbon-credit-facet.md) - Carbon credit facet implementation
- [Diamond Standard Guide](../../guides/diamond-standard.md) - Diamond pattern implementation
- [Environmental NFT Guide](../../guides/environmental-nfts.md) - Environmental asset tokenization
- [Carbon Offset Integration](../../guides/carbon-offset-integration.md) - Carbon offset program integration

---

*This library provides the core utilities and data structures for carbon credit management within the Gemforce platform, implementing secure storage patterns and essential operations for environmental asset tokenization.*