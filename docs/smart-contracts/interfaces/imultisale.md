# IMultiSale Interface

The `IMultiSale` interface defines the standard for smart contracts that enable multi-token sales, supporting various token types (ERC20, ERC721, ERC1155) and flexible payment methods. This interface is crucial for implementing diverse sale mechanisms within the Gemforce ecosystem, such as presales, public sales, and Dutch auctions.

## Overview

`IMultiSale` provides:

- **Token Flexibility**: Supports ERC20, ERC721, and ERC1155 tokens.
- **Payment Versatility**: Configurable payment tokens and currencies.
- **Sale Lifecycle Management**: Functions for starting, pausing, and ending sales.
- **Participation Control**: Mechanisms for whitelisting and setting purchase limits.
- **Event Tracking**: Comprehensive event logging for sale activities.

## Key Features

### Sale Management
- **Start/Pause/End**: Control the lifecycle of sales
- **Configuration**: Set sale parameters like price, duration, limits
- **Fund Collection**: Manage primary and secondary payment collection

### Purchase Mechanics
- **Direct Purchase**: Standard buy function
- **Batch Purchase**: Purchase multiple items or tokens at once
- **Dynamic Pricing**: Support for variable pricing models (e.g., fixed, exponential, inverse logarithmic)
- **Refunds**: Mechanism for handling failed purchases or refunds

### Whitelisting and Access Control
- **Whitelist Management**: Add and remove addresses from whitelists
- **Purchase Limits**: Set maximum purchase quantities per address
- **Role-Based Access**: Define roles for managing sale configurations

## Interface Definition

```solidity
interface IMultiSale {
    // Events
    event SaleStarted(
        bytes32 indexed saleId,
        address indexed seller,
        uint256 startTime,
        uint256 endTime
    );
    event SalePaused(bytes32 indexed saleId);
    event SaleEnded(bytes32 indexed saleId);
    event Purchase(
        bytes32 indexed saleId,
        address indexed buyer,
        address indexed token,
        uint256 indexed tokenId,
        uint256 amount,
        uint256 pricePaid,
        address paymentToken
    );
    event PaymentCollected(
        bytes32 indexed saleId,
        address indexed payee,
        address paymentToken,
        uint256 amount
    );
    event WhitelistUpdated(bytes32 indexed saleId, address indexed addr, bool whitelisted);
    event SaleConfigUpdated(bytes32 indexed saleId);

    // Enums
    enum SaleState {
        NOT_STARTED,
        ACTIVE,
        PAUSED,
        ENDED
    }

    enum TokenType {
        ERC20,
        ERC721,
        ERC1155
    }

    // Structs
    struct SaleConfig {
        address seller;
        address saleTokenAddress;
        TokenType saleTokenType;
        uint256 saleTokenId; // For ERC721/ERC1155, 0 for ERC20
        uint256 totalAmount; // For ERC20/ERC1155, total NFTs for ERC721
        uint256 price; // Base price or starting price
        address paymentTokenAddress;
        uint256 startTime;
        uint256 endTime;
        uint256 maxPurchasePerAddress;
        bool isWhitelistedSale;
        address initialReceiver; // Where initial payment goes, often a treasury
        uint256 initialPaymentAmount; // Amount to collect initially
        uint256 feePercentage; // Percentage fee for the platform
        address feeReceiver; // Address to receive fees
        string name;
        string description;
        bytes data; // Additional data for specific sale types (e.g., pricing curves)
    }

    struct SaleDetails {
        SaleConfig config;
        SaleState state;
        uint256 amountSold;
        uint256 totalRevenue;
        uint256 collectedPayment;
        address[] buyers;
        mapping(address => uint256) purchasedAmount;
        mapping(address => bool) whitelisted;
    }

    // Sale Management Functions
    function createSale(SaleConfig calldata config) external returns (bytes32 saleId);
    function startSale(bytes32 saleId) external;
    function pauseSale(bytes32 saleId) external;
    function endSale(bytes32 saleId) external;
    function updateSaleConfig(bytes32 saleId, SaleConfig calldata newConfig) external;

    // Purchase Functions
    function buy(bytes32 saleId, uint256 amount) external payable;
    function buyBatch(bytes32 saleId, uint256[] calldata amounts) external payable;
    function refund(bytes32 saleId, uint256 amount) external;

    // Whitelist Functions
    function addToWhitelist(bytes32 saleId, address[] calldata addrs) external;
    function removeFromWhitelist(bytes32 saleId, address[] calldata addrs) external;
    function isWhitelisted(bytes32 saleId, address addr) external view returns (bool);

    // Query Functions
    function getSaleConfig(bytes32 saleId) external view returns (SaleConfig memory);
    function getSaleState(bytes32 saleId) external view returns (SaleState);
    function getAmountSold(bytes32 saleId) external view returns (uint256);
    function getTotalRevenue(bytes32 saleId) external view returns (uint256);
    function getPurchasedAmount(bytes32 saleId, address buyer) external view returns (uint256);
    function getAllActiveSales() external view returns (bytes32[] memory);
    function getSalesBySeller(address seller) external view returns (bytes32[] memory);
    function getSalesByPaymentToken(address paymentToken) external view returns (bytes32[] memory);
    function getCurrentPrice(bytes32 saleId) external view returns (uint256);

    // Payment Collection Functions
    function sweepFunds(bytes32 saleId, address receiver) external;
    function collectFees(bytes32 saleId, address feeReceiver) external;
}
```

