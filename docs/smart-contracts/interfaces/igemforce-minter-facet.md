# IGemforceMinterFacet Interface

The `IGemforceMinterFacet` interface defines the standard functions for a minter facet within a Gemforce Diamond contract, specifically designed for minting and managing Gemforce-specific NFTs. This facet integrates with the core tokenization and attribute management systems of the Gemforce platform.

## Overview

`IGemforceMinterFacet` provides:

- **NFT Minting**: Functions to create new NFTs with specified metadata and attributes.
- **Supply Management**: Control over total supply, minted supply, and unique identifiers.
- **Attribute Integration**: Seamless integration with the platform's attribute system.
- **Royalty/Fee Handling**: Mechanisms to define and collect royalties or minting fees.
- **Event Tracking**: Comprehensive event logging for all minting operations.

## Key Features

### Minting Operations
- **Single Mint**: Mint one NFT at a time.
- **Batch Mint**: Mint multiple NFTs in a single transaction for efficiency.
- **Custom Metadata**: Attach custom metadata and attributes during minting.
- **Pre-minting Hooks**: Support for pre-minting validation or logic.

### Supply and Identifier Management
- **Token ID Generation**: Strategies for unique token ID generation.
- **Supply Tracking**: Keep track of total minted NFTs.
- **Provenance**: Record minter and minting timestamp.

### Royalty and Fee Mechanisms
- **Configurable Royalties**: Define royalty percentages and recipients.
- **Minting Fees**: Charge fees for minting operations.
- **Fee Distribution**: Integration with fee distribution mechanisms.

## Interface Definition

```solidity
interface IGemforceMinterFacet {
    // Events
    event GemforceMinted(
        address indexed minter,
        uint256 indexed tokenId,
        uint256 indexed quantity,
        address indexed receiver,
        bytes metadataURI,
        bytes attributes
    );
    event GemforceBatchMinted(
        address indexed minter,
        uint256[] tokenIds,
        uint256[] quantities,
        address indexed receiver,
        bytes metadataURI,
        bytes attributes
    );
    event MintFeeSet(address indexed token, uint256 amount);
    event RoyaltyInfoSet(address indexed receiver, uint96 feeNumerator);
    event BaseURIUpdated(string newBaseURI);

    // Structs
    struct MintConfig {
        uint256 initialSupply;
        uint256 pricePerUnit;
        address paymentToken; // Address of ERC20 token for payment, or address(0) for native currency
        uint256 maxSupply;
        bool isPausable;
        bool isBridgeable;
        string name;
        string symbol;
        string baseURI;
        string contractURI;
        bytes extraData; // For custom metadata or specific minting logic
    }

    struct MintBatchParams {
        uint256[] quantities;
        address receiver;
        bytes attributesData;
        bytes metadataURI;
    }

    // Core Minting Functions
    function mint(
        address receiver,
        uint256 quantity,
        bytes attributesData,
        bytes metadataURI
    ) external payable returns (uint256 newTokenId);

    function mintBatch(MintBatchParams[] calldata mintParams) external payable;

    // Configuration Functions
    function setMintFee(address paymentToken, uint256 amount) external;
    function setRoyaltyInfo(address receiver, uint96 feeNumerator) external;
    function setBaseURI(string memory newBaseURI) external;
    function setContractURI(string memory newContractURI) external;
    function setMaxSupply(uint256 newMaxSupply) external;

    // View Functions
    function getMintFee(address paymentToken) external view returns (uint256);
    function getRoyaltyInfo(uint256 tokenId, uint256 salePrice) external view returns (address receiver, uint256 royaltyAmount);
    function getBaseURI() external view returns (string memory);
    function getContractURI() external view returns (string memory);
    function getMaxSupply() external view returns (uint256);
    function getTotalMintedSupply() external view returns (uint256);
    function isPaused() external view returns (bool);
}
```

## Core Functions

### mint()
Mints a single or a specified quantity of new Gemforce NFTs to a designated receiver.

