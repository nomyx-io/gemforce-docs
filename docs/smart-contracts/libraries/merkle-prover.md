# MerkleProver Library

## Overview

The [`MerkleProver`](../../../contracts/libraries/MerkleProver.sol) library provides core utilities for Merkle tree proof verification within the Gemforce platform. This library implements secure and gas-efficient Merkle proof validation, enabling whitelist systems, airdrop distributions, and other scenarios requiring cryptographic proof of inclusion in a dataset.

## Key Features

- **Secure Proof Verification**: Cryptographically secure Merkle tree validation
- **Gas Efficient**: Optimized proof verification algorithms
- **Flexible Leaf Generation**: Support for various data types in leaf construction
- **Attack Prevention**: Protection against common Merkle tree vulnerabilities
- **Standard Compliance**: Compatible with standard Merkle tree implementations
- **Pure Functions**: Stateless operations for maximum reusability

## Library Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

library MerkleProver {
    // Proof verification
    function verify(bytes32 root, bytes32 leaf, bytes32[] memory proof) public pure returns (bool);
    
    // Leaf generation
    function getHash(address a, uint256 b) public pure returns (bytes32);
}
```

## Core Functions

### Proof Verification

#### `verify()`
```solidity
function verify(
    bytes32 root,
    bytes32 leaf,
    bytes32[] memory proof
) public pure returns (bool)
```

**Purpose**: Verify that a leaf is included in a Merkle tree with the given root.

**Parameters**:
- `root`: The Merkle root hash to verify against
- `leaf`: The leaf hash to prove inclusion for
- `proof`: Array of sibling hashes forming the proof path

**Returns**: Boolean indicating whether the proof is valid

**Algorithm**:
1. Start with the leaf hash as the computed hash
2. For each proof element, combine with computed hash in sorted order
3. Hash the combination using keccak256
4. Continue until all proof elements are processed
5. Compare final computed hash with provided root

**Security Features**:
- **Empty Proof Protection**: Prevents false positives when leaf equals root with empty proof
- **Sorted Hashing**: Ensures deterministic hash ordering regardless of tree structure
- **Overflow Protection**: Safe array iteration with bounds checking

**Implementation Details**:
```solidity
function verify(
    bytes32 root,
    bytes32 leaf,
    bytes32[] memory proof
) public pure returns (bool) {
    // Security check: prevent false positive when leaf == root with empty proof
    if (leaf == root && proof.length == 0) {
        return false;
    }

    bytes32 computedHash = leaf;

    for (uint256 i = 0; i < proof.length; i++) {
        bytes32 proofElement = proof[i];

        if (computedHash <= proofElement) {
            // Hash(current computed hash + current element of the proof)
            computedHash = keccak256(abi.encodePacked(computedHash, proofElement));
        } else {
            // Hash(current element of the proof + current computed hash)
            computedHash = keccak256(abi.encodePacked(proofElement, computedHash));
        }
    }

    // Check if the computed hash (root) is equal to the provided root
    return computedHash == root;
}
```

**Example Usage**:
```solidity
// Verify whitelist inclusion
bytes32 merkleRoot = 0x1234...;
bytes32 userLeaf = MerkleProver.getHash(msg.sender, allocation);
bytes32[] memory proof = [0xabcd..., 0xef01..., 0x2345...];