## Core Functions

### createSale()
Initializes and creates a new multi-token sale with the specified configuration.

**Parameters:**
- `config`: A `SaleConfig` struct containing all parameters for the sale.

**Returns:**
- `bytes32`: A unique identifier for the created sale (`saleId`).

**Usage:**
```solidity
IMultiSale.SaleConfig memory saleConfig = IMultiSale.SaleConfig({
    seller: msg.sender,
    saleTokenAddress: address(nftContract),
    saleTokenType: IMultiSale.TokenType.ERC721,
    saleTokenId: 0, // Not applicable for ERC721 general sale
    totalAmount: 100,
    price: 1 ether, // Price per NFT
    paymentTokenAddress: address(0), // ETH
    startTime: block.timestamp + 1 hours,
    endTime: block.timestamp + 2 hours,
    maxPurchasePerAddress: 5,
    isWhitelistedSale: false,
    initialReceiver: treasury,
    initialPaymentAmount: 0,
    feePercentage: 500, // 5%
    feeReceiver: platformFeeRecipient,
    name: "My NFT Sale",
    description: "A limited edition NFT collection.",
    data: ""
});

bytes32 mySaleId = multiSaleContract.createSale(saleConfig);
```

### buy()
Allows a buyer to purchase `amount` of tokens/NFTs from an active sale.

**Parameters:**
- `saleId`: The ID of the sale.
- `amount`: The quantity of tokens/NFTs to purchase.

**Behavior:**
- Requires `msg.value` to match the calculated total price if payable ETH is used.
- Transfers the sale tokens to the buyer and collects payment.
- Updates `amountSold` and `totalRevenue`.

### addToWhitelist()
Adds multiple addresses to the whitelist for a specific sale. Only applicable for whitelisted sales.

**Parameters:**
- `saleId`: The ID of the sale.
- `addrs`: An array of addresses to add to the whitelist.

## Implementation Example

### Basic MultiSale Contract

