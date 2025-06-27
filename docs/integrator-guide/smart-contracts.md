# Integrator's Guide: Smart Contracts

Integrating with Gemforce's smart contracts involves direct interaction with the blockchain. This guide will walk you through the essential tools, concepts, and best practices for interacting with Gemforce's Diamond-based smart contract architecture.

## Overview of Gemforce Smart Contracts

Gemforce utilizes the [Diamond Standard (EIP-2535)](../smart-contracts/diamond.md) for its core smart contract architecture. This modular approach offers several benefits:

-   **Modularity**: Functionalities are organized into smaller, self-contained `facets`.
-   **Upgradeability**: Facets can be added, replaced, or removed without redeploying the main Diamond contract, allowing for continuous upgrades and bug fixes.
-   **No 24KB Limit**: Bypasses the 24KB contract size limit, enabling extensive functionality.
-   **Shared Storage**: All facets share the same storage, ensuring consistent state across the system.

Key Gemforce smart contract components include:

-   **Diamond**: The main entry point, a proxy contract.
-   **Facets**: Contracts containing the business logic and functions.
-   **Libraries**: Reusable code used by facets.
-   **Interfaces**: Define the external functions of contracts.

## Essential Tools for Smart Contract Integration

To interact with Gemforce's smart contracts, you'll need:

1.  **Web3 Provider**: Connects your application to the blockchain (e.g., MetaMask, WalletConnect, Infura, Alchemy, your own node).
2.  **Web3 Library**: JavaScript libraries to interact with Ethereum (e.g., [Ethers.js](https://docs.ethers.org/) or [Web3.js](https://web3js.readthedocs.io/)). Ethers.js is generally preferred for its cleaner API and
    robust features.
3.  **Contract ABIs (Application Binary Interfaces)**: These JSON files define the functions and events of a smart contract, allowing your application to encode/decode function calls and event logs. You can obtain Gemforce contract ABIs from our deployed contracts or the `gem-base` project.

## Interacting with a Diamond Contract

A Diamond contract acts as a single address through which you interact with all connected facets. When you call a function on a Diamond, it delegates the call to the appropriate facet containing that function.

### Steps to Interact:

1.  **Get the Diamond Address**: This is the main address of the deployed Gemforce Diamond contract.
2.  **Obtain Facet ABIs**: While you interact with the Diamond's address, your Web3 library needs the ABI of the _facet_ that contains the function you want to call. This is because the ABI defines the function signatures.
3.  **Create a Contract Instance**: Instantiate your Web3 library's `Contract` object using the Diamond's address and the relevant Facet's ABI.
4.  **Call Functions / Listen for Events**: Use the contract instance to send transactions or query data.

### Example (Ethers.js with a Diamond Facet)

Suppose you want to interact with the `MarketplaceFacet` to list an NFT.

```typescript
import { ethers } from 'ethers';
// Import the ABI for MarketplaceFacet
import MarketplaceFacetABI from './MarketplaceFacet.json'; // Adjust path as needed

const GEMFORCE_DIAMOND_ADDRESS = "0x..."; // Replace with the actual deployed Diamond address
const YOUR_NFT_CONTRACT_ADDRESS = "0x..."; // Replace with your NFT contract address
const NFT_TOKEN_ID = 123;
const PRICE = ethers.utils.parseEther("0.5"); // 0.5 ETH
const PAYMENT_TOKEN_ADDRESS = "0x..."; // Address of WETH or other ERC20 accepted payment token, or address(0) for ETH

async function listNFTOnGemforceMarketplace() {
    try {
        // 1. Connect to a Web3 Provider and Signer (e.g., via MetaMask in a browser)
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        await provider.send("eth_requestAccounts", []); // Prompts user to connect wallet
        const signer = provider.getSigner();
        const userAddress = await signer.getAddress();
        console.log("Connected account:", userAddress);

        // 2. Create a contract instance using the Diamond Address and MarketplaceFacet ABI
        // Although you call methods on myMarketplace, the transaction goes through the Diamond.
        const myMarketplace = new ethers.Contract(
            GEMFORCE_DIAMOND_ADDRESS, // The Diamond's address
            MarketplaceFacetABI,      // The ABI of the Facet containing the function
            signer                    // The signer (user's wallet)
        );

        // 3. Approve the Marketplace to spend your NFT (if it's ERC721)
        // This assumes you have an ERC721 contract instance.
        const erc721Abi = ["function approve(address to, uint256 tokenId)"];
        const nftContract = new ethers.Contract(YOUR_NFT_CONTRACT_ADDRESS, erc721Abi, signer);
        
        console.log(`Approving MarketplaceFacet for NFT ID ${NFT_TOKEN_ID}...`);
        const approveTx = await nftContract.approve(GEMFORCE_DIAMOND_ADDRESS, NFT_TOKEN_ID);
        await approveTx.wait();
        console.log("Approval successful. Transaction hash:", approveTx.hash);

        // 4. Call the 'listItem' function from the MarketplaceFacet
        console.log("Listing NFT on marketplace...");
        const listItemTx = await myMarketplace.listItem(
            YOUR_NFT_CONTRACT_ADDRESS,
            NFT_TOKEN_ID,
            PRICE,
            PAYMENT_TOKEN_ADDRESS
        );
        const receipt = await listItemTx.wait();
        console.log("NFT listed successfully! Transaction hash:", receipt.hash);
        console.log("Events emitted:", receipt.events);

    } catch (error) {
        console.error("Error listing NFT:", error);
    }
}

// Call the function
// listNFTOnGemforceMarketplace();
```

## Reading Data from Smart Contracts (View/Pure Functions)

`view` and `pure` functions do not modify the blockchain state and can be called without sending a transaction (and thus without gas fees).

### Example (Ethers.js)

```typescript
import { ethers } from 'ethers';
import DiamondLoupeFacetABI from './DiamondLoupeFacet.json'; // ABI for DiamondLoupeFacet

const GEMFORCE_DIAMOND_ADDRESS = "0x..."; // Replace with the actual Diamond address
const PROVIDER_URL = "https://rpc.sepolia.org"; // Example RPC URL

async function getFacetAddresses() {
    try {
        const provider = new ethers.providers.JsonRpcProvider(PROVIDER_URL);

        // Create a contract instance with DiamondLoupeFacet ABI
        const diamondLoupe = new ethers.Contract(
            GEMFORCE_DIAMOND_ADDRESS,
            DiamondLoupeFacetABI,
            provider // No signer needed for view calls
        );

        // Call a view function to get all registered facet addresses
        const facetAddresses = await diamondLoupe.facetAddresses();
        console.log("All Facet Addresses:", facetAddresses);

        // You can then iterate through facetAddresses to get more details if needed
        for (const addr of facetAddresses) {
            const selectors = await diamondLoupe.facetFunctionSelectors(addr);
            console.log(`  Facet ${addr} has selectors:`, selectors);
        }

    } catch (error) {
        console.error("Error getting facet addresses:", error);
    }
}

// Call the function
// getFacetAddresses();
```

## Listening for Smart Contract Events

Smart contracts emit events to signal significant occurrences. Your application can listen for these events to react in real-time to on-chain changes without constantly polling.

### Example (Ethers.js)

```typescript
import { ethers } from 'ethers';
import MarketplaceFacetABI from './MarketplaceFacet.json'; // ABI for MarketplaceFacet

const GEMFORCE_DIAMOND_ADDRESS = "0x..."; // Replace with the actual deployed Diamond address
const PROVIDER_URL = "wss://base-sepolia.public.blastapi.io"; // Use a WebSocket provider for real-time events

async function listenForNFTListings() {
    try {
        const provider = new ethers.providers.WebSocketProvider(PROVIDER_URL);

        // Create a contract instance for listening to events
        const myMarketplace = new ethers.Contract(
            GEMFORCE_DIAMOND_ADDRESS,
            MarketplaceFacetABI,
            provider
        );

        console.log("Listening for 'ItemListed' events on the Marketplace...");

        // Listen for the ItemListed event
        // The event signature typically looks like: event ItemListed(address indexed nftContract, uint256 indexed tokenId, address indexed seller, uint256 price, address paymentToken);
        myMarketplace.on("ItemListed", (nftContract, tokenId, seller, price, paymentToken, event) => {
            console.log("\n--- New NFT Listed! ---");
            console.log("  NFT Contract:", nftContract);
            console.log("  Token ID:", tokenId.toString());
            console.log("  Seller:", seller);
            console.log("  Price:", ethers.utils.formatEther(price), "ETH"); // Assuming price in Wei
            // paymentToken might be address(0) for native currency
            console.log("  Payment Token:", paymentToken === ethers.constants.AddressZero ? "ETH" : paymentToken);
            console.log("  Transaction Hash:", event.transactionHash);
            console.log("-----------------------");
        });

    } catch (error) {
        console.error("Error setting up event listener:", error);
    }
}

// Call the function
// listenForNFTListings();
```

## Best Practices for Smart Contract Integration

-   **Gas Management**: Account for gas fees. Provide users with clear information about potential costs. For backend automation, ensure your calling account has sufficient funds.
-   **Error Handling**: Smart contract calls can revert. Implement robust `try-catch` blocks and parse revert messages for user feedback.
-   **Security**: Never expose private keys in client-side code. Use secure signing processes (e.g., MetaMask, DFNS). Follow secure coding guidelines for your dApp.
-   **Network Agnosticism**: Design your application to be configurable for different networks (e.g., Sepolia, Base Sepolia, Optimism Sepolia) by using network IDs, RPC URLs, and contract addresses.
-   **Diamond Specifics**: Always read the Diamond contract's specific documentation for details on its deployed facets and their expected behaviors.
-   **ABIs**: Ensure you are using the correct and up-to-date ABIs for the facets you intend to interact with.
-   **Optimistic UI/UX**: For state-changing transactions, provide immediate visual feedback to the user while waiting for transactions to be mined.

## Related Documentation

-   [Diamond Standard Overview](../smart-contracts/diamond.md)
-   [Facets documentation](../smart-contracts/facets/diamond-cut-facet.md)
-   [Interfaces documentation](../smart-contracts/interfaces/idiamond.md)
-   [Libraries documentation](../smart-contracts/libraries/lib-diamond.md)
-   [Authentication Guide](authentication.md)