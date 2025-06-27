# CarbonCreditFacet

## Overview

The [`CarbonCreditFacet.sol`](../../smart-contracts/facets/carbon-credit-facet.md) provides comprehensive carbon credit management functionality for ERC721 tokens within the Gemforce diamond system. This facet enables the tokenization, tracking, and retirement of environmental assets, supporting carbon offset programs and environmental sustainability initiatives.

## Contract Details

- **Contract Name**: `CarbonCreditFacet`
- **Inheritance**: `ICarbonCredit`, `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Carbon Credit Lifecycle Management
- Initialize carbon credit balances for NFTs
- Track available credits per token
- Permanent retirement of credits for offsetting
- Comprehensive status tracking

### 🔹 ERC721 Integration
- Carbon credits attached to specific NFT tokens
- Owner-only retirement permissions
- Token existence validation
- Seamless integration with existing NFT infrastructure

### 🔹 Batch Operations
- Efficient batch initialization for multiple tokens
- Bulk balance queries
- Gas-optimized operations for large-scale deployments

### 🔹 Environmental Compliance
- Permanent retirement prevents double-counting
- Transparent tracking for audit purposes
- Status reporting for compliance verification

## Core Data Structures

### CarbonCreditStatus Enum
```solidity
enum CarbonCreditStatus {
    NONE,     // Token has no carbon credits initialized
    ACTIVE,   // Token has available carbon credits
    RETIRED   // All carbon credits have been retired
}
```

### Storage Pattern
The facet uses the CarbonCreditLib library for storage management:
- Diamond storage pattern prevents storage collisions
- Efficient mapping of token IDs to credit balances
- Persistent storage across contract upgrades

## Core Functions

### Initialization Functions

#### `initializeCarbonCredit()`
```solidity
function initializeCarbonCredit(uint256 tokenId, uint256 initialBalance) external onlyOwner
```

**Purpose**: Initializes carbon credit balance for a specific ERC721 token.

**Parameters**:
- `tokenId` (uint256): ID of the ERC721 token to initialize
- `initialBalance` (uint256): Initial carbon credit balance (must be > 0)

**Access Control**: Owner only

**Validation**:
- Token ID must be greater than zero
- Initial balance must be greater than zero
- Token must exist (have a valid owner)
- Token must not already have carbon credits initialized

**Process**:
1. Validates input parameters
2. Checks token existence via ERC721 interface
3. Ensures token hasn't been previously initialized
4. Sets initial carbon credit balance
5. Emits CarbonCreditsInitialized event

**Events**: `CarbonCreditsInitialized(tokenId, initialBalance)`

**Example Usage**:
```solidity
// Initialize carbon credits for a newly minted environmental NFT
ICarbonCredit(diamond).initializeCarbonCredit(
    tokenId,
    1000  // 1000 carbon credits
);
```

**Error Conditions**:
- `"CARBON_CREDIT: INVALID_TOKEN_ID"` - Token ID is zero
- `"CARBON_CREDIT: INVALID_BALANCE"` - Initial balance is zero
- `"CARBON_CREDIT: NONEXISTENT_TOKEN"` - Token doesn't exist
- `"CARBON_CREDIT: ALREADY_INITIALIZED"` - Token already has credits

---

#### `batchInitializeCarbonCredits()`
```solidity
function batchInitializeCarbonCredits(
    uint256[] calldata tokenIds, 
    uint256[] calldata initialBalances
) external onlyOwner
```

**Purpose**: Efficiently initializes carbon credits for multiple tokens in a single transaction.

**Parameters**:
- `tokenIds` (uint256[]): Array of token IDs to initialize
- `initialBalances` (uint256[]): Array of initial balances corresponding to token IDs

**Access Control**: Owner only

**Validation**:
- Arrays must have equal length
- Each token must exist
- Standard initialization validation for each token

**Gas Optimization**: More efficient than multiple individual calls

**Example Usage**:
```solidity
// Batch initialize multiple environmental NFTs
uint256[] memory tokenIds = [1, 2, 3, 4, 5];
uint256[] memory balances = [1000, 1500, 800, 1200, 900];

