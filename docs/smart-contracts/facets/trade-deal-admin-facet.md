# TradeDealAdminFacet

## Overview

The [`TradeDealAdminFacet.sol`](/Users/sschepis/Development/gem-base/contracts/facets/TradeDealAdminFacet.sol) provides administrative functionality for managing trade deals within the Gemforce diamond system. This facet handles participant management, interest distribution, claim topic configuration, and token address management for trade deal operations.

## Contract Details

- **Contract Name**: `TradeDealAdminFacet`
- **Inheritance**: `Modifiers`
- **License**: MIT
- **Solidity Version**: ^0.8.0

## Key Features

### 🔹 Participant Management
- Add and remove trade deal participants
- Access control for participant operations
- Event-driven participant tracking
- Integration with identity verification

### 🔹 Interest Distribution
- Automated interest calculation and distribution
- Multi-pool interest allocation
- Token minting for interest payments
- Transparent distribution tracking

### 🔹 Configuration Management
- Set required claim topics for verification
- Configure token addresses for trade deals
- Update trade deal parameters
- Backward compatibility support

### 🔹 Administrative Controls
- Owner-only access for sensitive operations
- Comprehensive event logging
- Integration with TradeDealLib utilities
- Secure parameter validation

## Core Functions

### Participant Management Functions

#### `addTradeDealParticipant()`
```solidity
function addTradeDealParticipant(uint256 tradeDealId, address participant) external onlyOwner
```

**Purpose**: Adds a new participant to an existing trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `participant` (address): Address of the participant to add

**Access Control**: Owner only

**Process**:
1. Validates trade deal exists and is active
2. Checks participant eligibility and identity verification
3. Adds participant to trade deal participant list
4. Updates participant tracking and permissions
5. Emits TradeDealParticipantAdded event

**Events**: `TradeDealParticipantAdded(tradeDealId, participant)`

**Example Usage**:
```solidity
// Add verified participant to trade deal
address newParticipant = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;
uint256 tradeDealId = 1;

ITradeDealAdmin(diamond).addTradeDealParticipant(tradeDealId, newParticipant);
console.log("Participant added to trade deal:", tradeDealId);
```

**Requirements**:
- Trade deal must exist and be active
- Participant must not already be in the trade deal
- Participant must meet identity verification requirements
- Caller must be contract owner

---

#### `removeTradeDealParticipant()`
```solidity
function removeTradeDealParticipant(uint256 tradeDealId, address participant) external onlyOwner
```

**Purpose**: Removes a participant from an existing trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `participant` (address): Address of the participant to remove

**Access Control**: Owner only

**Process**:
1. Validates trade deal exists
2. Confirms participant is currently in the trade deal
3. Checks for outstanding obligations or positions
4. Removes participant from trade deal
5. Updates tracking and revokes permissions
6. Emits TradeDealParticipantRemoved event

**Events**: `TradeDealParticipantRemoved(tradeDealId, participant)`

**Example Usage**:
```solidity
// Remove participant from trade deal
address participantToRemove = 0x742d35Cc6634C0532925a3b8D0C9e3e0C8b0e5e1;
uint256 tradeDealId = 1;

ITradeDealAdmin(diamond).removeTradeDealParticipant(tradeDealId, participantToRemove);
console.log("Participant removed from trade deal:", tradeDealId);
```

**Requirements**:
- Trade deal must exist
- Participant must be currently in the trade deal
- Participant must not have outstanding obligations
- Caller must be contract owner

### Interest Distribution Functions

#### `tdDistributeInterest()`
```solidity
function tdDistributeInterest(uint256 tradeDealId) external onlyOwner
```

**Purpose**: Distributes accumulated interest for a specific trade deal across multiple pools and participants.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal for interest distribution

**Access Control**: Owner only

**Process**:
1. Calculates total accumulated interest for the trade deal
2. Distributes interest across different pools:
   - Invoice pool interest
   - Interest pool allocation
   - Participant distributions
3. Mints interest tokens as needed
4. Updates interest tracking and balances
5. Emits InterestDistributedForTradeDeal event

**Events**: `InterestDistributedForTradeDeal(tradeDealId, totalInterest, invoicePoolInterest, interestInterest, interestTokensMinted)`

