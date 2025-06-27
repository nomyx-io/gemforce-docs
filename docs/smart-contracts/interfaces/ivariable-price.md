# IVariablePrice Interface

The `IVariablePrice` interface defines the standard for smart contracts that implement dynamic pricing mechanisms within the Gemforce ecosystem. This interface allows for flexible and algorithmic determination of token or asset prices based on factors such as demand, time, supply, or other custom parameters.

## Overview

`IVariablePrice` provides:

- **Dynamic Pricing**: Functions to calculate prices based on various input parameters.
- **Pricing Strategy Management**: Mechanisms to select and configure different pricing models (e.g., fixed, linear, exponential, logarithmic).
- **Flexibility**: Supports multiple pricing scenarios for NFTs, tokens, or services.
- **Queryability**: Allows external contracts or off-chain systems to query current prices.

## Key Features

### Price Calculation
- **`getPrice()`**: Core function to obtain the current price given specific context (e.g., amount, current block, accumulated sales).
- **Strategy-Dependent Inputs**: The function parameters will vary based on the chosen pricing strategy.

### Strategy Management
- **Set Strategy**: Option to select and configure a pricing strategy.
- **Strategy Parameters**: Define parameters unique to each strategy (e.g., initial price, curve steepness, time decay).

### Integration
- Designed to be easily integrated into marketplaces, sale contracts, or any system requiring dynamic pricing.

## Interface Definition

```solidity
interface IVariablePrice {
    // Events
    event PriceStrategyUpdated(
        bytes32 indexed strategyId,
        string indexed strategyName,
        address indexed configurator
    );
    event PriceComponentUpdated(
        bytes32 indexed strategyId,
        string indexed componentName,
        bytes newValue,
        address indexed updater
    );

    // Struct to define a pricing strategy
    struct PricingStrategyConfig {
        string name;
        uint256 strategyType; // e.g., 0 for Fixed, 1 for Linear, 2 for Geometric, 3 for Logarithmic;
        bytes parameters; // ABI-encoded parameters specific to the strategy
        bool active;
    }

    // Core Pricing Function
    function getPrice(
        bytes32 strategyId,
        uint256 currentSupply, // Current items sold or tokens minted
        uint256 quantity,      // Quantity being purchased
        uint256 timeElapsed,   // Time since sale started (optional)
        uint256 initialPrice   // Base price or reference price
    ) external view returns (uint256 price);

    // Strategy Management Functions
    function setPricingStrategy(
        string calldata name,
        uint256 strategyType,
        bytes calldata parameters
    ) external;

    function activatePricingStrategy(bytes32 strategyId) external;
    function deactivatePricingStrategy(bytes32 strategyId) external;

    // View Functions
    function getPricingStrategyConfig(bytes32 strategyId) external view returns (PricingStrategyConfig memory);
    function getActiveStrategyId() external view returns (bytes32); // For contracts that only manage one active strategy
    // Function to get price for a specific strategy directly (if applicable)
    function getPriceForStrategy(
        bytes32 strategyId,
        uint256 currentSupply,
        uint256 quantity,
        uint256 timeElapsed,
        uint256 initialPrice
    ) external view returns (uint256 price);
}
```

## Core Functions

### getPrice()
Calculates the price for a specified quantity given the current state of the sale or system.

**Parameters:**
- `strategyId`: A unique identifier for the pricing strategy to use.
- `currentSupply`: The current quantity of items sold or tokens minted (used for supply-dependent curves).
- `quantity`: The number of items or tokens for which the price is being queried.
- `timeElapsed`: The time elapsed since the sale or pricing mechanism started (used for time-dependent curves).
- `initialPrice`: A base or starting price for the calculation.

**Returns:**
- `uint256`: The calculated total price for the given `quantity`.

**Usage:**
```solidity
// Get price based on a specific strategy ID
uint256 currentPrice = variablePriceContract.getPrice(
    myStrategyId,
    totalMintedNFTs, // e.g., 100
    1, // quantity to buy
    block.timestamp - saleStartTime,
    1 ether // base price
);
```