ICarbonCredit(diamond).batchInitializeCarbonCredits(tokenIds, balances);
```

### Credit Management Functions

#### `retireCarbonCredits()`
```solidity
function retireCarbonCredits(uint256 tokenId, uint256 amount) external onlyTokenOwner(tokenId)
```

**Purpose**: Permanently retires carbon credits, representing their use for carbon offsetting.

**Parameters**:
- `tokenId` (uint256): ID of the ERC721 token
- `amount` (uint256): Amount of carbon credits to retire

**Access Control**: Token owner only

**Validation**:
- Token ID must be greater than zero
- Amount must be greater than zero
- Token must have sufficient credit balance

**Process**:
1. Validates input parameters
2. Checks current credit balance
3. Ensures sufficient credits available
4. Permanently reduces credit balance
5. Emits retirement event with remaining balance

**Events**: `CarbonCreditsRetired(tokenId, amount, remainingBalance)`

**Example Usage**:
```solidity
// Retire 100 carbon credits for offsetting
ICarbonCredit(diamond).retireCarbonCredits(tokenId, 100);
```

**Important**: Retired credits cannot be recovered or reused.

### Query Functions

#### `getCarbonCreditBalance()`
```solidity
function getCarbonCreditBalance(uint256 tokenId) external view returns (uint256)
```

**Purpose**: Returns the current available carbon credit balance for a token.

**Parameters**:
- `tokenId` (uint256): ID of the ERC721 token

**Returns**: Current carbon credit balance (0 if not initialized)

**Example Usage**:
```solidity
uint256 balance = ICarbonCredit(diamond).getCarbonCreditBalance(tokenId);
console.log("Available credits:", balance);
```

---

#### `getCarbonCreditStatus()`
```solidity
function getCarbonCreditStatus(uint256 tokenId) external view returns (CarbonCreditStatus)
```

**Purpose**: Returns the carbon credit status for a token.

**Parameters**:
- `tokenId` (uint256): ID of the ERC721 token

**Returns**: CarbonCreditStatus enum value
- `NONE`: No credits initialized
- `ACTIVE`: Credits available for retirement
- `RETIRED`: All credits have been retired

**Example Usage**:
```solidity
CarbonCreditStatus status = ICarbonCredit(diamond).getCarbonCreditStatus(tokenId);

if (status == CarbonCreditStatus.ACTIVE) {
    // Credits available for retirement
} else if (status == CarbonCreditStatus.RETIRED) {
    // All credits have been used
}
```

---

#### `getAllCarbonCreditBalances()`
```solidity
function getAllCarbonCreditBalances(uint256[] calldata tokenIds) external view returns (uint256[] memory)
```

**Purpose**: Efficiently retrieves carbon credit balances for multiple tokens.

**Parameters**:
- `tokenIds` (uint256[]): Array of token IDs to query

**Returns**: Array of balances in the same order as input token IDs

**Gas Optimization**: Single call instead of multiple individual queries

**Example Usage**:
```solidity
uint256[] memory tokenIds = [1, 2, 3, 4, 5];
uint256[] memory balances = ICarbonCredit(diamond).getAllCarbonCreditBalances(tokenIds);

for (uint256 i = 0; i < tokenIds.length; i++) {
    console.log("Token", tokenIds[i], "has", balances[i], "credits");
}
```

## Events

### CarbonCreditsInitialized
```solidity
event CarbonCreditsInitialized(uint256 indexed tokenId, uint256 initialBalance);
```
Emitted when carbon credits are initialized for a token.

### CarbonCreditsRetired
```solidity
event CarbonCreditsRetired(uint256 indexed tokenId, uint256 amount, uint256 remainingBalance);
```
Emitted when carbon credits are retired, including the remaining balance.

## Integration Examples

### Environmental NFT Marketplace
```solidity
// 1. Mint environmental asset NFT
uint256 tokenId = IEnvironmentalNFT(diamond).mint(
    recipient,
    "Forest Conservation Project #1",
    metadataURI
);

// 2. Initialize with carbon credits
ICarbonCredit(diamond).initializeCarbonCredit(tokenId, 5000);