**Example Usage**:
```solidity
// Distribute interest for trade deal
uint256 tradeDealId = 1;

ITradeDealAdmin(diamond).tdDistributeInterest(tradeDealId);
console.log("Interest distributed for trade deal:", tradeDealId);
```

**Interest Distribution Breakdown**:
- **Total Interest**: Complete interest amount calculated
- **Invoice Pool Interest**: Portion allocated to invoice backing
- **Interest Interest**: Compound interest on existing interest
- **Interest Tokens Minted**: New tokens created for distribution

**Requirements**:
- Trade deal must exist and be active
- Interest must have accumulated since last distribution
- Trade deal must have active participants
- Caller must be contract owner

### Configuration Management Functions

#### `setTradeDealRequiredClaimTopics()`
```solidity
function setTradeDealRequiredClaimTopics(uint256 tradeDealId, uint256[] memory claimTopics) external onlyOwner
```

**Purpose**: Sets the required identity claim topics for trade deal participation.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `claimTopics` (uint256[]): Array of claim topic IDs required for participation

**Access Control**: Owner only

**Process**:
1. Validates trade deal exists
2. Updates required claim topics for the trade deal
3. Applies new requirements to future participants
4. Existing participants may need re-verification
5. Emits TradeDealRequiredClaimTopicsSet event

**Events**: `TradeDealRequiredClaimTopicsSet(tradeDealId, claimTopics)`

**Example Usage**:
```solidity
// Set KYC and accreditation requirements
uint256[] memory requiredClaims = new uint256[](3);
requiredClaims[0] = 1; // KYC verification
requiredClaims[1] = 2; // Accredited investor status
requiredClaims[2] = 3; // Geographic eligibility

ITradeDealAdmin(diamond).setTradeDealRequiredClaimTopics(tradeDealId, requiredClaims);
console.log("Claim topics set for trade deal:", tradeDealId);
```

**Common Claim Topics**:
- **1**: KYC (Know Your Customer) verification
- **2**: Accredited investor status
- **3**: Geographic eligibility
- **4**: Professional investor qualification
- **5**: Institutional investor status

---

#### `setTradeDealTokenAddresses()`
```solidity
function setTradeDealTokenAddresses(
    uint256 tradeDealId,
    address nftAddress,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
) external onlyOwner
```

**Purpose**: Configures the token contract addresses used by a trade deal.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `nftAddress` (address): Address of the NFT contract (currently unused)
- `collateralAddress` (address): Address of the collateral token contract
- `interestAddress` (address): Address of the interest token contract
- `usdcAddress` (address): Address of the USDC token contract

**Access Control**: Owner only

**Process**:
1. Validates trade deal exists
2. Updates token contract addresses in trade deal configuration
3. Validates token contracts implement required interfaces
4. Updates trade deal settings and parameters
5. Emits TradeDealUpdated event with complete configuration

**Events**: `TradeDealUpdated(tradeDealId, name, symbol, interestRate, collateralToInterestRatio, active, collateralAddress, interestAddress, usdcAddress)`

**Example Usage**:
```solidity
// Configure token addresses for trade deal
address collateralToken = 0x1234567890123456789012345678901234567890;
address interestToken = 0x2345678901234567890123456789012345678901;
address usdcToken = 0x3456789012345678901234567890123456789012;

ITradeDealAdmin(diamond).setTradeDealTokenAddresses(
    tradeDealId,
    address(0), // NFT address not used
    collateralToken,
    interestToken,
    usdcToken
);
```

**Token Address Roles**:
- **Collateral Address**: Token used as collateral backing
- **Interest Address**: Token used for interest payments
- **USDC Address**: Stablecoin for value reference and payments

---

#### `setTradeDealNFTAddress()`
```solidity
function setTradeDealNFTAddress(uint256 tradeDealId, address nftAddress) external onlyOwner
```

**Purpose**: Legacy function maintained for backward compatibility. Currently performs no operation.

**Parameters**:
- `tradeDealId` (uint256): ID of the trade deal
- `nftAddress` (address): NFT contract address (ignored)

**Access Control**: Owner only

**Note**: This function is a no-op maintained for backward compatibility with older integrations.

## Integration Examples

