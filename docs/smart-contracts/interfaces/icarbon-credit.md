# ICarbonCredit Interface

## Overview

The [`ICarbonCredit`](../../../contracts/interfaces/ICarbonCredit.sol) interface defines the standard contract for managing carbon credits associated with ERC721 tokens within the Gemforce platform. This interface enables the tokenization of environmental assets, allowing NFTs to represent carbon credits that can be tracked, managed, and retired for environmental impact purposes.

## Key Features

- **Carbon Credit Tokenization**: Associate carbon credit balances with ERC721 tokens
- **Credit Retirement**: Permanently retire carbon credits to prevent double-counting
- **Batch Operations**: Efficiently manage multiple tokens simultaneously
- **Status Tracking**: Monitor active vs. retired carbon credit status
- **Environmental Compliance**: Support for carbon offset and sustainability programs

## Interface Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { CarbonCreditStatus } from "../libraries/CarbonCreditLib.sol";

interface ICarbonCredit {
    // Events
    event CarbonCreditsInitialized(uint256 indexed tokenId, uint256 initialBalance);
    event CarbonCreditsRetired(uint256 indexed tokenId, uint256 amount, uint256 remainingBalance);
    
    // Core Functions
    function initializeCarbonCredit(uint256 tokenId, uint256 initialBalance) external;
    function retireCarbonCredits(uint256 tokenId, uint256 amount) external;
    function getCarbonCreditStatus(uint256 tokenId) external view returns (CarbonCreditStatus);
    function getCarbonCreditBalance(uint256 tokenId) external view returns (uint256);
    
    // Batch Operations
    function batchInitializeCarbonCredits(uint256[] calldata tokenIds, uint256[] calldata initialBalances) external;
    function getAllCarbonCreditBalances(uint256[] calldata tokenIds) external view returns (uint256[] memory);
}
```

## Core Functions

### Carbon Credit Initialization

#### `initializeCarbonCredit()`
```solidity
function initializeCarbonCredit(uint256 tokenId, uint256 initialBalance) external
```

**Purpose**: Initialize carbon credit balance for a specific ERC721 token.

**Parameters**:
- `tokenId` (uint256): The ID of the ERC721 token to associate with carbon credits
- `initialBalance` (uint256): The initial balance of carbon credits (must be > 0)

**Requirements**:
- Token must exist and have a valid owner
- Carbon credits must not already be initialized for this token
- Initial balance must be greater than zero
- Caller must have appropriate permissions

**Events Emitted**:
- [`CarbonCreditsInitialized`](icarbon-credit.md:12) with token ID and initial balance

**Example Usage**:
```solidity
// Initialize carbon credits for a newly minted environmental NFT
uint256 tokenId = 1;
uint256 initialCredits = 1000; // 1000 carbon credits

ICarbonCredit(diamond).initializeCarbonCredit(tokenId, initialCredits);
console.log("Initialized", initialCredits, "carbon credits for token", tokenId);
```

### Carbon Credit Retirement

#### `retireCarbonCredits()`
```solidity
function retireCarbonCredits(uint256 tokenId, uint256 amount) external
```

**Purpose**: Permanently retire carbon credits to prevent double-counting and demonstrate environmental impact.

**Parameters**:
- `tokenId` (uint256): The ID of the ERC721 token containing carbon credits
- `amount` (uint256): The amount of carbon credits to retire (must be whole number)

**Requirements**:
- Token must have sufficient carbon credit balance
- Amount must be greater than zero
- Amount must be less than or equal to current balance
- Caller must have appropriate permissions

**Events Emitted**:
- [`CarbonCreditsRetired`](icarbon-credit.md:18) with token ID, retired amount, and remaining balance

**Example Usage**:
```solidity
// Retire carbon credits to offset emissions
uint256 tokenId = 1;
uint256 retireAmount = 250; // Retire 250 carbon credits

// Check current balance first
uint256 currentBalance = ICarbonCredit(diamond).getCarbonCreditBalance(tokenId);
require(currentBalance >= retireAmount, "Insufficient carbon credits");