// 3. List on marketplace
IMarketplace(diamond).listItem(
    address(0),
    payable(seller),
    tokenId,
    1 ether,
    false,
    address(0)
);
```

### Carbon Offset Program
```solidity
// User purchases environmental NFT and retires credits for offsetting
function offsetCarbonFootprint(uint256 tokenId, uint256 offsetAmount) external {
    // Verify user owns the token
    require(IERC721(diamond).ownerOf(tokenId) == msg.sender, "Not token owner");
    
    // Check available credits
    uint256 available = ICarbonCredit(diamond).getCarbonCreditBalance(tokenId);
    require(available >= offsetAmount, "Insufficient credits");
    
    // Retire credits for offsetting
    ICarbonCredit(diamond).retireCarbonCredits(tokenId, offsetAmount);
    
    // Record offset in external system
    emit CarbonFootprintOffset(msg.sender, tokenId, offsetAmount);
}
```

### Batch Environmental Asset Management
```solidity
// Deploy multiple environmental projects
function deployEnvironmentalProjects(
    string[] memory projectNames,
    uint256[] memory creditAmounts
) external onlyOwner {
    uint256[] memory tokenIds = new uint256[](projectNames.length);
    
    // Mint NFTs for each project
    for (uint256 i = 0; i < projectNames.length; i++) {
        tokenIds[i] = IEnvironmentalNFT(diamond).mint(
            address(this),
            projectNames[i],
            generateMetadataURI(projectNames[i])
        );
    }
    
    // Batch initialize carbon credits
    ICarbonCredit(diamond).batchInitializeCarbonCredits(tokenIds, creditAmounts);
}
```

### Portfolio Tracking
```solidity
// Track carbon credit portfolio
function getPortfolioStatus(address owner) external view returns (
    uint256[] memory tokenIds,
    uint256[] memory balances,
    CarbonCreditStatus[] memory statuses
) {
    // Get all tokens owned by user
    tokenIds = getTokensByOwner(owner);
    
    // Get balances for all tokens
    balances = ICarbonCredit(diamond).getAllCarbonCreditBalances(tokenIds);
    
    // Get status for each token
    statuses = new CarbonCreditStatus[](tokenIds.length);
    for (uint256 i = 0; i < tokenIds.length; i++) {
        statuses[i] = ICarbonCredit(diamond).getCarbonCreditStatus(tokenIds[i]);
    }
}
```

## Environmental Standards Compliance

### International Standards
The carbon credit system supports compliance with:
- **Verified Carbon Standard (VCS)**
- **Clean Development Mechanism (CDM)**
- **Gold Standard**
- **Climate Action Reserve (CAR)**

### Audit Trail
- All initialization events are permanently recorded
- Retirement events provide complete audit trail
- Status tracking enables compliance verification
- Immutable blockchain records prevent tampering

### Double-Counting Prevention
- Credits can only be retired once
- No transfer of retired credits
- Permanent reduction in available balance
- Clear status indication for auditors

## Security Considerations

### Access Control
- Only contract owner can initialize credits
- Only token owner can retire credits
- Token existence validation prevents invalid operations

### Data Integrity
- Diamond storage pattern prevents storage collisions
- Immutable retirement records
- Overflow protection in arithmetic operations

### Audit Compliance
- Complete event logging for all operations
- Transparent balance tracking
- Status verification for compliance reporting

## Gas Optimization

### Batch Operations
- `batchInitializeCarbonCredits()` for efficient initialization
- `getAllCarbonCreditBalances()` for bulk queries
- Reduced transaction costs for large-scale operations

### Storage Efficiency
- Efficient mapping structures in CarbonCreditLib
- Minimal storage footprint per token
- Optimized for frequent read operations

## Error Handling

### Validation Errors
- `"CARBON_CREDIT: INVALID_TOKEN_ID"` - Invalid token ID
- `"CARBON_CREDIT: INVALID_BALANCE"` - Invalid balance amount
- `"CARBON_CREDIT: INVALID_AMOUNT"` - Invalid retirement amount

### State Errors
- `"CARBON_CREDIT: NONEXISTENT_TOKEN"` - Token doesn't exist
- `"CARBON_CREDIT: ALREADY_INITIALIZED"` - Token already has credits
- `"CARBON_CREDIT: INSUFFICIENT_BALANCE"` - Not enough credits to retire

### Access Errors
- `"Only token owner can call this function"` - Unauthorized retirement attempt

## Testing Considerations

### Unit Tests
- Credit initialization with various amounts
- Retirement operations and balance updates
- Status transitions and validation
- Batch operation efficiency

### Integration Tests
- NFT marketplace integration
- Multi-token portfolio management
- Environmental compliance workflows
- Audit trail verification

## Related Documentation

- [ICarbonCredit Interface](../interfaces/icarbon-credit.md) - Carbon credit interface
- [CarbonCreditLib](../libraries/carbon-credit-lib.md) - Carbon credit utility library
- [EIP-DRAFT-Carbon-Credit-Standard](../../eips/EIP-DRAFT-Carbon-Credit-Standard.md) - EIP specification
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md) - Implementation guide (used to replace environmental-assets.md)

---

*This facet implements the Carbon Credit Standard as defined in the Gemforce EIP suite, enabling comprehensive environmental asset management on the blockchain.*