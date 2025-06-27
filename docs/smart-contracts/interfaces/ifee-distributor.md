# IFeeDistributor Interface

The `IFeeDistributor` interface defines the standard for smart contracts responsible for distributing collected fees or revenues to multiple recipients based on predefined percentages or shares. This is a critical component for managing revenue sharing in various decentralized applications within the Gemforce ecosystem.

## Overview

`IFeeDistributor` provides:

- **Configurable Shares**: Define distribution percentages for multiple recipients.
- **Automated Distribution**: Functions to trigger and manage fee distribution.
- **Recipient Management**: Add, update, and remove recipients with their shares.
- **Event Logging**: Comprehensive event tracking for all distribution activities.

## Key Features

### Distribution Management
- **Manual Trigger**: Allow authorized parties to initiate distribution.
- **Automated Trigger (Optional)**: Can be integrated with external mechanisms for time-based or volume-based distribution.
- **Share Calculation**: Accurately calculate each recipient's share.

### Recipient and Share Management
- **Add Recipient**: Onboard new recipients with their respective shares.
- **Update Share**: Adjust existing recipient shares.
- **Remove Recipient**: Remove recipients from the distribution list.

### Fund Handling
- **Multi-Token Support**: Can be designed to distribute various ERC20 tokens or native currency (ETH).
- **Residual Handling**: Mechanisms to manage any remaining dust or undistributed funds.

## Interface Definition

```solidity
interface IFeeDistributor {
    // Events
    event FundsReceived(
        address indexed token,
        uint256 amount,
        address indexed sender
    );
    event FeesDistributed(
        address indexed token,
        uint256 totalAmount,
        uint256 leftOverAmount,
        address indexed distributor
    );
    event RecipientAdded(
        address indexed recipient,
        uint256 share,
        address indexed manager
    );
    event RecipientUpdated(
        address indexed recipient,
        uint256 oldShare,
        uint256 newShare,
        address indexed manager
    );
    event RecipientRemoved(
        address indexed recipient,
        uint256 share,
        address indexed manager
    );
    event FeeCollectorUpdated(
        address indexed oldCollector,
        address indexed newCollector
    );

    // Structs
    struct Recipient {
        address addr;
        uint256 share; // Basis points (e.g., 100 = 1%)
        uint256 lastDistributedAmount; // Amount last sent to this recipient
    }

    // Core Functions
    function distributeFees(address token) external returns (uint256 distributedAmount);
    function updateRecipient(address recipientAddr, uint256 newShare) external;
    function addRecipient(address recipientAddr, uint256 share) external;
    function removeRecipient(address recipientAddr) external;
    function setFeeCollector(address collector) external;

    // View Functions
    function getTotalShare() external view returns (uint256);
    function getRecipientShare(address recipientAddr) external view returns (uint256);
    function getRecipientCount() external view returns (uint256);
    function getRecipientAddress(uint256 index) external view returns (address);
    function getFeeCollector() external view returns (address);
    function getBalance(address token) external view returns (uint256);
    function getRecipientDetails(address recipientAddr) external view returns (Recipient memory);
}
```

## Core Functions

### distributeFees()
Initiates the distribution of collected fees for a specific token to all registered recipients based on their shares.

**Parameters:**
- `token`: The address of the ERC20 token to distribute. If `address(0)` is used, it refers to the native blockchain currency (e.g., ETH).

**Returns:**
- `uint256`: The total amount of the token that was successfully distributed.

**Usage:**
```solidity
// Distribute ETH
feeDistributor.distributeFees(address(0));

// Distribute DAI (ERC20 token)
feeDistributor.distributeFees(DAI_TOKEN_ADDRESS);
```

### addRecipient()
Adds a new address to the list of fee recipients with a specified share. Shares are typically defined in basis points (e.g., 100 = 1%).

**Parameters:**
- `recipientAddr`: The address of the new recipient.
- `share`: The share percentage of the recipient in basis points.