bool isValid = MerkleProver.verify(merkleRoot, userLeaf, proof);
require(isValid, "Invalid Merkle proof");
```

### Leaf Generation

#### `getHash()`
```solidity
function getHash(address a, uint256 b) public pure returns (bytes32)
```

**Purpose**: Generate a standardized leaf hash from an address and amount.

**Parameters**:
- `a`: Ethereum address (typically user address)
- `b`: Numeric value (typically allocation amount)

**Returns**: keccak256 hash of the packed parameters

**Implementation**:
```solidity
function getHash(address a, uint256 b) public pure returns (bytes32) {
    return keccak256(abi.encodePacked(a, b));
}
```

**Use Cases**:
- Whitelist leaf generation for address + allocation pairs
- Airdrop eligibility proofs
- Voting power verification
- Access control with quantified permissions

**Example Usage**:
```solidity
// Generate leaf for whitelist entry
address user = 0x742d35Cc6634C0532925a3b8D4C9db96590c6C87;
uint256 allocation = 1000 * 10**18; // 1000 tokens
bytes32 leaf = MerkleProver.getHash(user, allocation);
```

## Integration Examples

### Whitelist Token Sale
```solidity
// Token sale with Merkle tree whitelist
contract WhitelistTokenSale {
    using MerkleProver for bytes32;
    
    struct WhitelistConfig {
        bytes32 merkleRoot;
        uint256 salePrice;
        uint256 maxAllocation;
        uint256 startTime;
        uint256 endTime;
        bool active;
    }
    
    struct UserPurchase {
        uint256 purchased;
        bool claimed;
    }
    
    WhitelistConfig public whitelistConfig;
    mapping(address => UserPurchase) public userPurchases;
    mapping(bytes32 => bool) public usedLeaves;
    
    event WhitelistConfigured(bytes32 indexed merkleRoot, uint256 salePrice);
    event TokensPurchased(address indexed user, uint256 amount, uint256 totalCost);
    event ProofUsed(bytes32 indexed leaf, address indexed user);
    
    function configureWhitelist(
        bytes32 _merkleRoot,
        uint256 _salePrice,
        uint256 _maxAllocation,
        uint256 _startTime,
        uint256 _endTime
    ) external onlyOwner {
        whitelistConfig = WhitelistConfig({
            merkleRoot: _merkleRoot,
            salePrice: _salePrice,
            maxAllocation: _maxAllocation,
            startTime: _startTime,
            endTime: _endTime,
            active: true
        });
        
        emit WhitelistConfigured(_merkleRoot, _salePrice);
    }
    
    function purchaseTokens(
        uint256 allocation,
        uint256 purchaseAmount,
        bytes32[] calldata merkleProof
    ) external payable {
        require(whitelistConfig.active, "Whitelist sale not active");
        require(block.timestamp >= whitelistConfig.startTime, "Sale not started");
        require(block.timestamp <= whitelistConfig.endTime, "Sale ended");
        require(purchaseAmount > 0, "Purchase amount must be positive");
        
        // Generate leaf for this user and allocation
        bytes32 leaf = MerkleProver.getHash(msg.sender, allocation);
        
        // Verify Merkle proof
        bool isValidProof = MerkleProver.verify(
            whitelistConfig.merkleRoot,
            leaf,
            merkleProof
        );
        require(isValidProof, "Invalid Merkle proof");
        
        // Prevent proof reuse
        require(!usedLeaves[leaf], "Proof already used");
        
        // Check allocation limits
        UserPurchase storage userPurchase = userPurchases[msg.sender];
        require(
            userPurchase.purchased + purchaseAmount <= allocation,
            "Exceeds allocated amount"
        );
        require(
            userPurchase.purchased + purchaseAmount <= whitelistConfig.maxAllocation,
            "Exceeds max allocation"
        );
        
        // Calculate cost
        uint256 totalCost = purchaseAmount * whitelistConfig.salePrice;
        require(msg.value >= totalCost, "Insufficient payment");
        
        // Update user purchase record
        userPurchase.purchased += purchaseAmount;
        
        // Mark leaf as used if fully claimed
        if (userPurchase.purchased == allocation) {
            usedLeaves[leaf] = true;
            emit ProofUsed(leaf, msg.sender);
        }
        
        // Mint tokens to user
        _mintTokens(msg.sender, purchaseAmount);
        
        // Refund excess payment
        if (msg.value > totalCost) {
            payable(msg.sender).transfer(msg.value - totalCost);
        }
        
        emit TokensPurchased(msg.sender, purchaseAmount, totalCost);
    }
    
    function verifyWhitelistEligibility(
        address user,
        uint256 allocation,
        bytes32[] calldata merkleProof
    ) external view returns (bool isEligible, uint256 remainingAllocation) {
        bytes32 leaf = MerkleProver.getHash(user, allocation);
        
        isEligible = MerkleProver.verify(
            whitelistConfig.merkleRoot,
            leaf,
            merkleProof
        ) && !usedLeaves[leaf];
        
        if (isEligible) {
            remainingAllocation = allocation - userPurchases[user].purchased;
        }
    }
    
    function _mintTokens(address to, uint256 amount) internal {
        // Implementation would mint ERC20 tokens
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

### Airdrop Distribution System
```solidity
// Airdrop system with Merkle proof claims
contract MerkleAirdrop {
    using MerkleProver for bytes32;
    
    struct AirdropRound {
        bytes32 merkleRoot;
        uint256 totalTokens;
        uint256 claimedTokens;
        uint256 startTime;
        uint256 endTime;
        bool active;
        string description;
    }
    
    struct ClaimRecord {
        uint256 roundId;
        address claimer;
        uint256 amount;
        uint256 timestamp;
    }
    
    mapping(uint256 => AirdropRound) public airdropRounds;
    mapping(uint256 => mapping(bytes32 => bool)) public claimedLeaves;
    mapping(address => ClaimRecord[]) public userClaims;
    uint256 public nextRoundId;
    IERC20 public airdropToken;
    
    event AirdropRoundCreated(uint256 indexed roundId, bytes32 merkleRoot, uint256 totalTokens);
    event TokensClaimed(uint256 indexed roundId, address indexed claimer, uint256 amount);
    event RoundFinalized(uint256 indexed roundId, uint256 totalClaimed);
    
    constructor(address _airdropToken) {
        airdropToken = IERC20(_airdropToken);
    }
    
    function createAirdropRound(
        bytes32 _merkleRoot,
        uint256 _totalTokens,
        uint256 _startTime,
        uint256 _endTime,
        string memory _description
    ) external onlyOwner returns (uint256 roundId) {
        roundId = nextRoundId++;
        
        airdropRounds[roundId] = AirdropRound({
            merkleRoot: _merkleRoot,
            totalTokens: _totalTokens,
            claimedTokens: 0,
            startTime: _startTime,
            endTime: _endTime,
            active: true,
            description: _description
        });
        
        emit AirdropRoundCreated(roundId, _merkleRoot, _totalTokens);
    }
    
    function claimAirdrop(
        uint256 roundId,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external {
        AirdropRound storage round = airdropRounds[roundId];
        require(round.active, "Airdrop round not active");
        require(block.timestamp >= round.startTime, "Airdrop not started");
        require(block.timestamp <= round.endTime, "Airdrop ended");
        
        // Generate leaf for this claim
        bytes32 leaf = MerkleProver.getHash(msg.sender, amount);
        
        // Verify Merkle proof
        bool isValidProof = MerkleProver.verify(
            round.merkleRoot,
            leaf,
            merkleProof
        );
        require(isValidProof, "Invalid Merkle proof");
        
        // Prevent double claiming
        require(!claimedLeaves[roundId][leaf], "Already claimed");
        
        // Mark as claimed
        claimedLeaves[roundId][leaf] = true;
        round.claimedTokens += amount;
        
        // Record claim
        userClaims[msg.sender].push(ClaimRecord({
            roundId: roundId,
            claimer: msg.sender,
            amount: amount,
            timestamp: block.timestamp
        }));
        
        // Transfer tokens
        require(
            airdropToken.transfer(msg.sender, amount),
            "Token transfer failed"
        );
        
        emit TokensClaimed(roundId, msg.sender, amount);
    }
    
    function batchClaimAirdrop(
        uint256[] calldata roundIds,
        uint256[] calldata amounts,
        bytes32[][] calldata merkleProofs
    ) external {
        require(
            roundIds.length == amounts.length && amounts.length == merkleProofs.length,
            "Array length mismatch"
        );
        
        uint256 totalClaim = 0;
        
        for (uint256 i = 0; i < roundIds.length; i++) {
            uint256 roundId = roundIds[i];
            uint256 amount = amounts[i];
            bytes32[] calldata proof = merkleProofs[i];
            
            AirdropRound storage round = airdropRounds[roundId];
            require(round.active, "Airdrop round not active");
            require(block.timestamp >= round.startTime, "Airdrop not started");
            require(block.timestamp <= round.endTime, "Airdrop ended");
            
            bytes32 leaf = MerkleProver.getHash(msg.sender, amount);
            
            bool isValidProof = MerkleProver.verify(round.merkleRoot, leaf, proof);
            require(isValidProof, "Invalid Merkle proof");
            require(!claimedLeaves[roundId][leaf], "Already claimed");
            
            claimedLeaves[roundId][leaf] = true;
            round.claimedTokens += amount;
            totalClaim += amount;
            
            userClaims[msg.sender].push(ClaimRecord({
                roundId: roundId,
                claimer: msg.sender,
                amount: amount,
                timestamp: block.timestamp
            }));
            
            emit TokensClaimed(roundId, msg.sender, amount);
        }
        
        // Single token transfer for all claims
        require(
            airdropToken.transfer(msg.sender, totalClaim),
            "Token transfer failed"
        );
    }
    
    function verifyEligibility(
        uint256 roundId,
        address user,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external view returns (bool isEligible, bool alreadyClaimed) {
        AirdropRound memory round = airdropRounds[roundId];
        bytes32 leaf = MerkleProver.getHash(user, amount);
        
        isEligible = round.active && 
                    block.timestamp >= round.startTime && 
                    block.timestamp <= round.endTime &&
                    MerkleProver.verify(round.merkleRoot, leaf, merkleProof);
        
        alreadyClaimed = claimedLeaves[roundId][leaf];
    }
    
    function getUserClaims(address user) external view returns (ClaimRecord[] memory) {
        return userClaims[user];
    }
    
    function getRoundInfo(uint256 roundId) external view returns (
        bytes32 merkleRoot,
        uint256 totalTokens,
        uint256 claimedTokens,
        uint256 remainingTokens,
        uint256 startTime,
        uint256 endTime,
        bool active,
        string memory description
    ) {
        AirdropRound memory round = airdropRounds[roundId];
        return (
            round.merkleRoot,
            round.totalTokens,
            round.claimedTokens,
            round.totalTokens - round.claimedTokens,
            round.startTime,
            round.endTime,
            round.active,
            round.description
        );
    }
    
    function finalizeRound(uint256 roundId) external onlyOwner {
        AirdropRound storage round = airdropRounds[roundId];
        require(round.active, "Round not active");
        require(block.timestamp > round.endTime, "Round not ended");
        
        round.active = false;
        
        // Reclaim unclaimed tokens
        uint256 unclaimedTokens = round.totalTokens - round.claimedTokens;
        if (unclaimedTokens > 0) {
            require(
                airdropToken.transfer(msg.sender, unclaimedTokens),
                "Token reclaim failed"
            );
        }
        
        emit RoundFinalized(roundId, round.claimedTokens);
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

### Voting System with Merkle Proofs
```solidity
// Voting system with Merkle tree voter eligibility
contract MerkleVoting {
    using MerkleProver for bytes32;
    
    struct Proposal {
        uint256 id;
        string title;
        string description;
        bytes32 voterMerkleRoot;
        uint256 startTime;
        uint256 endTime;
        uint256 yesVotes;
        uint256 noVotes;
        uint256 totalVotingPower;
        bool executed;
        bool passed;
    }
    
    struct Vote {
        address voter;
        bool support;
        uint256 votingPower;
        uint256 timestamp;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(bytes32 => bool)) public hasVoted;
    mapping(uint256 => Vote[]) public proposalVotes;
    uint256 public nextProposalId;
    uint256 public constant VOTING_DURATION = 7 days;
    uint256 public constant EXECUTION_DELAY = 2 days;
    
    event ProposalCreated(uint256 indexed proposalId, string title, bytes32 voterRoot);
    event VoteCast(uint256 indexed proposalId, address indexed voter, bool support, uint256 votingPower);
    event ProposalExecuted(uint256 indexed proposalId, bool passed);
    
    function createProposal(
        string memory title,
        string memory description,
        bytes32 voterMerkleRoot,
        uint256 totalVotingPower
    ) external onlyGovernance returns (uint256 proposalId) {
        proposalId = nextProposalId++;
        
        proposals[proposalId] = Proposal({
            id: proposalId,
            title: title,
            description: description,
            voterMerkleRoot: voterMerkleRoot,
            startTime: block.timestamp,
            endTime: block.timestamp + VOTING_DURATION,
            yesVotes: 0,
            noVotes: 0,
            totalVotingPower: totalVotingPower,
            executed: false,
            passed: false
        });
        
        emit ProposalCreated(proposalId, title, voterMerkleRoot);
    }
    
    function vote(
        uint256 proposalId,
        bool support,
        uint256 votingPower,
        bytes32[] calldata merkleProof
    ) external {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp >= proposal.startTime, "Voting not started");
        require(block.timestamp <= proposal.endTime, "Voting ended");
        require(!proposal.executed, "Proposal already executed");
        
        // Generate leaf for voter eligibility
        bytes32 leaf = MerkleProver.getHash(msg.sender, votingPower);
        
        // Verify voter eligibility
        bool isEligible = MerkleProver.verify(
            proposal.voterMerkleRoot,
            leaf,
            merkleProof
        );
        require(isEligible, "Invalid voting eligibility proof");
        
        // Prevent double voting
        require(!hasVoted[proposalId][leaf], "Already voted");
        hasVoted[proposalId][leaf] = true;
        
        // Record vote
        if (support) {
            proposal.yesVotes += votingPower;
        } else {
            proposal.noVotes += votingPower;
        }
        
        proposalVotes[proposalId].push(Vote({
            voter: msg.sender,
            support: support,
            votingPower: votingPower,
            timestamp: block.timestamp
        }));
        
        emit VoteCast(proposalId, msg.sender, support, votingPower);
    }
    
    function executeProposal(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp > proposal.endTime + EXECUTION_DELAY, "Execution delay not met");
        require(!proposal.executed, "Proposal already executed");
        
        proposal.executed = true;
        
        // Determine if proposal passed (simple majority)
        uint256 totalVotes = proposal.yesVotes + proposal.noVotes;
        proposal.passed = proposal.yesVotes > proposal.noVotes && 
                          totalVotes >= (proposal.totalVotingPower / 4); // 25% quorum
        
        if (proposal.passed) {
            _executeProposalLogic(proposalId);
        }
        
        emit ProposalExecuted(proposalId, proposal.passed);
    }
    
    function verifyVoterEligibility(
        uint256 proposalId,
        address voter,
        uint256 votingPower,
        bytes32[] calldata merkleProof
    ) external view returns (bool isEligible, bool hasAlreadyVoted) {
        Proposal memory proposal = proposals[proposalId];
        bytes32 leaf = MerkleProver.getHash(voter, votingPower);
        
        isEligible = MerkleProver.verify(proposal.voterMerkleRoot, leaf, merkleProof);
        hasAlreadyVoted = hasVoted[proposalId][leaf];
    }
    
    function getProposalResults(uint256 proposalId) external view returns (
        uint256 yesVotes,
        uint256 noVotes,
        uint256 totalVotes,
        uint256 participationRate,
        bool canExecute,
        bool executed,
        bool passed
    ) {
        Proposal memory proposal = proposals[proposalId];
        yesVotes = proposal.yesVotes;
        noVotes = proposal.noVotes;
        totalVotes = yesVotes + noVotes;
        participationRate = (totalVotes * 100) / proposal.totalVotingPower;
        canExecute = block.timestamp > proposal.endTime + EXECUTION_DELAY && !proposal.executed;
        executed = proposal.executed;
        passed = proposal.passed;
    }
    
    function getProposalVotes(uint256 proposalId) external view returns (Vote[] memory) {
        return proposalVotes[proposalId];
    }
    
    function _executeProposalLogic(uint256 proposalId) internal {
        // Implementation would execute the actual proposal logic
    }
    
    modifier onlyGovernance() {
        // Implementation would check governance authorization
        _;
    }
}
```

## Security Considerations

### Proof Validation Security
- **Empty Proof Protection**: Prevents false positives when leaf equals root
- **Deterministic Ordering**: Sorted hash combination prevents manipulation
- **Replay Attack Prevention**: Leaf tracking prevents proof reuse
- **Input Validation**: Proper bounds checking and parameter validation

### Cryptographic Security
- **Standard Algorithms**: Uses keccak256 for cryptographic security
- **Collision Resistance**: Merkle tree structure provides collision resistance
- **Preimage Resistance**: Hash functions prevent reverse engineering
- **Second Preimage Resistance**: Protects against proof forgery

### Implementation Security
- **Pure Functions**: Stateless operations prevent state manipulation
- **Gas Efficiency**: Optimized algorithms prevent DoS attacks
- **Overflow Protection**: Safe arithmetic operations
- **Memory Safety**: Proper array bounds checking

## Gas Optimization

### Verification Efficiency
- **Minimal Storage**: Pure functions with no storage operations
- **Optimized Loops**: Efficient proof iteration
- **Reduced Hashing**: Minimal hash operations per verification
- **Batch Operations**: Support for batch proof verification

### Memory Efficiency
- **Stack Operations**: Minimal memory allocation
- **Efficient Encoding**: Optimized abi.encodePacked usage
- **Proof Size**: Logarithmic proof size relative to tree size
- **Reusable Functions**: Pure functions for maximum reusability

## Error Handling

### Common Errors
- Invalid Merkle proof verification failure
- Empty proof with leaf equal to root
- Array bounds errors in proof iteration
- Hash computation failures

### Best Practices
- Always validate proof arrays before processing
- Check for edge cases (empty proofs, single-element trees)
- Implement proper error messages for debugging
- Use safe arithmetic operations

## Testing Considerations

### Unit Tests
- Proof verification accuracy for various tree sizes
- Edge case handling (empty proofs, single nodes)
- Hash generation consistency
- Security vulnerability testing

### Integration Tests
- Whitelist system integration
- Airdrop distribution workflows
- Voting system integration
- Cross-contract proof verification

### Security Testing
- Proof forgery attempts
- Replay attack prevention
- Edge case exploitation
- Gas consumption analysis

## Related Documentation

- [MultiSaleLib](multi-sale-lib.md) - Multi-token sale whitelist integration
- [Whitelist Systems Guide](../../guides/whitelist-systems.md) - Whitelist implementation patterns
- [Airdrop Guide](../../guides/airdrop-systems.md) - Airdrop distribution best practices
- [Cryptographic Security](../../guides/cryptographic-security.md) - Security considerations
- [Gas Optimization Guide](../../guides/gas-optimization.md) - Performance optimization techniques

---

*This library provides secure and efficient Merkle proof verification utilities for the Gemforce platform, enabling whitelist systems, airdrop distributions, voting mechanisms, and other scenarios requiring cryptographic proof of inclusion with protection against common attack vectors.*