# MerkleProver Library

## Overview

The [`MerkleProver`](../../smart-contracts/libraries/merkle-prover.md) library provides core utilities for Merkle tree proof verification within the Gemforce platform. This library implements secure and gas-efficient Merkle proof validation, enabling whitelist systems, airdrop distributions, and other scenarios requiring cryptographic proof of inclusion in a dataset.

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
// On-chain voting system with Merkle proof voter registration
contract MerkleVoting {
    using MerkleProver for bytes32;
    
    struct Proposal {
        string description;
        uint256 voteCountYay;
        uint256 voteCountNay;
        uint256 totalVotes;
        bytes32 merkleRoot; // Whitelist of eligible voters
        uint256 snapshotBlock;
        uint256 startTime;
        uint256 endTime;
        bool active;
        bool executed;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;
    uint256 public nextProposalId;
    
    event ProposalCreated(uint256 indexed proposalId, string description, bytes32 merkleRoot);
    event VoteCast(uint256 indexed proposalId, address indexed voter, bool support, uint256 votingPower);
    event ProposalExecuted(uint256 indexed proposalId);
    
    function createProposal(
        string memory description,
        bytes32 merkleRoot,
        uint256 snapshotBlock,
        uint256 startTime,
        uint256 endTime
    ) external onlyOwner returns (uint256 proposalId) {
        proposalId = nextProposalId++;
        
        proposals[proposalId] = Proposal({
            description: description,
            voteCountYay: 0,
            voteCountNay: 0,
            totalVotes: 0,
            merkleRoot: merkleRoot,
            snapshotBlock: snapshotBlock,
            startTime: startTime,
            endTime: endTime,
            active: true,
            executed: false
        });
        
        emit ProposalCreated(proposalId, description, merkleRoot);
    }
    
    function castVote(
        uint256 proposalId,
        bool support,
        uint256 votingPower,
        bytes32[] calldata merkleProof
    ) external {
        Proposal storage proposal = proposals[proposalId];
        require(proposal.active, "Proposal not active");
        require(block.timestamp >= proposal.startTime, "Voting not started");
        require(block.timestamp <= proposal.endTime, "Voting ended");
        require(!hasVoted[proposalId][msg.sender], "Already voted on this proposal");
        
        // Verify voter eligibility using Merkle proof
        bytes32 leaf = MerkleProver.getHash(msg.sender, votingPower);
        bool isValid = MerkleProver.verify(
            proposal.merkleRoot,
            leaf,
            merkleProof
        );
        require(isValid, "Invalid voter proof");
        
        hasVoted[proposalId][msg.sender] = true;
        
        if (support) {
            proposal.voteCountYay += votingPower;
        } else {
            proposal.voteCountNay += votingPower;
        }
        proposal.totalVotes += votingPower;
        
        emit VoteCast(proposalId, msg.sender, support, votingPower);
    }
    
    function executeProposal(uint256 proposalId) external onlyOwner {
        Proposal storage proposal = proposals[proposalId];
        require(!proposal.executed, "Proposal already executed");
        require(block.timestamp > proposal.endTime, "Voting not ended");
        require(proposal.active, "Proposal not active for execution");
        
        // Example: simple majority
        require(proposal.voteCountYay > proposal.voteCountNay, "Proposal not passed");
        
        proposal.executed = true;
        proposal.active = false; // Deactivate after execution
        
        emit ProposalExecuted(proposalId);
    }
    
    function getVotingStatus(uint256 proposalId) external view returns (
        string memory description,
        uint256 voteYay,
        uint256 voteNay,
        uint256 totalVotes,
        uint256 requiredVotesForPass,
        bool active,
        bool executed
    ) {
        Proposal memory proposal = proposals[proposalId];
        description = proposal.description;
        voteYay = proposal.voteCountYay;
        voteNay = proposal.voteCountNay;
        totalVotes = proposal.totalVotes;
        requiredVotesForPass = (proposal.totalVotes / 2) + 1; // Simple majority example
        active = proposal.active;
        executed = proposal.executed;
    }
    
    modifier onlyOwner() {
        // Implementation would check ownership
        _;
    }
}
```

## Security Considerations

### Proof Validation Security
- **Correct Root**: Always ensure the Merkle root is correct and from a trusted source.
- **Uniqueness of Leaves**: Prevent double-claiming by marking used leaves (hash of user+amount).
- **Correct Hashing**: Ensure leaf node hashing matches the tree generation.

### Cryptographic Security
- **Strong Hashing**: Use collision-resistant hash functions (e.g., keccak256).
- **No Predictable Randomness**: Avoid on-chain randomness for critical parameters.

### Implementation Security
- **Access Control**: Secure functions that set Merkle roots or control airdrop parameters.
- **Reentrancy Protection**: Critical for functions that transfer tokens.

## Gas Optimization

### Verification Efficiency
- The `verify()` function is highly optimized for gas, computing hashes iteratively.

### Memory Efficiency
- `proof` array is passed as `memory` to prevent unnecessary storage writes.

## Error Handling

### Common Errors
- `Invalid Merkle proof`: The provided proof does not validate against the root.
- `Proof already used`: The leaf has already been claimed/used.
- `Not active`: The sale/airdrop is not active or has ended.
- `Insufficient allocation`: Attempting to claim more than whitelisted amount.

## Best Practices

### Merkle Tree Generation
- Generate Merkle trees off-chain using a secure and reliable process.
- The Merkle root must be committed on-chain in a timely and secure manner.

### Leaf Data
- Standardize the data structure for leaves (e.g., `keccak256(abi.encodePacked(address, amount))`).
- Ensure the off-chain system that generates leaves and proofs matches the on-chain logic.

### Integration Checklist
- Verify proof generation and verification across all environments (local, testnet, production).
- Clearly communicate to users how to generate their participation data (e.g., wallet address + allocation).

## Related Documentation

- [Multi Sale Facet](../../smart-contracts/facets/multi-sale-facet.md) - For token sales integration.
- [SDK & Libraries: Blockchain Utilities](../../sdk-libraries/blockchain.md) - General blockchain interaction utilities.
- [Developer Guides: Automated Testing Setup](../../developer-guides/automated-testing-setup.md)