**Access Control:**
- Typically restricted to the contract owner or an authorized `feeCollector`.

### updateRecipient()
Updates the share percentage of an existing recipient.

**Parameters:**
- `recipientAddr`: The address of the recipient whose share is to be updated.
- `newShare`: The new share percentage for the recipient.

### removeRecipient()
Removes an address from the list of fee recipients.

**Parameters:**
- `recipientAddr`: The address of the recipient to be removed.

## Implementation Example

### Basic FeeDistributor Contract

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract FeeDistributor is IFeeDistributor, Ownable {
    using SafeMath for uint256;

    uint256 public constant TOTAL_BASIS_POINTS = 10000; // 100%

    // Mapping from recipient address to their share (in basis points)
    mapping(address => uint256) private _recipientShares;
    // Array to maintain order and iterate over recipients
    address[] private _recipients;
    // Total sum of all recipient shares
    uint256 private _totalShares;

    address private _feeCollector;

    constructor() {
        _feeCollector = msg.sender;
    }

    modifier onlyFeeCollector() {
        require(msg.sender == _feeCollector, "Only fee collector can call this function");
        _;
    }

    receive() external payable {
        emit FundsReceived(address(0), msg.value, msg.sender);
    }

    // Handles ERC20 transfers directly to the contract
    function onERC20Received(address, uint256 amount) external returns (bool) {
        emit FundsReceived(msg.sender, amount, tx.origin); // Assume msg.sender is the ERC20 token contract
        return true;
    }

    function distributeFees(address token) external override returns (uint256 distributedAmount) {
        require(_recipients.length > 0, "No recipients to distribute to");
        require(_totalShares <= TOTAL_BASIS_POINTS, "Invalid total shares configuration");

        uint256 contractBalance;
        if (token == address(0)) {
            contractBalance = address(this).balance;
        } else {
            contractBalance = IERC20(token).balanceOf(address(this));
        }
        require(contractBalance > 0, "No funds to distribute");

        distributedAmount = 0;
        uint256 remainingAmount = contractBalance;

        for (uint256 i = 0; i < _recipients.length; i++) {
            address recipientAddr = _recipients[i];
            uint256 share = _recipientShares[recipientAddr];
            
            // Calculate amount for current recipient
            uint256 amountToSend = contractBalance.mul(share).div(TOTAL_BASIS_POINTS);

            if (amountToSend > 0) {
                if (token == address(0)) {
                    // Send ETH
                    payable(recipientAddr).transfer(amountToSend);
                } else {
                    // Send ERC20
                    IERC20(token).transfer(recipientAddr, amountToSend);
                }
                distributedAmount = distributedAmount.add(amountToSend);
                remainingAmount = remainingAmount.sub(amountToSend);
            }
        }
        emit FeesDistributed(token, distributedAmount, remainingAmount, msg.sender);
    }

    function addRecipient(address recipientAddr, uint256 share) external override onlyFeeCollector {
        require(recipientAddr != address(0), "Cannot add zero address");
        require(_recipientShares[recipientAddr] == 0, "Recipient already exists");
        require(_totalShares.add(share) <= TOTAL_BASIS_POINTS, "Total shares exceed 100%");

        _recipientShares[recipientAddr] = share;
        _recipients.push(recipientAddr);
        _totalShares = _totalShares.add(share);
        emit RecipientAdded(recipientAddr, share, msg.sender);
    }

    function updateRecipient(address recipientAddr, uint256 newShare) external override onlyFeeCollector {
        require(recipientAddr != address(0), "Cannot update zero address");
        require(_recipientShares[recipientAddr] != 0, "Recipient does not exist");
        
        uint256 oldShare = _recipientShares[recipientAddr];
        require(_totalShares.sub(oldShare).add(newShare) <= TOTAL_BASIS_POINTS, "Total shares exceed 100%");

        _recipientShares[recipientAddr] = newShare;
        _totalShares = _totalShares.sub(oldShare).add(newShare);
        emit RecipientUpdated(recipientAddr, oldShare, newShare, msg.sender);
    }

    function removeRecipient(address recipientAddr) external override onlyFeeCollector {
        require(recipientAddr != address(0), "Cannot remove zero address");
        uint256 shareToRemove = _recipientShares[recipientAddr];
        require(shareToRemove > 0, "Recipient does not exist");

        delete _recipientShares[recipientAddr];
        _totalShares = _totalShares.sub(shareToRemove);

        // Remove from dynamic array by swapping with last element and popping
        for (uint256 i = 0; i < _recipients.length; i++) {
            if (_recipients[i] == recipientAddr) {
                _recipients[i] = _recipients[_recipients.length - 1];
                _recipients.pop();
                break;
            }
        }
        emit RecipientRemoved(recipientAddr, shareToRemove, msg.sender);
    }

    function setFeeCollector(address collector) external override onlyOwner {
        require(collector != address(0), "New fee collector cannot be the zero address");
        emit FeeCollectorUpdated(_feeCollector, collector);
        _feeCollector = collector;
    }

    function getTotalShare() external view override returns (uint256) {
        return _totalShares;
    }

    function getRecipientShare(address recipientAddr) external view override returns (uint256) {
        return _recipientShares[recipientAddr];
    }

    function getRecipientCount() external view override returns (uint256) {
        return _recipients.length;
    }

    function getRecipientAddress(uint256 index) external view override returns (address) {
        require(index < _recipients.length, "Index out of bounds");
        return _recipients[index];
    }

    function getFeeCollector() external view override returns (address) {
        return _feeCollector;
    }

    function getBalance(address token) external view override returns (uint256) {
        if (token == address(0)) {
            return address(this).balance;
        } else {
            return IERC20(token).balanceOf(address(this));
        }
    }

    function getRecipientDetails(address recipientAddr) external view override returns (Recipient memory) {
        return Recipient({
            addr: recipientAddr,
            share: _recipientShares[recipientAddr],
            lastDistributedAmount: 0 // This would require more complex state tracking
        });
    }
}
```

## Security Considerations

### Access Control
- **`onlyOwner`**: Critical administrative functions (like changing the fee collector) should be restricted to the contract owner.
- **`onlyFeeCollector`**: Functions modifying recipient lists or shares should be restricted to an authorized fee collector role.

### Fund Handling
- **Reentrancy**: Implement reentrancy guards for `distributeFees` if external calls are made that could re-enter the contract (though direct token transfers are less prone to this).
- **Exact Amounts**: Ensure precise calculations to avoid sending unintended amounts or leaving dust. Using SafeMath is crucial.
- **Eth/Token Differences**: Correctly handle native currency (ETH) transfers vs. ERC20 token transfers.
- **Dust Management**: Consider a mechanism to handle small remaining amounts (dust) if total shares don't perfectly sum up to `TOTAL_BASIS_POINTS` or if precision issues arise.

### Configuration
- **Share Validation**: Prevent total shares from exceeding `TOTAL_BASIS_POINTS` (100%) to avoid potential over-distribution.
- **Zero Address Checks**: Always validate non-zero addresses for recipients and collectors.

## Best Practices

### Shares Configuration
1. **Basis Points**: Use basis points (e.g., 10,000 for 100%) for share representation to avoid floating-point errors common in Solidity.
2. **Sum to 100%**: Ensure that the sum of all recipient shares equals `TOTAL_BASIS_POINTS` to guarantee full distribution and no residual funds left in the contract (unless intentional).

### Modularity
1. **Separation of Concerns**: Keep fee distribution logic separate from other core application logic.
2. **Upgradeable**: Design the contract to be upgradeable if shares or recipients might change frequently or if new distribution logic is anticipated.

### Gas Efficiency
1. **Array Iteration**: Be mindful of gas costs for iterating over a large number of recipients; large arrays can become expensive. Consider a pull-based mechanism for recipients to claim their funds if recipient count is very high.

## Integration Examples

### Frontend Integration (React/Web3Modal)

```typescript
import { ethers } from 'ethers';
import feeDistributorABI from './FeeDistributor.json'; // ABI of the FeeDistributor contract