**Parameters:**
- `receiver`: The address to receive the minted NFTs.
- `quantity`: The number of NFTs to mint.
- `attributesData`: Bytes array containing encoded attribute data for the NFTs.
- `metadataURI`: URI pointing to off-chain metadata (e.g., JSON file).

**Behavior:**
- Handles payment of minting fees, if configured.
- Increments the total minted supply.

**Usage:**
```solidity
// Mint 1 NFT to a specific address with attributes and metadata URI
gemforceMinter.mint(
    recipientAddress,
    1,
    abi.encodePacked("color", "red", "size", "L"),
    "ipfs://example.com/metadata/1.json"
);

// Mint 5 NFTs (if applicable)
gemforceMinter.mint(
    recipientAddress,
    5,
    abi.encodePacked("series", "A"),
    "ipfs://example.com/metadata/series_a.json"
);
```

### mintBatch()
Mints multiple batches of NFTs in a single transaction. Each element in `mintParams` represents a separate minting operation with its own quantity, receiver, attributes, and metadata.

**Parameters:**
- `mintParams`: An array of `MintBatchParams` structs, each detailing a batch minting request.

**Usage:**
```solidity
IGemforceMinterFacet.MintBatchParams[] memory batches = new IGemforceMinterFacet.MintBatchParams[](2);
batches[0] = IGemforceMinterFacet.MintBatchParams({
    quantities: 10,
    receiver: user1,
    attributesData: abi.encodePacked("tier", "gold"),
    metadataURI: "ipfs://gold.json"
});
batches[1] = IGemforceMinterFacet.MintBatchParams({
    quantities: 5,
    receiver: user2,
    attributesData: abi.encodePacked("tier", "silver"),
    metadataURI: "ipfs://silver.json"
});

gemforceMinter.mintBatch(batches);
```

### setMintFee()
Sets the minting fee for a specific payment token.

**Parameters:**
- `paymentToken`: The address of the ERC20 token to be used for fee payment, or `address(0)` for native currency (ETH).
- `amount`: The amount of the specified `paymentToken` required per mint.

### setRoyaltyInfo()
Sets the royalty information for the NFTs minted by this facet.

**Parameters:**
- `receiver`: The address that will receive the royalties.
- `feeNumerator`: A fraction (numerator) used to calculate the royalty amount from the sale price, compatible with EIP-2981.

## Implementation Example