```solidity
contract MultiSale is IMultiSale {
    // Storage
    mapping(bytes32 => SaleDetails) public sales;
    mapping(address => bytes32[]) public salesBySeller;
    mapping(bytes32 => address => uint256) private _purchasedAmount; // Keep track of amounts
    bytes32[] private _allSales;
    
    // Dependencies (example simplified, in real scenario these would be injected)
    IERC20 private _erc20;
    IERC721 private _erc721;
    IERC1155 private _erc1155;

    constructor(address erc20Addr, address erc721Addr, address erc1155Addr) {
        _erc20 = IERC20(erc20Addr);
        _erc721 = IERC721(erc721Addr);
        _erc1155 = IERC1155(erc1155Addr);
    }

    function createSale(SaleConfig calldata config) external returns (bytes32 saleId) {
        saleId = keccak256(abi.encodePacked(block.timestamp, config.seller));
        require(sales[saleId].config.seller == address(0), "Sale ID collision");

        // Basic validation
        require(config.totalAmount > 0, "Total amount must be greater than 0");
        require(config.price > 0, "Price must be greater than 0");
        require(config.endTime > config.startTime, "End time must be after start time");
        require(config.initialReceiver != address(0), "Initial receiver cannot be zero address");

        // Initialize SaleDetails
        sales[saleId].config = config;
        sales[saleId].state = SaleState.NOT_STARTED;
        sales[saleId].amountSold = 0;
        sales[saleId].totalRevenue = 0;
        sales[saleId].collectedPayment = 0;
        sales[saleId].config.seller = msg.sender; // Ensure seller is creation caller

        salesBySeller[msg.sender].push(saleId);
        _allSales.push(saleId);
        emit SaleConfigUpdated(saleId);
    }

    function startSale(bytes32 saleId) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.state == SaleState.NOT_STARTED || sale.state == SaleState.PAUSED, "Sale not in votable state");
        require(block.timestamp >= sale.config.startTime, "Sale has not started yet");
        require(block.timestamp <= sale.config.endTime, "Sale has already ended");

        sale.state = SaleState.ACTIVE;
        emit SaleStarted(saleId, msg.sender, sale.config.startTime, sale.config.endTime);
    }

    function pauseSale(bytes32 saleId) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.state == SaleState.ACTIVE, "Sale not active");

        sale.state = SaleState.PAUSED;
        emit SalePaused(saleId);
    }

    function endSale(bytes32 saleId) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender || block.timestamp > sale.config.endTime, "Caller not seller or sale not ended");
        require(sale.state == SaleState.ACTIVE || sale.state == SaleState.PAUSED, "Sale already ended or not started");

        sale.state = SaleState.ENDED;
        emit SaleEnded(saleId);
    }

    function buy(bytes32 saleId, uint256 amount) external payable {
        SaleDetails storage sale = sales[saleId];
        require(sale.state == SaleState.ACTIVE, "Sale not active");
        require(sale.amountSold + amount <= sale.config.totalAmount, "Exceeds total available");
        require(amount > 0, "Amount must be greater than 0");

        if (sale.config.isWhitelistedSale) {
            require(sale.whitelisted[msg.sender], "Not whitelisted");
        }
        require(_purchasedAmount[saleId][msg.sender] + amount <= sale.config.maxPurchasePerAddress, "Exceeds max purchase per address");

        uint256 currentPrice = getCurrentPrice(saleId); // Supports dynamic pricing
        uint256 totalPrice = currentPrice * amount;

        if (sale.config.paymentTokenAddress == address(0)) { // ETH payment
            require(msg.value == totalPrice, "Incorrect ETH amount sent");
        } else { // ERC20 payment
            require(msg.value == 0, "Do not send ETH for ERC20 payment");
            IERC20(sale.config.paymentTokenAddress).transferFrom(msg.sender, address(this), totalPrice);
        }

        // Transfer sale tokens
        _transferSaleTokens(saleId, msg.sender, amount);

        sale.amountSold += amount;
        sale.totalRevenue += totalPrice;
        _purchasedAmount[saleId][msg.sender] += amount;

        bool buyerFound = false;
        for (uint i = 0; i < sale.buyers.length; i++) {
        	if (sale.buyers[i] == msg.sender) {
        		buyerFound = true;
        		break;
        	}
        }
        if (!buyerFound) {
        	sale.buyers.push(msg.sender);
        }

        emit Purchase(saleId, msg.sender, sale.config.saleTokenAddress, sale.config.saleTokenId, amount, totalPrice, sale.config.paymentTokenAddress);
    }

    function _transferSaleTokens(bytes32 saleId, address buyer, uint256 amount) internal {
        SaleConfig storage config = sales[saleId].config;
        if (config.saleTokenType == TokenType.ERC20) {
            _erc20.transfer(buyer, amount); // from MultiSale contract's balance
        } else if (config.saleTokenType == TokenType.ERC721) {
            for (uint256 i = 0; i < amount; i++) {
                // Assuming saleTokenId is the starting ID or a way to select NFTs
                _erc721.transferFrom(address(this), buyer, config.saleTokenId + sales[saleId].amountSold + i);
            }
        } else if (config.saleTokenType == TokenType.ERC1155) {
            _erc1155.safeTransferFrom(address(this), buyer, config.saleTokenId, amount, "");
        }
    }

    function addToWhitelist(bytes32 saleId, address[] calldata addrs) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.config.isWhitelistedSale, "Not a whitelisted sale");

        for (uint256 i = 0; i < addrs.length; i++) {
            require(addrs[i] != address(0), "Cannot whitelist zero address");
            sale.whitelisted[addrs[i]] = true;
            emit WhitelistUpdated(saleId, addrs[i], true);
        }
    }

    // Example of a simple getCurrentPrice, could be replaced by an external VariablePriceLib
    function getCurrentPrice(bytes32 saleId) public view returns (uint256) {
        return sales[saleId].config.price;
    }

    function getSaleConfig(bytes32 saleId) external view returns (SaleConfig memory) {
        return sales[saleId].config;
    }

    function getSaleState(bytes32 saleId) external view returns (SaleState) {
        return sales[saleId].state;
    }

    function getAmountSold(bytes32 saleId) external view returns (uint256) {
        return sales[saleId].amountSold;
    }
    
    function getTotalRevenue(bytes32 saleId) external view returns (uint256) {
        return sales[saleId].totalRevenue;
    }
    
    function getPurchasedAmount(bytes32 saleId, address buyer) external view returns (uint256) {
        return _purchasedAmount[saleId][buyer];
    }
    
    function getAllActiveSales() external view returns (bytes32[] memory) {
        bytes32[] memory activeSales;
        uint256 count = 0;
        for (uint i = 0; i < _allSales.length; i++) {
            if (sales[_allSales[i]].state == SaleState.ACTIVE) {
                count++;
            }
        }
        activeSales = new bytes32[](count);
        uint256 j = 0;
        for (uint i = 0; i < _allSales.length; i++) {
            if (sales[_allSales[i]].state == SaleState.ACTIVE) {
                activeSales[j] = _allSales[i];
                j++;
            }
        }
        return activeSales;
    }
    
    function getSalesBySeller(address seller) external view returns (bytes32[] memory) {
        return salesBySeller[seller];
    }
    
    function getSalesByPaymentToken(address paymentToken) external view returns (bytes32[] memory) {
        bytes32[] memory salesList;
        uint256 count = 0;
        for (uint i = 0; i < _allSales.length; i++) {
            if (sales[_allSales[i]].config.paymentTokenAddress == paymentToken) {
                count++;
            }
        }
        salesList = new bytes32[](count);
        uint256 j = 0;
        for (uint i = 0; i < _allSales.length; i++) {
            if (sales[_allSales[i]].config.paymentTokenAddress == paymentToken) {
                salesList[j] = _allSales[i];
                j++;
            }
        }
        return salesList;
    }

    function refund(bytes32 saleId, uint256 amount) external {
        // Implement refund logic: check if refund is allowed, transfer funds back
        // This is a simplified example. Real-world would involve more checks.
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Only seller can refund");
        require(sale.state == SaleState.ENDED, "Sale must be ended to refund");
        // Further logic to identify who to refund and how much
        // For demonstration, let's assume it's a direct refund mechanism.
        // Requires a way to track individual buyer payments which isn't fully detailed in this basic interface.
        // E.g., if a purchase failed midway or for return policies.
        if (sale.config.paymentTokenAddress == address(0)) {
            payable(msg.sender).transfer(amount);
        } else {
            IERC20(sale.config.paymentTokenAddress).transfer(msg.sender, amount);
        }
    }

    function sweepFunds(bytes32 saleId, address receiver) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.state == SaleState.ENDED, "Sale must be ended");

        uint256 balance = (sale.config.paymentTokenAddress == address(0)) ? address(this).balance : IERC20(sale.config.paymentTokenAddress).balanceOf(address(this));
        require(balance > 0, "No funds to sweep");

        uint256 amountToSweep = balance - (balance * sale.config.feePercentage / 10000); // Exclude fees
        if (sale.config.paymentTokenAddress == address(0)) {
            payable(receiver).transfer(amountToSweep);
        } else {
            IERC20(sale.config.paymentTokenAddress).transfer(receiver, amountToSweep);
        }
        sale.collectedPayment += amountToSweep;
        emit PaymentCollected(saleId, receiver, sale.config.paymentTokenAddress, amountToSweep);
    }
    
    function collectFees(bytes32 saleId, address feeReceiver) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.state == SaleState.ENDED, "Sale must be ended to collect fees");

        uint256 totalCollected = (sale.config.paymentTokenAddress == address(0)) ? address(this).balance : IERC20(sale.config.paymentTokenAddress).balanceOf(address(this));
        uint256 feeAmount = totalCollected * sale.config.feePercentage / 10000;

        if (feeAmount > 0) {
            if (sale.config.paymentTokenAddress == address(0)) {
                payable(feeReceiver).transfer(feeAmount);
            } else {
                IERC20(sale.config.paymentTokenAddress).transfer(feeReceiver, feeAmount);
            }
            emit PaymentCollected(saleId, feeReceiver, sale.config.paymentTokenAddress, feeAmount);
        }
    }

    function updateSaleConfig(bytes32 saleId, SaleConfig calldata newConfig) external {
        SaleDetails storage sale = sales[saleId];
        require(sale.config.seller == msg.sender, "Caller not seller");
        require(sale.state == SaleState.NOT_STARTED || sale.state == SaleState.PAUSED, "Can only update config when not active");

        // Implement logic to update only allowed fields to prevent abuse
        sale.config = newConfig; // For simplicity, replacing whole config. In real scenario, selective updates.
        emit SaleConfigUpdated(saleId);
    }
}
```