### setPricingStrategy()
Defines or updates a new pricing strategy. The `parameters` field is `abi.encoded` data specific to the chosen `strategyType`.

**Parameters:**
- `name`: A descriptive name for the strategy (e.g., "LinearDutchAuction").
- `strategyType`: An enum or integer representing the type of pricing curve (e.g., `0` for Fixed, `1` for Linear).
- `parameters`: Bytes containing ABI-encoded parameters specific to the chosen strategy (e.g., `slopeForLinear`, `decayRateForExponential`).

**Access Control:**
- Typically restricted to the contract owner or an authorized administrator.

## Implementation Example

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract VariablePriceEngine is IVariablePrice, Ownable {
    using SafeMath for uint256;

    // Defines different pricing curve types
    enum StrategyType {
        FIXED,              // 0: Price is always fixed
        LINEAR_DECREASE,    // 1: Price decreases linearly over time or supply
        EXPONENTIAL_DECREASE, // 2: Price decreases exponentially over time or supply
        LOGARITHMIC_INCREASE, // 3: Price increases logarithmically with supply
        BATCH_AUCTION       // 4: For specific batch processing, often simpler
    }

    uint256 public constant TOTAL_BASIS_POINTS = 10000; // For percentage calculations (100% = 10000 bp)

    // Mapping from strategyId to its configuration
    mapping(bytes32 => PricingStrategyConfig) private _pricingStrategies;
    // Store strategy IDs for enumeration
    bytes32[] private _strategyIds;
    // Optional: a single active strategy
    bytes32 private _activeStrategyId;

    constructor() {
        // Owner set to deployer by default due to Ownable
        // A default fixed price strategy could be set here
    }

    function setPricingStrategy(
        string calldata name,
        uint256 strategyType,
        bytes calldata parameters
    ) external override onlyOwner {
        bytes32 strategyId = keccak256(abi.encodePacked(name, strategyType));
        // Ensure strategyType is valid
        require(strategyType <= uint256(StrategyType.BATCH_AUCTION), "Invalid strategy type");

        _pricingStrategies[strategyId] = PricingStrategyConfig({
            name: name,
            strategyType: strategyType,
            parameters: parameters,
            active: true // Active by default when set
        });
        
        bool found = false;
        for(uint256 i=0; i < _strategyIds.length; i++) {
        	if(_strategyIds[i] == strategyId) {
        		found = true;
        		break;
        	}
        }
        if(!found) {
        	_strategyIds.push(strategyId);
        }

        emit PriceStrategyUpdated(strategyId, name, msg.sender);
    }

    function activatePricingStrategy(bytes32 strategyId) external override onlyOwner {
        require(_pricingStrategies[strategyId].name.length > 0, "Strategy not found"); // Check if strategy exists
        _pricingStrategies[strategyId].active = true;
        _activeStrategyId = strategyId; // Set as global active strategy if applicable
        emit PriceComponentUpdated(strategyId, "active", abi.encode(true), msg.sender);
    }

    function deactivatePricingStrategy(bytes32 strategyId) external override onlyOwner {
        require(_pricingStrategies[strategyId].name.length > 0, "Strategy not found");
        _pricingStrategies[strategyId].active = false;
        if (_activeStrategyId == strategyId) {
            _activeStrategyId = bytes32(0); // Clear active strategy if deactivated
        }
        emit PriceComponentUpdated(strategyId, "active", abi.encode(false), msg.sender);
    }

    function getPrice(
        bytes32 strategyId,
        uint256 currentSupply,
        uint256 quantity,
        uint256 timeElapsed,
        uint256 initialPrice
    ) external view override returns (uint256 price) {
        PricingStrategyConfig memory config = _pricingStrategies[strategyId];
        require(config.name.length > 0 && config.active, "Strategy not found or inactive");

        price = _calculatePrice(
            config.strategyType,
            config.parameters,
            currentSupply,
            quantity,
            timeElapsed,
            initialPrice
        );
        return price;
    }

    function getPriceForStrategy(
        bytes32 strategyId,
        uint256 currentSupply,
        uint256 quantity,
        uint256 timeElapsed,
        uint256 initialPrice
    ) external view override returns (uint256 price) {
        return getPrice(strategyId, currentSupply, quantity, timeElapsed, initialPrice);
    }

    function _calculatePrice(
        uint256 strategyType,
        bytes memory parameters,
        uint256 currentSupply,
        uint256 quantity,
        uint256 timeElapsed,
        uint256 initialPrice
    ) internal view returns (uint256) {
        if (strategyType == uint256(StrategyType.FIXED)) {
            // Parameters: none needed for fixed, or a single uint256 for the fixed price itself
            return initialPrice.mul(quantity);
        } else if (strategyType == uint256(StrategyType.LINEAR_DECREASE)) {
            // Parameters: (uint256 decayRatePerUnit, uint256 floorPrice)
            (uint256 decayRatePerUnit, uint256 floorPrice) = abi.decode(parameters, (uint256, uint256));
            uint256 effectivePrice = initialPrice.sub(currentSupply.mul(decayRatePerUnit));
            effectivePrice = effectivePrice > floorPrice ? effectivePrice : floorPrice;
            return effectivePrice.mul(quantity);
        } else if (strategyType == uint256(StrategyType.EXPONENTIAL_DECREASE)) {
            // Parameters: (uint256 decayFactor, uint256 floorPrice)
            // Example: initialPrice * (decayFactor / 10000)^currentSupply
            // decayFactor is in basis points, e.g., 9900 for 99% (0.99)
            (uint256 decayFactor, uint256 floorPrice) = abi.decode(parameters, (uint256, uint256));
            uint256 priceAtCurrentSupply = initialPrice;
            for (uint256 i = 0; i < currentSupply; i++) {
                priceAtCurrentSupply = priceAtCurrentSupply.mul(decayFactor).div(TOTAL_BASIS_POINTS); 
            }
            priceAtCurrentSupply = priceAtCurrentSupply > floorPrice ? priceAtCurrentSupply : floorPrice;
            return priceAtCurrentSupply.mul(quantity);
        } else if (strategyType == uint256(StrategyType.LOGARITHMIC_INCREASE)) {
            // Parameters: (uint256 growthFactor, uint256 capPrice)
            // This is complex on-chain; often approximated or done off-chain.
            // Placeholder: simple growth based on quantity
            (uint256 growthFactor, uint256 capPrice) = abi.decode(parameters, (uint256, uint256));
            // Mathematical log function is not easily available in Solidity.
            // A simplified approximation or a lookup table would be needed for a real scenario.
            // For example, an approximation: price increases with currentSupply's square root
            // using Newton's method for sqrt, or a simplified linear increase to approximate.
            
            // This is a simplified example of logarithmic-like growth:
            // Price increases by (growthFactor * sqrt(currentSupply)) units (growthFactor in basis points)
            uint256 currentPrice = initialPrice;
            if (currentSupply > 0) {
                // Approximate square root (e.g., using integer square root if available or simple loop)
                uint256 sqrtSupply = currentSupply;
                if (currentSupply > 1) { // Simple integer sqrt approximation
                    uint256 z = currentSupply.add(1).div(2);
                    uint256 y = currentSupply;
                    while (z < y) {
                        y = z;
                        z = (currentSupply.div(z).add(z)).div(2);
                    }
                    sqrtSupply = y;
                }
                currentPrice = currentPrice.add(sqrtSupply.mul(growthFactor).div(TOTAL_BASIS_POINTS));
            }
            
            currentPrice = currentPrice < capPrice ? currentPrice : capPrice;
            return currentPrice.mul(quantity);

        } else if (strategyType == uint256(StrategyType.BATCH_AUCTION)) {
            // Specific logic for batch auctions, might consider lowest bid in range
            // or fixed price for batches.
            return initialPrice.mul(quantity);
        }
        else {
            revert("Unknown pricing strategy type");
        }
    }

    function getPricingStrategyConfig(bytes32 strategyId) external view override returns (PricingStrategyConfig memory) {
        return _pricingStrategies[strategyId];
    }

    function getActiveStrategyId() external view override returns (bytes32) {
        return _activeStrategyId;
    }

    function getAllStrategyIds() external view returns (bytes32[] memory) {
        return _strategyIds;
    }
}
```

## Security Considerations

### Access Control
- **`onlyOwner`**: Functions that modify or configure pricing strategies (`setPricingStrategy`, `activatePricingStrategy`, `deactivatePricingStrategy`) must be restricted to the contract owner or an authorized administrator. Unauthorized changes to pricing logic could lead to financial exploits.

### Mathematical Precision
- **Fixed-Point Arithmetic**: Solidity does not natively support floating-point numbers. All price calculations must use fixed-point arithmetic, typically by scaling all values by a large constant (e.g., `10**18` for `WAD` or `10**4` for basis points) and then performing division/multiplication with `SafeMath`.
- **`SafeMath`**: Crucial for all arithmetic operations to prevent overflow and underflow vulnerabilities.

### Strategy Implementation
- **Complexity**: Highly complex mathematical functions (e.g., true logarithms, exponentials) are difficult and gas-expensive to implement accurately on-chain. Approximations or off-chain calculations validated by the contract might be necessary.
- **Precision Loss**: Be aware of potential precision loss when performing divisions and multiplications in fixed-point arithmetic. Choose appropriate scaling factors.

## Best Practices

### Modular Strategies
1. **Separation of Concerns**: Each pricing strategy (linear, exponential, fixed, etc.) should ideally be a distinct, self-contained unit within the contract or even a separate library, invoked by a dispatcher-like `getPrice` function.
2. **Parameters**: Clearly define and document the `abi.encoded` `parameters` required for each strategy type.

### Oracles for External Data
1. **External Factors**: If pricing depends on external factors (e.g., real-time market data, off-chain computations), use secure oracles (like Chainlink) to feed this data to the contract.

### Gas Efficiency
1. **Pre-computation**: If a price curve is static or changes infrequently, consider pre-computing checkpoints or a lookup table on-chain to reduce gas costs during `getPrice` calls.
2. **Pull vs. Push**: If prices are pushed from an external source, ensure that the pushing mechanism is secure and not easily manipulable.

## Integration Examples

### Frontend Integration (TypeScript via Ethers.js)

```typescript
import { ethers, Contract } from 'ethers';
import VariablePriceABI from './VariablePriceEngine.json'; // ABI for IVariablePrice