```solidity
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Enumerable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Metadata.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Context.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

// Assuming LibDiamond and AttributeLib are available through the Diamond
interface IDiamond {
    function owner() external view returns (address);
    function addFunction(address functionToCall, bytes4 _selector) external;
    function removeFunction(bytes4 _selector) external;
}

library AttributeLib {
    function setAttributes(uint256 tokenId, bytes calldata data) internal;
    // other attribute-related functions
}

contract GemforceMinterFacet is IGemforceMinterFacet, ERC2981, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdCounter;

    string private _baseURI;
    string private _contractURI;
    uint256 private _maxSupply;
    uint256 private _totalMintedSupply;
    
    mapping(address => uint256) private _mintFees; // paymentToken => feeAmount
    address private _royaltyReceiver;
    uint96 private _royaltyFeeNumerator;

    // ERC165 support
    bytes4 private constant _INTERFACE_ID_ERC721 = 0x80ac58cd;
    bytes4 private constant _INTERFACE_ID_ERC721ENUMERABLE = 0x780e9d63;
    bytes4 private constant _INTERFACE_ID_ERC721METADATA = 0x5b5e139f;

    constructor(uint256 maxSupply_, string memory baseURI_, string memory contractURI_) {
        _maxSupply = maxSupply_;
        _baseURI = baseURI_;
        _contractURI = contractURI_;
        // _setOwner(msg.sender); // In a diamond, owner is typically set by OwnershipFacet
    }

    // This needs to be hooked up to the ERC721 token's _mint function (e.g. from ERC721Facet)
    // For demonstration, we simulate the mint
    function _safeMint(address to, uint256 tokenId) internal {
        require(_totalMintedSupply < _maxSupply, "Supply limit reached");
        // In a real diamond, this would call into an ERC721Facet or similar
        // For now, we increment the counter and minted supply
        _tokenIdCounter.increment();
        _totalMintedSupply++;
        // emit Transfer(address(0), to, tokenId); // Simulate ERC721 Transfer event
    }

    function mint(
        address receiver,
        uint256 quantity,
        bytes attributesData,
        bytes metadataURI
    ) external payable override returns (uint256 newTokenId) {
        require(receiver != address(0), "Receiver cannot be zero address");
        require(quantity > 0, "Quantity must be greater than 0");
        require(_totalMintedSupply + quantity <= _maxSupply, "Exceeds max supply");

        // Handle mint fees
        uint256 requiredFee = _mintFees[address(0)] * quantity; // For native currency
        if (requiredFee > 0) {
            require(msg.value >= requiredFee, "Insufficient ETH for mint fee");
            // Transfer excess ETH back
            if (msg.value > requiredFee) {
                payable(msg.sender).transfer(msg.value - requiredFee);
            }
            // Transfer fee to owner/fee collector (assuming owner is fee collector)
            payable(owner()).transfer(requiredFee);
        }

        // For ERC20 fee, it would be an ERC20.transferFrom call before minting

        newTokenId = _tokenIdCounter.current();
        for (uint256 i = 0; i < quantity; i++) {
            _safeMint(receiver, newTokenId + i);
            AttributeLib.setAttributes(newTokenId + i, attributesData); // Integrate with attribute system
            // Associated metadataURI could be stored or linked
        }

        emit GemforceMinted(msg.sender, newTokenId, quantity, receiver, metadataURI, attributesData);
    }

    function mintBatch(MintBatchParams[] calldata mintParams) external payable override {
        uint256 totalQuantity = 0;
        uint256 totalEthFee = 0;

        for (uint256 i = 0; i < mintParams.length; i++) {
            require(mintParams[i].quantities > 0, "Quantity must be greater than 0");
            require(mintParams[i].receiver != address(0), "Receiver cannot be zero address");
            totalQuantity += mintParams[i].quantities;
            totalEthFee += _mintFees[address(0)] * mintParams[i].quantities;
        }

        require(_totalMintedSupply + totalQuantity <= _maxSupply, "Exceeds max supply for batch mint");

        if (totalEthFee > 0) {
            require(msg.value >= totalEthFee, "Insufficient ETH for batch mint fee");
            // Transfer excess ETH back if any
            if (msg.value > totalEthFee) {
                payable(msg.sender).transfer(msg.value - totalEthFee);
            }
            payable(owner()).transfer(totalEthFee); // Transfer total fee
        }

        for (uint256 i = 0; i < mintParams.length; i++) {
            uint256 currentTokenId = _tokenIdCounter.current();
            for (uint256 j = 0; j < mintParams[i].quantities; j++) {
                _safeMint(mintParams[i].receiver, currentTokenId + j);
                AttributeLib.setAttributes(currentTokenId + j, mintParams[i].attributesData);
            }
            emit GemforceBatchMinted(
                msg.sender,
                _getTokenIdsFromRange(currentTokenId, mintParams[i].quantities),
                new uint256[](0), // quantities for each token, if different
                mintParams[i].receiver,
                mintParams[i].metadataURI,
                mintParams[i].attributesData
            );
        }
    }

    function _getTokenIdsFromRange(uint256 startTokenId, uint256 count) internal pure returns (uint256[] memory) {
        uint256[] memory tokenIds = new uint256[](count);
        for (uint256 i = 0; i < count; i++) {
            tokenIds[i] = startTokenId + i;
        }
        return tokenIds;
    }

    function setMintFee(address paymentToken, uint256 amount) external override onlyOwner {
        _mintFees[paymentToken] = amount;
        emit MintFeeSet(paymentToken, amount);
    }

    function getMintFee(address paymentToken) external view override returns (uint256) {
        return _mintFees[paymentToken];
    }

    function setRoyaltyInfo(address receiver, uint96 feeNumerator) external override onlyOwner {
        _royaltyReceiver = receiver;
        _royaltyFeeNumerator = feeNumerator;
        emit RoyaltyInfoSet(receiver, feeNumerator);
        _setDefaultRoyalty(receiver, feeNumerator); // Sets default for ERC2981
    }

    // Override the ERC2981 royaltyInfo to always use the contract-wide settings
    function royaltyInfo(uint256 _tokenId, uint256 _salePrice)
        public
        view
        override
        returns (address receiver, uint256 royaltyAmount)
    {
        return (_royaltyReceiver, _salePrice * _royaltyFeeNumerator / ERC2981_DENOMINATOR);
    }

    function getBaseURI() external view override returns (string memory) {
        return _baseURI;
    }

    function setBaseURI(string memory newBaseURI) external override onlyOwner {
        _baseURI = newBaseURI;
        emit BaseURIUpdated(newBaseURI);
    }

    function getContractURI() external view override returns (string memory) {
        return _contractURI;
    }

    function setContractURI(string memory newContractURI) external override onlyOwner {
        _contractURI = newContractURI;
    }

    function getMaxSupply() external view override returns (uint256) {
        return _maxSupply;
    }

    function setMaxSupply(uint256 newMaxSupply) external override onlyOwner {
        require(newMaxSupply >= _totalMintedSupply, "New max supply cannot be less than minted supply");
        _maxSupply = newMaxSupply;
    }

    function getTotalMintedSupply() external view override returns (uint256) {
        return _totalMintedSupply;
    }

    function isPaused() external view override returns (bool) {
        // This would typically involve a Pausable contract or similar logic
        return false;
    }

    // ERC165 support
    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC2981, Context) returns (bool) {
        return 
            interfaceId == type(IGemforceMinterFacet).interfaceId ||
            interfaceId == _INTERFACE_ID_ERC721 || // Placeholder, assuming underlying ERC721
            interfaceId == _INTERFACE_ID_ERC721METADATA ||
            interfaceId == _INTERFACE_ID_ERC721ENUMERABLE ||
            super.supportsInterface(interfaceId);
    }
}
```

