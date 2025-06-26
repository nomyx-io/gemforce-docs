# EIP-DRAFT: Carbon Credit Standard for Tokenized Environmental Assets

## Simple Summary

A standardized interface for tokenizing, trading, and retiring carbon credits and other environmental assets on the Ethereum blockchain with full lifecycle tracking and verification.

## Abstract

This EIP proposes a comprehensive standard for carbon credits that enables:
- Tokenization of verified carbon credits as NFTs or fungible tokens
- Lifecycle tracking from issuance to retirement
- Integration with carbon registries and verification bodies
- Automated retirement and offset mechanisms
- Fractional ownership and trading capabilities
- Compliance with international carbon credit standards

## Motivation

The carbon credit market lacks standardization and transparency, making it difficult to verify the authenticity and prevent double-counting of credits. A blockchain-based standard can provide immutable tracking, automated verification, and seamless integration with DeFi protocols while maintaining compliance with existing carbon credit frameworks.

## Specification

### Core Interface

```solidity
interface ICarbonCredit {
    enum CreditStatus { ISSUED, ACTIVE, RETIRED, CANCELLED }
    enum CreditType { REMOVAL, AVOIDANCE, REDUCTION }
    enum Standard { VCS, CDM, GOLD_STANDARD, CAR, ACR, CUSTOM }
    
    struct CarbonCreditData {
        uint256 creditId;
        string projectId;
        Standard standard;
        CreditType creditType;
        uint256 vintage; // Year of carbon impact
        uint256 quantity; // Tonnes of CO2 equivalent
        string methodology;
        string verifier;
        string registry;
        uint256 issuanceDate;
        CreditStatus status;
        address currentOwner;
        string metadata; // IPFS hash or URI
    }
    
    struct RetirementData {
        uint256 creditId;
        uint256 quantity;
        address retiringEntity;
        string beneficiary;
        string retirementReason;
        uint256 retirementDate;
        string retirementCertificate; // IPFS hash
    }
    
    // Events
    event CarbonCreditIssued(uint256 indexed creditId, string projectId, Standard standard, uint256 quantity, uint256 vintage);
    event CarbonCreditTransferred(uint256 indexed creditId, address indexed from, address indexed to, uint256 quantity);
    event CarbonCreditRetired(uint256 indexed creditId, address indexed retiringEntity, uint256 quantity, string beneficiary);
    event CarbonCreditCancelled(uint256 indexed creditId, uint256 quantity, string reason);
    event CarbonCreditFractionalized(uint256 indexed creditId, address indexed fractionalToken, uint256 totalShares);
    event OffsetClaimed(address indexed entity, uint256 totalQuantity, uint256[] creditIds, string offsetClaim);
    
    // Core Functions
    function issueCarbonCredit(CarbonCreditData memory creditData) external returns (uint256 creditId);
    function transferCredit(uint256 creditId, address to, uint256 quantity) external;
    function retireCredit(uint256 creditId, uint256 quantity, string memory beneficiary, string memory reason) external returns (bytes32 retirementId);
    function cancelCredit(uint256 creditId, uint256 quantity, string memory reason) external;
    
    // Fractionalization
    function fractionalizeCarbonCredit(uint256 creditId, uint256 totalShares, string memory tokenName, string memory tokenSymbol) external returns (address fractionalToken);
    function redeemFractionalShares(address fractionalToken, uint256 shares) external;
    
    // Offset Claims
    function claimOffset(uint256[] memory creditIds, uint256[] memory quantities, string memory offsetClaim) external;
    function getOffsetHistory(address entity) external view returns (RetirementData[] memory);
    
    // View Functions
    function getCarbonCreditData(uint256 creditId) external view returns (CarbonCreditData memory);
    function getRetirementData(bytes32 retirementId) external view returns (RetirementData memory);
    function getCreditsByOwner(address owner) external view returns (uint256[] memory);
    function getCreditsByProject(string memory projectId) external view returns (uint256[] memory);
    function getCreditsByVintage(uint256 vintage) external view returns (uint256[] memory);
    function getTotalSupply() external view returns (uint256);
    function getTotalRetired() external view returns (uint256);
    function getAvailableCredits() external view returns (uint256);
    
    // Verification
    function verifyCreditAuthenticity(uint256 creditId) external view returns (bool isValid, string memory verificationDetails);
    function checkDoubleSpending(uint256 creditId) external view returns (bool hasDoubleSpending);
}

interface ICarbonCreditRegistry {
    struct ProjectData {
        string projectId;
        string name;
        string description;
        string location;
        string methodology;
        address projectDeveloper;
        Standard standard;
        CreditType creditType;
        uint256 estimatedCredits;
        uint256 issuedCredits;
        bool active;
        string verificationBody;
        string registryUrl;
    }
    
    // Events
    event ProjectRegistered(string indexed projectId, address indexed developer, Standard standard);
    event ProjectUpdated(string indexed projectId, bool active);
    event VerificationBodyAdded(address indexed verifier, string name);
    
    // Core Functions
    function registerProject(ProjectData memory projectData) external;
    function updateProject(string memory projectId, ProjectData memory projectData) external;
    function addVerificationBody(address verifier, string memory name, Standard[] memory authorizedStandards) external;
    function isAuthorizedVerifier(address verifier, Standard standard) external view returns (bool);
    function getProjectData(string memory projectId) external view returns (ProjectData memory);
    function getAllProjects() external view returns (string[] memory);
}
```