const VARIABLE_PRICE_ADDRESS = "0x..."; // Deployed VariablePriceEngine contract address

// Assumes 'provider' is already set up from Web3Modal, Metamask, etc.
const getPriceContract = (signer?: ethers.Signer) => {
    return new Contract(VARIABLE_PRICE_ADDRESS, VariablePriceABI, signer || new ethers.providers.Web3Provider(window.ethereum));
};

async function getPriceFromContract(
    strategyId: string, // hex string
    currentSupply: number,
    quantity: number,
    timeElapsed: number,
    initialPrice: ethers.BigNumberish
): Promise<ethers.BigNumber> {
    try {
        const priceContract = getPriceContract();
        const price = await priceContract.getPrice(
            strategyId,
            currentSupply,
            quantity,
            timeElapsed,
            initialPrice
        );
        return price;
    } catch (error) {
        console.error("Error getting price:", error);
        throw error;
    }
}

async function setLinearDecreaseStrategy(
    name: string,
    decayRatePerUnit: ethers.BigNumberish,
    floorPrice: ethers.BigNumberish
): Promise<void> {
    try {
        const priceContract = getPriceContract(getPriceContract().signer as ethers.Signer);
        const strategyType_LINEAR_DECREASE = 1; // Corresponds to enum in Solidity
        const parameters = ethers.utils.defaultAbiCoder.encode(
            ["uint256", "uint256"], // Match _calculatePrice's abi.decode for LINEAR_DECREASE
            [decayRatePerUnit, floorPrice]
        );
        const tx = await priceContract.setPricingStrategy(
            name,
            strategyType_LINEAR_DECREASE,
            parameters
        );
        await tx.wait();
        alert(`Linear Decrease strategy '${name}' set!`);
    } catch (error) {
        console.error("Error setting strategy:", error);
        alert("Failed to set pricing strategy.");
    }
}