## Security Considerations

### Access Control
- **Owner Privileges**: Crucial functions like `setMintFee`, `setRoyaltyInfo`, `setBaseURI`, and `setMaxSupply` must be restricted to the contract owner or an authorized administrator.
- **Diamond Integration**: Ensure that the facet's access control aligns with the overall Diamond contract's ownership and access management.

### Supply Management
- **Max Supply Enforcement**: Rigorous checks to prevent minting beyond the `maxSupply`.
- **Token ID Uniqueness**: Ensure that minted token IDs are unique and avoid collisions.
- **Re-entrancy**: While low risk for simple minting, complex payment or external calls within mint operations should implement re-entrancy guards.

### Fee Handling
- **Eth/Token Distinction**: Correctly handle native currency (ETH) payments versus ERC20 token payments, ensuring proper `msg.value` checks and `transferFrom` calls.
- **Overpayment/Underpayment**: Implement mechanisms to refund overpayments or revert if underpaid.

## Best Practices

### Metadata and Attributes
1. **URI Standards**: Adhere to established standards (e.g., EIP-1577 for `ERC721Metadata` or `contractURI` for collection-level metadata) for off-chain metadata.
2. **On-chain Attributes**: Utilize a dedicated attribute system (like `AttributeLib`) for on-chain verifiable traits, ensuring data integrity.

### Efficiency
1. **Batch Operations**: Provide batch minting functionality (`mintBatch`) to reduce transaction costs for users minting multiple NFTs.
2. **Gas Optimization**: Optimize internal logic to minimize gas consumption, especially for frequently called functions.

### Modularity
1. **Facet Separation**: Ensure functions related to minting are encapsulated within this facet, adhering to the Diamond Standard's principle of modularity.
2. **Upgradeability**: Design the facet to be easily upgradeable or replaceable within the Diamond structure.

## Integration Examples

### Frontend Integration (TypeScript via Ethers.js)