// Retire the credits
ICarbonCredit(diamond).retireCarbonCredits(tokenId, retireAmount);
console.log("Retired", retireAmount, "carbon credits from token", tokenId);
```

### Status and Balance Queries

#### `getCarbonCreditStatus()`
```solidity
function getCarbonCreditStatus(uint256 tokenId) external view returns (CarbonCreditStatus)
```

**Purpose**: Get the current status of carbon credits for a token.

**Parameters**:
- `tokenId` (uint256): The ID of the ERC721 token

**Returns**: 
- [`CarbonCreditStatus`](../libraries/carbon-credit-lib.md:4) enum value:
  - `ACTIVE`: Token has remaining carbon credits
  - `RETIRED`: All carbon credits have been retired

**Example Usage**:
```solidity
// Check carbon credit status
uint256 tokenId = 1;
CarbonCreditStatus status = ICarbonCredit(diamond).getCarbonCreditStatus(tokenId);

if (status == CarbonCreditStatus.ACTIVE) {
    console.log("Token", tokenId, "has active carbon credits");
} else {
    console.log("Token", tokenId, "carbon credits are fully retired");
}
```

#### `getCarbonCreditBalance()`
```solidity
function getCarbonCreditBalance(uint256 tokenId) external view returns (uint256)
```

**Purpose**: Get the current carbon credit balance for a specific token.

**Parameters**:
- `tokenId` (uint256): The ID of the ERC721 token

**Returns**: Current balance of carbon credits for the token

**Example Usage**:
```solidity
// Check current balance
uint256 tokenId = 1;
uint256 balance = ICarbonCredit(diamond).getCarbonCreditBalance(tokenId);
console.log("Token", tokenId, "has", balance, "carbon credits remaining");
```

## Batch Operations

### Batch Initialization

#### `batchInitializeCarbonCredits()`
```solidity
function batchInitializeCarbonCredits(uint256[] calldata tokenIds, uint256[] calldata initialBalances) external
```

**Purpose**: Initialize carbon credits for multiple tokens in a single transaction.

**Parameters**:
- `tokenIds` (uint256[]): Array of ERC721 token IDs
- `initialBalances` (uint256[]): Array of initial balances corresponding to token IDs

**Requirements**:
- Arrays must have the same length
- All tokens must exist and not have carbon credits initialized
- All initial balances must be greater than zero
- Caller must have appropriate permissions

**Gas Optimization**: More efficient than multiple individual calls

**Example Usage**:
```solidity
// Batch initialize carbon credits for multiple environmental NFTs
uint256[] memory tokenIds = new uint256[](3);
tokenIds[0] = 1;
tokenIds[1] = 2;
tokenIds[2] = 3;

uint256[] memory initialBalances = new uint256[](3);
initialBalances[0] = 1000; // Forest conservation project
initialBalances[1] = 500;  // Solar energy project
initialBalances[2] = 750;  // Wind energy project

ICarbonCredit(diamond).batchInitializeCarbonCredits(tokenIds, initialBalances);
console.log("Batch initialized carbon credits for", tokenIds.length, "tokens");
```

### Batch Balance Query

#### `getAllCarbonCreditBalances()`
```solidity
function getAllCarbonCreditBalances(uint256[] calldata tokenIds) external view returns (uint256[] memory)
```

**Purpose**: Get carbon credit balances for multiple tokens efficiently.

**Parameters**:
- `tokenIds` (uint256[]): Array of ERC721 token IDs to query

**Returns**: Array of carbon credit balances corresponding to the input token IDs

**Example Usage**:
```solidity
// Get balances for multiple tokens
uint256[] memory tokenIds = new uint256[](3);
tokenIds[0] = 1;
tokenIds[1] = 2;
tokenIds[2] = 3;

uint256[] memory balances = ICarbonCredit(diamond).getAllCarbonCreditBalances(tokenIds);