// Example usage
// const myStrategyId = "0x..."; // Get this from backend or previous setup
// const calculatedPrice = await getPriceFromContract(myStrategyId, 100, 1, 3600, ethers.utils.parseEther("1"));
// console.log("Calculated Price:", ethers.utils.formatEther(calculatedPrice));

// setLinearDecreaseStrategy("NFT_Price_Decay", ethers.utils.parseUnits("0.001", "ether"), ethers.utils.parseEther("0.1")); // 0.001 ETH decay per unit, 0.1 ETH floor
```

### Backend Integration (Node.js for a pricing service)

```javascript
const Web3 = require('web3');
const VariablePriceABI = require('./VariablePriceEngine.json').abi;

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const variablePriceAddress = '0x...'; // Your deployed VariablePriceEngine contract address
const adminPrivateKey = 'YOUR_ADMIN_PRIVATE_KEY'; // Private key of the account authorized to set strategies

const variablePriceContract = new web3.eth.Contract(VariablePriceABI, variablePriceAddress);
const adminAccount = web3.eth.accounts.privateKeyToAccount(adminPrivateKey);
web3.eth.accounts.wallet.add(adminAccount);

async function getConfiguredPrice(strategyName, currentSupply, quantity, timeElapsed, initialPrice) {
    try {
        // Need to derive strategyId from name and type in JS if not static
        // For simplicity, let's assume a known strategyId for a fixed price strategy
        const FIXED_PRICE_STRATEGY_ID = web3.utils.keccak256(web3.eth.abi.encodePacked("FixedPrice", 0)); // Assuming FixedPrice strategy type is 0

        const price = await variablePriceContract.methods.getPrice(
            FIXED_PRICE_STRATEGY_ID,
            currentSupply,
            quantity,
            timeElapsed,
            initialPrice
        ).call();

        console.log(`Price for ${quantity} items: ${web3.utils.fromWei(price, 'ether')} ETH`);
        return price;
    } catch (error) {
        console.error("Backend: Error getting price:", error);
        throw error;
    }
}