## Security Considerations

### Access Control
- **Seller Privileges**: Ensure only the designated seller can manage sale lifecycle.
- **Whitelist Management**: Restrict whitelist updates to authorized roles.
- **Fund Control**: Implement secure mechanisms for fund collection and fee distribution.

### Fund Safety
- **Reentrancy Protection**: Use reentrancy guards on all critical transfer functions.
- **Input Validation**: Strict validation of `amount`, `price`, and `addresses`.
- **Token Handling**: Proper handling of ERC20, ERC721, and ERC1155 transfers.

### Sale Integrity
- **State Checks**: Verify sale state (`ACTIVE`, `PAUSED`, `ENDED`) before transactions.
- **Price Consistency**: Ensure pricing logic is robust and tamper-proof.
- **Overflow/Underflow**: Protect against integer overflows/underflows in calculations.

## Best Practices

### Sale Configuration
1. **Clear Parameters**: Define unambiguous sale parameters.
2. **Flexible Pricing**: Support various pricing models (fixed, dynamic).
3. **Refund Policy**: Implement clear and transparent refund mechanisms.

### Token Management
1. **Approve Before Transfer**: For ERC20, ensure the MultiSale contract has sufficient allowance.
2. **Safe ERC721/ERC1155 Transfers**: Use `safeTransferFrom` to prevent token loss.
3. **Ownership Transfer**: Ensure sale tokens are transferred to the contract prior to sale.