### Trade Deal Participant Management
```solidity
// Comprehensive participant management system
contract TradeDealParticipantManager {
    struct ParticipantInfo {
        address participant;
        uint256 joinedAt;
        uint256[] claimTopics;
        bool active;
    }
    
    mapping(uint256 => mapping(address => ParticipantInfo)) public tradeDealParticipants;
    mapping(uint256 => address[]) public participantLists;
    
    function addVerifiedParticipant(
        uint256 tradeDealId,
        address participant,
        uint256[] memory verifiedClaims
    ) external onlyOwner {
        // Verify participant has required claims
        uint256[] memory requiredClaims = getTradeDealRequiredClaims(tradeDealId);
        require(hasRequiredClaims(verifiedClaims, requiredClaims), "Missing required claims");
        
        // Add to trade deal
        ITradeDealAdmin(diamond).addTradeDealParticipant(tradeDealId, participant);
        
        // Track participant info
        tradeDealParticipants[tradeDealId][participant] = ParticipantInfo({
            participant: participant,
            joinedAt: block.timestamp,
            claimTopics: verifiedClaims,
            active: true
        });
        
        participantLists[tradeDealId].push(participant);
        
        emit ParticipantAdded(tradeDealId, participant, verifiedClaims);
    }
    
    function removeParticipant(uint256 tradeDealId, address participant) external onlyOwner {
        require(tradeDealParticipants[tradeDealId][participant].active, "Participant not active");
        
        // Remove from trade deal
        ITradeDealAdmin(diamond).removeTradeDealParticipant(tradeDealId, participant);
        
        // Update tracking
        tradeDealParticipants[tradeDealId][participant].active = false;
        
        // Remove from list
        _removeFromParticipantList(tradeDealId, participant);
        
        emit ParticipantRemoved(tradeDealId, participant);
    }
    
    function hasRequiredClaims(
        uint256[] memory userClaims,
        uint256[] memory requiredClaims
    ) internal pure returns (bool) {
        for (uint256 i = 0; i < requiredClaims.length; i++) {
            bool hasRequiredClaim = false;
            for (uint256 j = 0; j < userClaims.length; j++) {
                if (userClaims[j] == requiredClaims[i]) {
                    hasRequiredClaim = true;
                    break;
                }
            }
            if (!hasRequiredClaim) {
                return false;
            }
        }
        return true;
    }
}
```

### Automated Interest Distribution System
```solidity
// Automated interest distribution with scheduling
contract InterestDistributionScheduler {
    struct DistributionSchedule {
        uint256 tradeDealId;
        uint256 frequency; // in seconds
        uint256 lastDistribution;
        bool active;
    }
    
    mapping(uint256 => DistributionSchedule) public distributionSchedules;
    uint256[] public scheduledTradeDealIds;
    
    function setDistributionSchedule(
        uint256 tradeDealId,
        uint256 frequency
    ) external onlyOwner {
        distributionSchedules[tradeDealId] = DistributionSchedule({
            tradeDealId: tradeDealId,
            frequency: frequency,
            lastDistribution: block.timestamp,
            active: true
        });
        
        scheduledTradeDealIds.push(tradeDealId);
        emit DistributionScheduleSet(tradeDealId, frequency);
    }
    
    function executeScheduledDistributions() external {
        for (uint256 i = 0; i < scheduledTradeDealIds.length; i++) {
            uint256 tradeDealId = scheduledTradeDealIds[i];
            DistributionSchedule storage schedule = distributionSchedules[tradeDealId];
            
            if (schedule.active && 
                block.timestamp >= schedule.lastDistribution + schedule.frequency) {
                
                // Execute distribution
                ITradeDealAdmin(diamond).tdDistributeInterest(tradeDealId);
                
                // Update schedule
                schedule.lastDistribution = block.timestamp;
                
                emit ScheduledDistributionExecuted(tradeDealId, block.timestamp);
            }
        }
    }
    
    function getDistributionStatus(uint256 tradeDealId) external view returns (
        bool isDue,
        uint256 nextDistribution,
        uint256 timeSinceLastDistribution
    ) {
        DistributionSchedule memory schedule = distributionSchedules[tradeDealId];
        
        if (!schedule.active) {
            return (false, 0, 0);
        }
        
        timeSinceLastDistribution = block.timestamp - schedule.lastDistribution;
        nextDistribution = schedule.lastDistribution + schedule.frequency;
        isDue = block.timestamp >= nextDistribution;
    }
}
```