async function configureExponentialDecreaseStrategy(strategyName, decayFactor, floorPrice) {
    try {
        const strategyType_EXPONENTIAL_DECREASE = 2; // Corresponds to enum in Solidity
        const parameters = web3.eth.abi.encodeParameters(
            ['uint256', 'uint256'], // decayFactor (basis points, 10000 = 100%), floorPrice
            [decayFactor, floorPrice]
        );

        const tx = variablePriceContract.methods.setPricingStrategy(
            strategyName,
            strategyType_EXPONENTIAL_DECREASE,
            parameters
        );
        const gasLimit = await tx.estimateGas({ from: adminAccount.address });
        const receipt = await tx.send({ from: adminAccount.address, gas: gasLimit });
        console.log(`Exponential Decrease strategy '${strategyName}' configured. Tx Hash: ${receipt.transactionHash}`);
        return receipt;
    } catch (error) {
        console.error("Backend: Error configuring strategy:", error);
        throw error;
    }
}

// Example usage
// getConfiguredPrice("FixedPrice", 0, 1, 0, web3.utils.toWei("0.5", "ether"));
// configureExponentialDecreaseStrategy("EarlyBirdSale", 9900, web3.utils.toWei("0.05", "ether")); // 99% of current price, 0.05 ETH floor
```

## Related Documentation

- [Variable Price Library](../libraries/variable-price-lib.md)
- [MultiSale Interface](./imultisale.md) (often uses variable pricing)
- [OpenZeppelin SafeMath](https://docs.openzeppelin.com/contracts/4.x/api/utils#SafeMath)
- [Solidity ABI Encoding and Decoding](https://docs.soliditylang.org/en/latest/abi-spec.html)

## Standards Compliance

- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.