```typescript
import { ethers, Contract } from 'ethers';
import MinterFacetABI from './GemforceMinterFacet.json'; // ABI for the IGemforceMinterFacet

const MINTER_FACET_ADDRESS = "0x..."; // Address of the deployed MinterFacet within the Diamond
const DIAMOND_ADDRESS = "0x..."; // Address of the Diamond contract itself

// Assuming provider from WalletConnect, Metamask, etc.
const getSigner = () => new ethers.providers.Web3Provider(window.ethereum).getSigner();
const getMinterFacet = () => new Contract(MINTER_FACET_ADDRESS, MinterFacetABI, getSigner());

interface MintParams {
    receiver: string;
    quantity: number;
    attributesData: string; // Hex string for bytes
    metadataURI: string;
    value?: ethers.BigNumberish; // For ETH payment
}

async function handleMintNFT(params: MintParams) {
    try {
        const minterFacet = getMinterFacet();
        const tx = await minterFacet.mint(
            params.receiver,
            params.quantity,
            ethers.utils.arrayify(params.attributesData), // Convert hex string to bytes array
            ethers.utils.toUtf8Bytes(params.metadataURI), // Convert string to bytes array
            { value: params.value }
        );
        await tx.wait();
        alert("NFT minted successfully!");
        console.log("Mint Transaction Hash:", tx.hash);
    } catch (error) {
        console.error("Error minting NFT:", error);
        alert("Failed to mint NFT. Check console for details.");
    }
}

async function handleSetMintFee(paymentTokenAddress: string, amount: ethers.BigNumberish) {
    try {
        const minterFacet = getMinterFacet();
        const tx = await minterFacet.setMintFee(paymentTokenAddress, amount);
        await tx.wait();
        alert("Mint fee updated.");
    } catch (error) {
        console.error("Error setting mint fee:", error);
        alert("Failed to set mint fee.");
    }
}
```

### Backend Integration (Node.js/Web3.js for off-chain minting trigger)

```javascript
const Web3 = require('web3');
const GemforceMinterFacetABI = require('./GemforceMinterFacet.json').abi;

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const minterFacetAddress = '0x...'; // Address of your deployed minter facet
const adminPrivateKey = 'YOUR_ADMIN_PRIVATE_KEY'; // Private key of the account authorized to mint

const minterFacetContract = new web3.eth.Contract(GemforceMinterFacetABI, minterFacetAddress);
const adminAccount = web3.eth.accounts.privateKeyToAccount(adminPrivateKey);
web3.eth.accounts.wallet.add(adminAccount);

async function triggerMint(receiverAddress, quantity, attributesDataHex, metadataURI) {
    try {
        console.log(`Triggering mint for ${quantity} NFTs to ${receiverAddress}...`);
        const tx = minterFacetContract.methods.mint(
            receiverAddress,
            quantity,
            web3.utils.hexToBytes(attributesDataHex),
            web3.utils.hexToBytes(web3.utils.utf8ToHex(metadataURI))
        );

        const gas = await tx.estimateGas({ from: adminAccount.address });
        const receipt = await tx.send({ from: adminAccount.address, gas: gas });

        console.log('Mint transaction sent:', receipt.transactionHash);
        return receipt;
    } catch (error) {
        console.error('Error triggering mint:', error);
        throw error;
    }
}

// Example usage
// triggerMint("0xUserWalletAddress", 1, "0xabcdef123456", "https://api.example.com/nft/1");
```

## Related Documentation

- [Diamond Standard Overview](../diamond.md)
- [Attribute Library](../libraries/attribute-lib.md)
- [Fee Distributor Library](../libraries/fee-distributor-lib.md)
- [EIP-2981 NFT Royalty Standard](https://eips.ethereum.org/EIPS/eip-2981)
- [ERC721 Standard](https://eips.ethereum.org/EIPS/eip-721)

## Standards Compliance

- **EIP-2535**: Designed as a facet for the Diamond Standard.
- **EIP-2981**: Implements the NFT Royalty Standard for royalty distribution.
- **ERC721**: Provides minting functionality for ERC721-compatible NFTs (assumes an underlying ERC721 facet for actual token management).
- **ERC165**: Supports interface detection.