### Trade Deal Configuration Manager
```solidity
// Comprehensive trade deal configuration management
contract TradeDealConfigManager {
    struct TradeDealConfig {
        string name;
        string symbol;
        uint256 interestRate;
        uint256 collateralToInterestRatio;
        address collateralAddress;
        address interestAddress;
        address usdcAddress;
        uint256[] requiredClaimTopics;
        bool active;
    }
    
    mapping(uint256 => TradeDealConfig) public tradeDealConfigs;
    
    function configureTradeDeal(
        uint256 tradeDealId,
        TradeDealConfig memory config
    ) external onlyOwner {
        // Set token addresses
        ITradeDealAdmin(diamond).setTradeDealTokenAddresses(
            tradeDealId,
            address(0), // NFT address not used
            config.collateralAddress,
            config.interestAddress,
            config.usdcAddress
        );
        
        // Set required claim topics
        ITradeDealAdmin(diamond).setTradeDealRequiredClaimTopics(
            tradeDealId,
            config.requiredClaimTopics
        );
        
        // Store configuration
        tradeDealConfigs[tradeDealId] = config;
        
        emit TradeDealConfigured(tradeDealId, config);
    }
    
    function updateTradeDealTokens(
        uint256 tradeDealId,
        address newCollateralAddress,
        address newInterestAddress,
        address newUsdcAddress
    ) external onlyOwner {
        TradeDealConfig storage config = tradeDealConfigs[tradeDealId];
        
        // Update addresses
        ITradeDealAdmin(diamond).setTradeDealTokenAddresses(
            tradeDealId,
            address(0),
            newCollateralAddress,
            newInterestAddress,
            newUsdcAddress
        );
        
        // Update stored config
        config.collateralAddress = newCollateralAddress;
        config.interestAddress = newInterestAddress;
        config.usdcAddress = newUsdcAddress;
        
        emit TradeDealTokensUpdated(tradeDealId, newCollateralAddress, newInterestAddress, newUsdcAddress);
    }
    
    function updateRequiredClaims(
        uint256 tradeDealId,
        uint256[] memory newClaimTopics
    ) external onlyOwner {
        // Update claim topics
        ITradeDealAdmin(diamond).setTradeDealRequiredClaimTopics(tradeDealId, newClaimTopics);
        
        // Update stored config
        tradeDealConfigs[tradeDealId].requiredClaimTopics = newClaimTopics;
        
        emit RequiredClaimsUpdated(tradeDealId, newClaimTopics);
    }
}
```

### Interest Distribution Analytics
```solidity
// Analytics and reporting for interest distributions
contract InterestDistributionAnalytics {
    struct DistributionRecord {
        uint256 timestamp;
        uint256 totalInterest;
        uint256 invoicePoolInterest;
        uint256 interestInterest;
        uint256 interestTokensMinted;
        uint256 participantCount;
    }
    
    mapping(uint256 => DistributionRecord[]) public distributionHistory;
    mapping(uint256 => uint256) public totalDistributedInterest;
    mapping(uint256 => uint256) public totalTokensMinted;
    
    function recordDistribution(
        uint256 tradeDealId,
        uint256 totalInterest,
        uint256 invoicePoolInterest,
        uint256 interestInterest,
        uint256 interestTokensMinted,
        uint256 participantCount
    ) external onlyAuthorized {
        DistributionRecord memory record = DistributionRecord({
            timestamp: block.timestamp,
            totalInterest: totalInterest,
            invoicePoolInterest: invoicePoolInterest,
            interestInterest: interestInterest,
            interestTokensMinted: interestTokensMinted,
            participantCount: participantCount
        });
        
        distributionHistory[tradeDealId].push(record);
        totalDistributedInterest[tradeDealId] += totalInterest;
        totalTokensMinted[tradeDealId] += interestTokensMinted;
        
        emit DistributionRecorded(tradeDealId, record);
    }
    
    function getDistributionStats(uint256 tradeDealId) external view returns (
        uint256 totalDistributions,
        uint256 totalInterestDistributed,
        uint256 totalTokensMintedForInterest,
        uint256 averageDistributionAmount,
        uint256 lastDistributionTime
    ) {
        DistributionRecord[] memory history = distributionHistory[tradeDealId];
        
        totalDistributions = history.length;
        totalInterestDistributed = totalDistributedInterest[tradeDealId];
        totalTokensMintedForInterest = totalTokensMinted[tradeDealId];
        
        if (totalDistributions > 0) {
            averageDistributionAmount = totalInterestDistributed / totalDistributions;
            lastDistributionTime = history[totalDistributions - 1].timestamp;
        }
    }
    
    function getRecentDistributions(
        uint256 tradeDealId,
        uint256 count
    ) external view returns (DistributionRecord[] memory) {
        DistributionRecord[] memory history = distributionHistory[tradeDealId];
        
        if (history.length == 0) {
            return new DistributionRecord[](0);
        }
        
        uint256 startIndex = history.length > count ? history.length - count : 0;
        uint256 resultLength = history.length - startIndex;
        
        DistributionRecord[] memory recent = new DistributionRecord[](resultLength);
        
        for (uint256 i = 0; i < resultLength; i++) {
            recent[i] = history[startIndex + i];
        }
        
        return recent;
    }
}
```