### Key Features

#### 1. Comprehensive Credit Tracking
- **Full Lifecycle**: Track credits from issuance through retirement
- **Immutable Records**: Blockchain-based immutable tracking
- **Status Management**: Clear status transitions and validation
- **Metadata Integration**: IPFS integration for detailed documentation

#### 2. Multiple Credit Standards
- **Standard Compliance**: Support for VCS, CDM, Gold Standard, and others
- **Flexible Framework**: Extensible to new standards and methodologies
- **Verification Integration**: Integration with recognized verification bodies

#### 3. Fractionalization Support
- **Fractional Ownership**: Enable fractional ownership of large credits
- **ERC20 Integration**: Create fungible tokens for fractional shares
- **Redemption Mechanism**: Convert fractional shares back to whole credits

#### 4. Automated Offset Claims
- **Batch Retirement**: Retire multiple credits in single transaction
- **Offset Tracking**: Track offset claims and retirement history
- **Certificate Generation**: Generate retirement certificates

### Credit Lifecycle

```solidity
// 1. Register Project
ProjectData memory project = ProjectData({
    projectId: "PROJ-001",
    name: "Reforestation Project",
    methodology: "VM0006",
    // ... other fields
});
registry.registerProject(project);

// 2. Issue Credits
CarbonCreditData memory credit = CarbonCreditData({
    projectId: "PROJ-001",
    standard: Standard.VCS,
    creditType: CreditType.REMOVAL,
    vintage: 2024,
    quantity: 1000,
    // ... other fields
});
uint256 creditId = carbonCredit.issueCarbonCredit(credit);

// 3. Transfer Credits
carbonCredit.transferCredit(creditId, buyer, 500);

// 4. Retire Credits for Offset
bytes32 retirementId = carbonCredit.retireCredit(creditId, 250, "Company XYZ", "Annual carbon neutrality");

// 5. Claim Offset
uint256[] memory creditIds = new uint256[](1);
creditIds[0] = creditId;
uint256[] memory quantities = new uint256[](1);
quantities[0] = 250;
carbonCredit.claimOffset(creditIds, quantities, "2024 Carbon Neutrality Claim");
```

## Rationale

### NFT vs Fungible Token Approach
The standard supports both approaches:
- **NFT Approach**: Each credit batch is unique with specific vintage, project, and verification data
- **Fungible Approach**: Credits with identical characteristics can be fungible
- **Hybrid Approach**: NFTs can be fractionalized into fungible tokens

### Immutable Retirement
Once credits are retired, they cannot be transferred or used again, preventing double-counting and ensuring offset integrity.

### Registry Integration
Integration with existing carbon registries ensures compliance with established standards and verification processes.

### Metadata Standards
Using IPFS for metadata storage ensures decentralized, permanent storage of verification documents and project details.

## Implementation Details

### Credit Issuance