const FEE_DISTRIBUTOR_ADDRESS = "0x..."; // Deploy address of your FeeDistributor contract

const getProvider = () => new ethers.providers.Web3Provider(window.ethereum);
const getSigner = () => getProvider().getSigner();
const getFeeDistributorContract = () => new ethers.Contract(FEE_DISTRIBUTOR_ADDRESS, feeDistributorABI, getSigner());

async function handleDistributeFees(tokenAddress: string) {
    try {
        const feeDistributor = getFeeDistributorContract();
        const tx = await feeDistributor.distributeFees(tokenAddress);
        await tx.wait();
        alert("Fees distributed successfully!");
    } catch (error) {
        console.error("Error distributing fees:", error);
        alert("Failed to distribute fees.");
    }
}

async function handleAddRecipient(recipientAddress: string, share: number) {
    try {
        const feeDistributor = getFeeDistributorContract();
        // Assuming share is in basis points, e.g., 2500 for 25%
        const tx = await feeDistributor.addRecipient(recipientAddress, share);
        await tx.wait();
        alert("Recipient added successfully!");
    } catch (error) {
        console.error("Error adding recipient:", error);
        alert("Failed to add recipient.");
    }
}

// Example usage in a React component
// <button onClick={() => handleDistributeFees("0x0000000000000000000000000000000000000000")}>Distribute ETH Fees</button>
// <button onClick={() => handleAddRecipient("0xRecipientAddress", 1000)}>Add 10% Recipient</button>
```

### Backend Integration (Node.js/Ethers.js)

```javascript
const { ethers } = require("ethers");
const FeeDistributorABI = require("./FeeDistributor.json").abi;