for (uint256 i = 0; i < tokenIds.length; i++) {
    console.log("Token", tokenIds[i], "balance:", balances[i]);
}
```

## Integration Examples

### Environmental NFT Marketplace
```solidity
// Integration with environmental NFT marketplace
contract EnvironmentalMarketplace {
    ICarbonCredit public carbonCredit;
    IERC721 public environmentalNFT;
    
    struct EnvironmentalListing {
        uint256 tokenId;
        uint256 price;
        uint256 carbonCredits;
        address seller;
        bool active;
    }
    
    mapping(uint256 => EnvironmentalListing) public listings;
    
    event EnvironmentalNFTListed(
        uint256 indexed tokenId,
        uint256 price,
        uint256 carbonCredits,
        address indexed seller
    );
    
    event CarbonCreditsRetiredOnPurchase(
        uint256 indexed tokenId,
        uint256 retiredAmount,
        address indexed buyer
    );
    
    constructor(address _carbonCredit, address _environmentalNFT) {
        carbonCredit = ICarbonCredit(_carbonCredit);
        environmentalNFT = IERC721(_environmentalNFT);
    }
    
    function listEnvironmentalNFT(
        uint256 tokenId,
        uint256 price,
        bool retireCreditsOnSale
    ) external {
        require(environmentalNFT.ownerOf(tokenId) == msg.sender, "Not token owner");
        
        uint256 creditBalance = carbonCredit.getCarbonCreditBalance(tokenId);
        require(creditBalance > 0, "No carbon credits associated");
        
        listings[tokenId] = EnvironmentalListing({
            tokenId: tokenId,
            price: price,
            carbonCredits: creditBalance,
            seller: msg.sender,
            active: true
        });
        
        emit EnvironmentalNFTListed(tokenId, price, creditBalance, msg.sender);
    }
    
    function purchaseEnvironmentalNFT(
        uint256 tokenId,
        uint256 creditsToRetire
    ) external payable {
        EnvironmentalListing storage listing = listings[tokenId];
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        
        uint256 currentCredits = carbonCredit.getCarbonCreditBalance(tokenId);
        require(creditsToRetire <= currentCredits, "Cannot retire more credits than available");
        
        // Transfer NFT
        environmentalNFT.safeTransferFrom(listing.seller, msg.sender, tokenId);
        
        // Retire carbon credits if requested
        if (creditsToRetire > 0) {
            carbonCredit.retireCarbonCredits(tokenId, creditsToRetire);
            emit CarbonCreditsRetiredOnPurchase(tokenId, creditsToRetire, msg.sender);
        }
        
        // Transfer payment
        payable(listing.seller).transfer(msg.value);
        
        // Deactivate listing
        listing.active = false;
    }
    
    function getListingWithCarbonInfo(uint256 tokenId) external view returns (
        EnvironmentalListing memory listing,
        uint256 currentCarbonBalance,
        CarbonCreditStatus status
    ) {
        listing = listings[tokenId];
        currentCarbonBalance = carbonCredit.getCarbonCreditBalance(tokenId);
        status = carbonCredit.getCarbonCreditStatus(tokenId);
    }
}
```

### Carbon Offset Program
```solidity
// Automated carbon offset program
contract CarbonOffsetProgram {
    ICarbonCredit public carbonCredit;
    
    struct OffsetProject {
        string name;
        string description;
        uint256[] tokenIds;
        uint256 totalCredits;
        uint256 retiredCredits;
        address projectOwner;
        bool active;
    }
    
    mapping(uint256 => OffsetProject) public projects;
    mapping(address => uint256[]) public userOffsets;
    uint256 public nextProjectId;
    
    event ProjectCreated(uint256 indexed projectId, string name, address indexed owner);
    event CreditsRetiredForOffset(uint256 indexed projectId, uint256 amount, address indexed offsetter);
    event ProjectCompleted(uint256 indexed projectId, uint256 totalRetired);
    
    function createOffsetProject(
        string memory name,
        string memory description,
        uint256[] memory tokenIds
    ) external returns (uint256 projectId) {
        projectId = nextProjectId++;
        
        uint256 totalCredits = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 balance = carbonCredit.getCarbonCreditBalance(tokenIds[i]);
            require(balance > 0, "Token has no carbon credits");
            totalCredits += balance;
        }
        
        projects[projectId] = OffsetProject({
            name: name,
            description: description,
            tokenIds: tokenIds,
            totalCredits: totalCredits,
            retiredCredits: 0,
            projectOwner: msg.sender,
            active: true
        });
        
        emit ProjectCreated(projectId, name, msg.sender);
    }
    
    function offsetEmissions(
        uint256 projectId,
        uint256 creditsToOffset
    ) external payable {
        OffsetProject storage project = projects[projectId];
        require(project.active, "Project not active");
        require(creditsToOffset > 0, "Must offset at least 1 credit");
        
        uint256 availableCredits = project.totalCredits - project.retiredCredits;
        require(creditsToOffset <= availableCredits, "Insufficient credits in project");
        
        // Calculate payment (simplified - would integrate with pricing oracle)
        uint256 costPerCredit = 10 ether; // $10 per credit in wei
        uint256 totalCost = creditsToOffset * costPerCredit;
        require(msg.value >= totalCost, "Insufficient payment");
        
        // Retire credits from project tokens
        uint256 creditsRemaining = creditsToOffset;
        for (uint256 i = 0; i < project.tokenIds.length && creditsRemaining > 0; i++) {
            uint256 tokenId = project.tokenIds[i];
            uint256 tokenBalance = carbonCredit.getCarbonCreditBalance(tokenId);
            
            if (tokenBalance > 0) {
                uint256 toRetire = creditsRemaining > tokenBalance ? tokenBalance : creditsRemaining;
                carbonCredit.retireCarbonCredits(tokenId, toRetire);
                creditsRemaining -= toRetire;
            }
        }
        
        project.retiredCredits += creditsToOffset;
        userOffsets[msg.sender].push(creditsToOffset);
        
        // Transfer payment to project owner
        payable(project.projectOwner).transfer(totalCost);
        
        emit CreditsRetiredForOffset(projectId, creditsToOffset, msg.sender);
        
        // Check if project is completed
        if (project.retiredCredits >= project.totalCredits) {
            project.active = false;
            emit ProjectCompleted(projectId, project.retiredCredits);
        }
    }
    
    function getProjectStatus(uint256 projectId) external view returns (
        string memory name,
        uint256 totalCredits,
        uint256 retiredCredits,
        uint256 availableCredits,
        bool active,
        uint256[] memory tokenBalances
    ) {
        OffsetProject memory project = projects[projectId];
        
        name = project.name;
        totalCredits = project.totalCredits;
        retiredCredits = project.retiredCredits;
        availableCredits = totalCredits - retiredCredits;
        active = project.active;
        
        tokenBalances = carbonCredit.getAllCarbonCreditBalances(project.tokenIds);
    }
    
    function getUserOffsetHistory(address user) external view returns (uint256[] memory) {
        return userOffsets[user];
    }
}
```

### Carbon Credit Analytics Dashboard
```solidity
// Analytics and reporting for carbon credits
contract CarbonCreditAnalytics {
    ICarbonCredit public carbonCredit;
    
    struct AnalyticsData {
        uint256 totalTokensWithCredits;
        uint256 totalActiveCredits;
        uint256 totalRetiredCredits;
        uint256 averageCreditsPerToken;
        uint256 retirementRate;
    }
    
    mapping(uint256 => uint256) public tokenInitialBalances;
    mapping(uint256 => uint256) public tokenRetirementHistory;
    uint256[] public trackedTokens;
    
    event AnalyticsUpdated(AnalyticsData data);
    event TokenAddedToTracking(uint256 indexed tokenId, uint256 initialBalance);
    
    function addTokenToTracking(uint256 tokenId) external {
        uint256 balance = carbonCredit.getCarbonCreditBalance(tokenId);
        require(balance > 0, "Token has no carbon credits");
        
        tokenInitialBalances[tokenId] = balance;
        trackedTokens.push(tokenId);
        
        emit TokenAddedToTracking(tokenId, balance);
    }
    
    function updateAnalytics() external returns (AnalyticsData memory data) {
        uint256[] memory balances = carbonCredit.getAllCarbonCreditBalances(trackedTokens);
        
        uint256 totalActive = 0;
        uint256 totalRetired = 0;
        uint256 tokensWithCredits = 0;
        
        for (uint256 i = 0; i < trackedTokens.length; i++) {
            uint256 tokenId = trackedTokens[i];
            uint256 currentBalance = balances[i];
            uint256 initialBalance = tokenInitialBalances[tokenId];
            
            if (currentBalance > 0) {
                tokensWithCredits++;
            }
            
            totalActive += currentBalance;
            totalRetired += (initialBalance - currentBalance);
        }
        
        data = AnalyticsData({
            totalTokensWithCredits: tokensWithCredits,
            totalActiveCredits: totalActive,
            totalRetiredCredits: totalRetired,
            averageCreditsPerToken: tokensWithCredits > 0 ? totalActive / tokensWithCredits : 0,
            retirementRate: (totalActive + totalRetired) > 0 ? (totalRetired * 100) / (totalActive + totalRetired) : 0
        });
        
        emit AnalyticsUpdated(data);
    }
    
    function getTokenRetirementHistory(uint256 tokenId) external view returns (
        uint256 initialBalance,
        uint256 currentBalance,
        uint256 totalRetired,
        uint256 retirementPercentage
    ) {
        initialBalance = tokenInitialBalances[tokenId];
        currentBalance = carbonCredit.getCarbonCreditBalance(tokenId);
        totalRetired = initialBalance - currentBalance;
        retirementPercentage = initialBalance > 0 ? (totalRetired * 100) / initialBalance : 0;
    }
    
    function generateRetirementReport(uint256[] memory tokenIds) external view returns (
        uint256 totalInitial,
        uint256 totalCurrent,
        uint256 totalRetired,
        uint256 overallRetirementRate
    ) {
        uint256[] memory balances = carbonCredit.getAllCarbonCreditBalances(tokenIds);
        
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 initial = tokenInitialBalances[tokenIds[i]];
            uint256 current = balances[i];
            
            totalInitial += initial;
            totalCurrent += current;
            totalRetired += (initial - current);
        }
        
        overallRetirementRate = totalInitial > 0 ? (totalRetired * 100) / totalInitial : 0;
    }
}
```

## Events

### Carbon Credit Lifecycle Events
```solidity
event CarbonCreditsInitialized(
    uint256 indexed tokenId,
    uint256 initialBalance
);