```solidity
function issueCarbonCredit(CarbonCreditData memory creditData) external returns (uint256 creditId) {
    require(isAuthorizedIssuer(msg.sender), "Unauthorized issuer");
    require(registry.getProjectData(creditData.projectId).active, "Project not active");
    
    creditId = nextCreditId++;
    creditData.creditId = creditId;
    creditData.status = CreditStatus.ACTIVE;
    creditData.currentOwner = msg.sender;
    creditData.issuanceDate = block.timestamp;
    
    credits[creditId] = creditData;
    ownerCredits[msg.sender].push(creditId);
    
    emit CarbonCreditIssued(creditId, creditData.projectId, creditData.standard, creditData.quantity, creditData.vintage);
}
```

### Credit Retirement

```solidity
function retireCredit(uint256 creditId, uint256 quantity, string memory beneficiary, string memory reason) external returns (bytes32 retirementId) {
    CarbonCreditData storage credit = credits[creditId];
    require(credit.currentOwner == msg.sender, "Not credit owner");
    require(credit.status == CreditStatus.ACTIVE, "Credit not active");
    require(quantity <= credit.quantity, "Insufficient quantity");
    
    retirementId = keccak256(abi.encodePacked(creditId, quantity, block.timestamp, msg.sender));
    
    RetirementData memory retirement = RetirementData({
        creditId: creditId,
        quantity: quantity,
        retiringEntity: msg.sender,
        beneficiary: beneficiary,
        retirementReason: reason,
        retirementDate: block.timestamp,
        retirementCertificate: "" // Generated off-chain
    });
    
    retirements[retirementId] = retirement;
    credit.quantity -= quantity;
    
    if (credit.quantity == 0) {
        credit.status = CreditStatus.RETIRED;
    }
    
    emit CarbonCreditRetired(creditId, msg.sender, quantity, beneficiary);
}
```

### Security Considerations

1. **Authorization**: Proper authorization for credit issuance and management
2. **Double-Spending Prevention**: Mechanisms to prevent double-counting of credits
3. **Verification**: Integration with trusted verification bodies
4. **Immutable Retirement**: Ensure retired credits cannot be reused
5. **Registry Validation**: Validate project registration and status

### Compliance Considerations

1. **International Standards**: Compliance with VCS, CDM, and other standards
2. **Regulatory Requirements**: Consideration of emerging regulations
3. **Audit Trails**: Complete audit trails for compliance reporting
4. **Verification Requirements**: Integration with recognized verification bodies

## Integration Examples

### With DeFi Protocols

```solidity
contract CarbonOffsetDeFi {
    ICarbonCredit public carbonCredit;
    
    function offsetTransaction(uint256 creditId, uint256 quantity) external {
        // Retire credits to offset transaction
        carbonCredit.retireCredit(creditId, quantity, "DeFi Transaction Offset", "Automated offset");
    }
}
```

### With Corporate Sustainability

```solidity
contract CorporateOffsetManager {
    ICarbonCredit public carbonCredit;
    mapping(address => uint256) public annualOffsets;
    
    function executeAnnualOffset(uint256[] memory creditIds, uint256[] memory quantities) external {
        uint256 totalOffset = 0;
        for (uint256 i = 0; i < quantities.length; i++) {
            totalOffset += quantities[i];
        }
        
        carbonCredit.claimOffset(creditIds, quantities, "Annual Carbon Neutrality");
        annualOffsets[msg.sender] += totalOffset;
    }
}
```

## Test Cases

Comprehensive test cases should cover:
- Credit issuance and lifecycle management
- Transfer and ownership tracking
- Retirement and offset claiming
- Fractionalization and redemption
- Registry integration
- Verification and compliance
- Edge cases and error conditions

## Reference Implementation

The reference implementation includes:
- [`CarbonCreditFacet.sol`](../contracts/facets/CarbonCreditFacet.sol) - Core carbon credit functionality
- [`ICarbonCredit.sol`](../contracts/interfaces/ICarbonCredit.sol) - Interface definition
- [`CarbonCreditLib.sol`](../contracts/libraries/CarbonCreditLib.sol) - Supporting library functions

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).