const provider = new ethers.JsonRpcProvider("YOUR_RPC_URL");
const wallet = new ethers.Wallet("YOUR_PRIVATE_KEY", provider);
const feeDistributorAddress = "0x..."; // Your deployed FeeDistributor contract address

const feeDistributorContract = new ethers.Contract(feeDistributorAddress, FeeDistributorABI, wallet);

async function performFeeDistribution(tokenAddress) {
    try {
        console.log(`Initiating fee distribution for ${tokenAddress === ethers.ZeroAddress ? "ETH" : "ERC20 token"}...`);
        const tx = await feeDistributorContract.distributeFees(tokenAddress);
        await tx.wait();
        console.log(`Distribution transaction successful: ${tx.hash}`);
    } catch (error) {
        console.error("Error during fee distribution:", error);
        throw error;
    }
}

async function addNewRecipient(address, share) {
    try {
        console.log(`Adding new recipient ${address} with share ${share}...`);
        const tx = await feeDistributorContract.addRecipient(address, share);
        await tx.wait();
        console.log(`Recipient added transaction successful: ${tx.hash}`);
    } catch (error) {
        console.error("Error adding new recipient:", error);
        throw error;
    }
}

// Example usage
// performFeeDistribution(ethers.ZeroAddress); // Distribute ETH
// addNewRecipient("0xNewRecipientAddress", 500); // Add a recipient with 5% share
```

## Related Documentation

- [Fee Distributor Library](../libraries/fee-distributor-lib.md)
- [ERC20 Standard](https://eips.ethereum.org/EIPS/eip-20)
- [Ownable Contract](https://docs.openzeppelin.com/contracts/5.x/api/access#Ownable)
- [Safemath Library](https://docs.openzeppelin.com/contracts/2.x/api/math#SafeMath)

## Standards Compliance

- **ERC20**: Compatible with ERC20 tokens for distribution.
- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.