## Events

### Participant Management Events
```solidity
event TradeDealParticipantAdded(uint256 indexed tradeDealId, address indexed participant);
event TradeDealParticipantRemoved(uint256 indexed tradeDealId, address indexed participant);
```

### Interest Distribution Events
```solidity
event InterestDistributedForTradeDeal(
    uint256 indexed tradeDealId,
    uint256 totalInterest,
    uint256 invoicePoolInterest,
    uint256 interestInterest,
    uint256 interestTokensMinted
);
```

### Configuration Events
```solidity
event TradeDealRequiredClaimTopicsSet(uint256 indexed tradeDealId, uint256[] claimTopics);
event TradeDealUpdated(
    uint256 indexed tradeDealId,
    string name,
    string symbol,
    uint256 interestRate,
    uint256 collateralToInterestRatio,
    bool active,
    address collateralAddress,
    address interestAddress,
    address usdcAddress
);
```

## Security Considerations

### Access Control
- All functions restricted to contract owner
- Participant validation before addition/removal
- Token address validation before configuration
- Claim topic verification for participants

### Interest Distribution Security
- Precise interest calculation validation
- Overflow protection for large distributions
- Token minting authorization checks
- Distribution frequency limits

### Configuration Security
- Token contract interface validation
- Address zero checks for critical addresses
- Claim topic existence verification
- Configuration change event logging

## Gas Optimization

### Batch Operations
- Consider batching participant operations
- Optimize interest distribution calculations
- Minimize storage operations
- Use events for off-chain tracking

### Efficient Storage
- Pack configuration data efficiently
- Use mappings for participant tracking
- Optimize claim topic storage
- Cache frequently accessed data

## Error Handling

### Common Errors
- Trade deal not found
- Participant already exists/doesn't exist
- Invalid token addresses
- Insufficient permissions
- Interest distribution failures

### Best Practices
- Validate all inputs before processing
- Check trade deal state before operations
- Verify participant eligibility
- Handle token contract failures gracefully

## Testing Considerations

### Unit Tests
- Participant addition and removal
- Interest distribution calculations
- Configuration updates
- Access control enforcement

### Integration Tests
- Multi-participant scenarios
- Interest distribution workflows
- Configuration change impacts
- Event emission verification

## Related Documentation

- [TradeDealManagementFacet](./trade-deal-management-facet.md) - Core trade deal operations
- [TradeDealOperationsFacet](./trade-deal-operations-facet.md) - Operational functions
- [TradeDealLib](../libraries/trade-deal-lib.md) - Trade deal utilities
- [IdentityRegistryFacet](./identity-registry-facet.md) - Identity verification
- [Trade Deal Guide](../../guides/trade-deals.md) - Implementation guide

---

*This facet provides comprehensive administrative functionality for trade deal management within the Gemforce platform, enabling secure participant management, interest distribution, and configuration control.*