### Monitoring and Events
1. **Comprehensive Events**: Emit events for all significant actions (purchase, start, end, whitelist).
2. **Off-chain Indexing**: Use events for off-chain indexing and analytics.

## Integration Examples

### Frontend Integration (TypeScript)

```typescript
import { ethers, Contract } from 'ethers';
import MultiSaleABI from './MultiSale.json';

interface SaleConfig {
    seller: string;
    saleTokenAddress: string;
    saleTokenType: number; // Enum: 0=ERC20, 1=ERC721, 2=ERC1155
    saleTokenId: number;
    totalAmount: number;
    price: number;
    paymentTokenAddress: string;
    startTime: number;
    endTime: number;
    maxPurchasePerAddress: number;
    isWhitelistedSale: boolean;
    initialReceiver: string;
    initialPaymentAmount: number;
    feePercentage: number;
    feeReceiver: string;
    name: string;
    description: string;
    data: string;
}

class MultiSaleService {
    private contract: Contract;

    constructor(contractAddress: string, signer: ethers.Signer) {
        this.contract = new Contract(contractAddress, MultiSaleABI, signer);
    }

    async createNewSale(config: SaleConfig): Promise<string> {
        const tx = await this.contract.createSale(config);
        const receipt = await tx.wait();
        const event = receipt.events?.find((e: any) => e.event === 'SaleConfigUpdated');
        return event?.args?.saleId;
    }

    async startSale(saleId: string): Promise<void> {
        const tx = await this.contract.startSale(saleId);
        await tx.wait();
    }

    async buyTokens(saleId: string, amount: number, value: ethers.BigNumberish = 0): Promise<void> {
        const tx = await this.contract.buy(saleId, amount, { value });
        await tx.wait();
    }

    async getSaleDetails(saleId: string): Promise<any> {
        return await this.contract.getSaleConfig(saleId);
    }

    async checkWhitelist(saleId: string, address: string): Promise<boolean> {
        return await this.contract.isWhitelisted(saleId, address);
    }
}
```