event CarbonCreditsRetired(
    uint256 indexed tokenId,
    uint256 amount,
    uint256 remainingBalance
);
```

## Data Types

### CarbonCreditStatus Enum
```solidity
enum CarbonCreditStatus {
    ACTIVE,    // Token has remaining carbon credits
    RETIRED    // All carbon credits have been retired
}
```

## Security Considerations

### Access Control
- Initialization requires appropriate permissions
- Retirement operations must be authorized
- Token existence validation before operations
- Prevention of double initialization

### Financial Security
- Accurate balance tracking and calculations
- Prevention of over-retirement
- Secure state transitions
- Immutable retirement records

### Environmental Integrity
- Prevention of double-counting retired credits
- Permanent retirement to ensure environmental impact
- Transparent tracking of credit lifecycle
- Compliance with carbon accounting standards

## Gas Optimization

### Efficient Operations
- Batch operations for multiple tokens
- Optimized storage layout for balances
- Minimal external calls in view functions
- Efficient array operations for bulk queries

### Storage Optimization
- Packed storage structures where possible
- Efficient mapping usage for balance tracking
- Optimized batch operations
- Cached calculations for frequently accessed data

## Error Handling

### Common Errors
- Token does not exist
- Carbon credits already initialized
- Insufficient balance for retirement
- Invalid amounts (zero or negative)
- Array length mismatches in batch operations

### Best Practices
- Validate token existence before operations
- Check balances before retirement
- Validate array parameters in batch operations
- Provide clear error messages for debugging
- Handle edge cases gracefully

## Testing Considerations

### Unit Tests
- Interface compliance verification
- Balance initialization and tracking
- Retirement functionality and validation
- Batch operation efficiency and correctness
- Event emission verification

### Integration Tests
- Integration with ERC721 contracts
- Environmental marketplace workflows
- Carbon offset program functionality
- Analytics and reporting accuracy
- Cross-contract interaction testing

## Related Documentation

- [CarbonCreditFacet](../facets/carbon-credit-facet.md) - Implementation of carbon credit functionality
- [CarbonCreditLib](../libraries/carbon-credit-lib.md) - Carbon credit utilities and data structures
- [Environmental NFT Guide](../../guides/environmental-nfts.md) - Implementation guide for environmental assets
- [Carbon Offset Integration](../../guides/carbon-offset-integration.md) - Integration with offset programs
- [ERC721 Standards](../../standards/erc721-extensions.md) - Token standard extensions

---

*This interface defines the standard contract for carbon credit management within the Gemforce platform, enabling the tokenization and retirement of environmental assets for sustainability and carbon offset programs.*