### Backend Integration (Node.js with web3.js)

```javascript
const Web3 = require('web3');
const MultiSaleABI = require('./MultiSale.json').abi;

class MultiSaleBackend {
    constructor(providerUrl, multiSaleContractAddress) {
        this.web3 = new Web3(providerUrl);
        this.multiSaleContract = new this.web3.eth.Contract(MultiSaleABI, multiSaleContractAddress);
    }

    async addAddressesToWhitelist(privateKey, saleId, addresses) {
        const account = this.web3.eth.accounts.privateKeyToAccount(privateKey);
        this.web3.eth.accounts.wallet.add(account);

        const gasEstimate = await this.multiSaleContract.methods.addToWhitelist(saleId, addresses).estimateGas({ from: account.address });
        const tx = await this.multiSaleContract.methods.addToWhitelist(saleId, addresses).send({ from: account.address, gas: gasEstimate });
        console.log(`Whitelist updated for sale ${saleId}: ${tx.transactionHash}`);
        return tx;
    }

    async getActiveSales() {
        return await this.multiSaleContract.methods.getAllActiveSales().call();
    }

    async handlePurchaseWebhook(saleId, buyerAddress, amount, pricePaid, paymentToken) {
        console.log(`Processing purchase for sale: ${saleId}`);
        console.log(`Buyer: ${buyerAddress}, Amount: ${amount}, Price: ${pricePaid}, Payment Token: ${paymentToken}`);
        // Here you would integrate with your backend systems, e.g.,
        // update internal databases, trigger notifications, etc.
    }
}
```

## Related Documentation

- [Variable Price Library](../libraries/variable-price-lib.md)
- [ERC20 Standard](https://eips.ethereum.org/EIPS/eip-20)
- [ERC721 Standard](https://eips.ethereum.org/EIPS/eip-721)
- [ERC1155 Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [Fee Distributor Library](../libraries/fee-distributor-lib.md)
- [Deployment Guide](../../developer-guides/development-environment-setup.md)

## Standards Compliance

- **ERC20**: Tokens used for payment or sale.
- **ERC721**: NFTs used for sale.
- **ERC1155**: Multi-tokens used for sale.
- **EIP-165**: Interface detection standard (implicitly used by calling `IERC*` functions).