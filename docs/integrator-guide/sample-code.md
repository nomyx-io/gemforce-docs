# Integrator's Guide: Sample Code

This section provides practical code samples and snippets to help you quickly get started with integrating your applications with the Gemforce platform. The examples cover common use cases and demonstrate how to interact with various Gemforce components, including smart contracts, cloud functions, and APIs.

## Overview of Sample Code

The samples are categorized by the Gemforce component they demonstrate:

-   **Smart Contract Interaction**: Examples for deploying, calling functions, and listening to events on Gemforce's Diamond contracts.
-   **Cloud Function & REST API Usage**: Demonstrations of user authentication, data management (CRUD), and invoking custom cloud logic.
-   **Advanced Integrations**: Code for features like DFNS transaction signing, webhook handling, and data analytics.

Each example is designed to be self-contained where possible, but you may need to refer to other sections of this Integrator's Guide for full context (e.g., [Authentication](authentication.md), [Smart Contracts](smart-contracts.md)).

## 1. Smart Contract Interaction Examples

### 1.1 Deploying a Diamond Contract (using Hardhat/Ethers.js)

This example shows a simplified Hardhat script to deploy a Gemforce-style Diamond contract with some initial facets.

```typescript
// scripts/deployDiamond.ts
import { ethers } from "hardhat";
import { FacetCutAction } from "../@gemforce-sdk/hardhat-diamond/types"; // Assuming SDK provides this enum

async function deployGemforceDiamond() {
    const [deployer] = await ethers.getSigners();
    console.log(`Deploying contracts with the account: ${deployer.address}`);

    // Deploy LibDiamond
    const LibDiamond = await ethers.getContractFactory("LibDiamond");
    const libDiamond = await LibDiamond.deploy();
    await libDiamond.deployed();
    console.log(`LibDiamond deployed to: ${libDiamond.address}`);

    // Deploy Diamond
    const Diamond = await ethers.getContractFactory("Diamond", {
        libraries: {
            LibDiamond: libDiamond.address,
        },
    });
    const diamond = await Diamond.deploy(deployer.address, true); // initial owner, whether to add owner as facet
    await diamond.deployed();
    console.log(`Gemforce Diamond deployed to: ${diamond.address}`);

    // Deploy Facets
    const FacetA = await ethers.getContractFactory("FacetA");
    const facetA = await FacetA.deploy();
    await facetA.deployed();
    console.log(`FacetA deployed to: ${facetA.address}`);

    const FacetB = await ethers.getContractFactory("FacetB");
    const facetB = await FacetB.deploy();
    await facetB.deployed();
    console.log(`FacetB deployed to: ${facetB.address}`);

    // Get DiamondCutFacet and DiamondLoupeFacet (assuming they are part of the Diamond initially or deployed separately)
    // For simplicity, let's assume Diamond.sol contains basic DiamondCutFacet functions or these are injected
    const DiamondCutFacet = await ethers.getContractFactory("DiamondCutFacet");
    const diamondCutFacet = await DiamondCutFacet.attach(diamond.address);

    // Prepare facet cuts
    const facetCuts = [
        {
            facetAddress: facetA.address,
            action: FacetCutAction.Add,
            functionSelectors: FacetA.interface.getSighash("function1()"), // Example selector
        },
        {
            facetAddress: facetB.address,
            action: FacetCutAction.Add,
            functionSelectors: FacetB.interface.getSighash("function2()"), // Example selector
        }
    ];

    // Perform the diamond cut to add facets
    console.log("Adding facets to Diamond...");
    const diamondCutTx = await diamondCutFacet.diamondCut(
        facetCuts,
        ethers.constants.AddressZero, // initContract
        "0x" // initData
    );
    await diamondCutTx.wait();
    console.log("Facets added to Diamond.");

    return diamond.address;
}

// Call the deployment function
// deployGemforceDiamond()
//     .then((address) => console.log(`Deployment complete. Diamond address: ${address}`))
//     .catch((error) => {
//         console.error(error);
//         process.exit(1);
//     });
```

### 1.2 Interacting with Marketplace Listing (Frontend Ethers.js)

This example shows how a frontend dApp user might list an ERC721 NFT on the Gemforce marketplace.

```typescript
// frontend/src/components/ListNFT.tsx
import React, { useState } from 'react';
import { ethers, Contract } from 'ethers';
import MarketplaceFacetABI from '../abi/MarketplaceFacet.json'; // ABI
import ERC721ABI from '../abi/ERC721.json'; // ERC721 ABI for approval

const GEMFORCE_DIAMOND_ADDRESS = "0x..."; // Replace with your deployed Diamond address
const MARKETPLACE_FACET_ADDRESS = GEMFORCE_DIAMOND_ADDRESS; // Interact via the Diamond address
const WETH_ADDRESS = "0x..."; // Example WETH address for payment token

const ListNFT: React.FC = () => {
    const [nftContractAddress, setNftContractAddress] = useState('');
    const [tokenId, setTokenId] = useState('');
    const [price, setPrice] = useState('');
    const [loading, setLoading] = useState(false);
    const [message, setMessage] = useState('');

    const connectWallet = async () => {
        if (!window.ethereum) {
            setMessage("MetaMask or compatible wallet not detected.");
            return null;
        }
        await window.ethereum.request({ method: 'eth_requestAccounts' });
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        return provider.getSigner();
    };

    const handleListNFT = async () => {
        setLoading(true);
        setMessage('');
        try {
            const signer = await connectWallet();
            if (!signer) return;

            const nftContract = new Contract(nftContractAddress, ERC721ABI, signer);
            const marketplace = new Contract(MARKETPLACE_FACET_ADDRESS, MarketplaceFacetABI, signer);

            // 1. Approve the Marketplace Facet to transfer the NFT
            setMessage("Approving NFT transfer...");
            const approveTx = await nftContract.approve(MARKETPLACE_FACET_ADDRESS, tokenId);
            await approveTx.wait();
            setMessage(`Approval successful. Tx: ${approveTx.hash}`);

            // 2. List the NFT on the marketplace
            setMessage("Listing NFT on marketplace...");
            // Use ethers.utils.parseEther for values in ETH; otherwise convert to Wei
            const priceInWei = ethers.utils.parseEther(price);
            const listTx = await marketplace.listItem(
                nftContractAddress,
                tokenId,
                priceInWei,
                WETH_ADDRESS // Or ethers.constants.AddressZero for native ETH
            );
            await listTx.wait();
            
            setMessage(`NFT listed successfully! Tx: ${listTx.hash}`);
            setNftContractAddress('');
            setTokenId('');
            setPrice('');
        } catch (error: any) {
            console.error("Error listing NFT:", error);
            setMessage(`Error: ${error.message || "Something went wrong."}`);
        } finally {
            setLoading(false);
        }
    };

    return (
        <div>
            <h2>List NFT on Gemforce Marketplace</h2>
            <input
                type="text"
                placeholder="NFT Contract Address"
                value={nftContractAddress}
                onChange={(e) => setNftContractAddress(e.target.value)}
            />
            <input
                type="number"
                placeholder="Token ID"
                value={tokenId}
                onChange={(e) => setTokenId(e.target.value)}
            />
            <input
                type="text"
                placeholder="Price (ETH)"
                value={price}
                onChange={(e) => setPrice(e.target.value)}
            />
            <button onClick={handleListNFT} disabled={loading}>
                {loading ? 'Processing...' : 'List NFT'}
            </button>
            {message && <p>{message}</p>}
        </div>
    );
};

export default ListNFT;
```

### 1.3 Listening for Smart Contract Events (Backend Node.js)

This Node.js example demonstrates how a backend service can listen for `TradeDealCreated` events from the `TradeDealManagementFacet` of the Gemforce Diamond.

```typescript
// backend/src/listeners/tradeDealListener.ts
import { ethers } from 'ethers';
import TradeDealManagementFacetABI from '../../abi/TradeDealManagementFacet.json'; // ABI

const GEMFORCE_DIAMOND_ADDRESS = "0x..."; // Replace with your deployed Diamond address
const RPC_URL = "wss://base-sepolia.public.blastapi.io"; // Use a WebSocket provider for real-time events

const setupTradeDealListener = () => {
    try {
        const provider = new ethers.providers.WebSocketProvider(RPC_URL);

        // Create a contract instance focused on the TradeDealManagementFacet's events
        const tradeDealManagement = new ethers.Contract(
            GEMFORCE_DIAMOND_ADDRESS,
            TradeDealManagementFacetABI,
            provider
        );

        console.log("Listening for 'TradeDealCreated' events...");

        // Event signature: event TradeDealCreated(bytes32 indexed tradeDealId, address indexed creator, uint256 indexed amount, address assetToken);
        tradeDealManagement.on("TradeDealCreated", (tradeDealId, creator, amount, assetToken, event) => {
            console.log("\n--- New Trade Deal Created! ---");
            console.log("  Trade Deal ID:", tradeDealId);
            console.log("  Creator:", creator);
            console.log("  Amount:", ethers.utils.formatEther(amount)); // Assuming amount in Wei
            console.log("  Asset Token:", assetToken);
            console.log("  Transaction Hash:", event.transactionHash);
            console.log("-------------------------------");

            // Here, you would typically:
            // - Update your backend database (e.g., save the new trade deal)
            // - Trigger notifications (e.g., push notification to users)
            // - Initiate further off-chain processing
        });

    } catch (error) {
        console.error("Error setting up Trade Deal listener:", error);
    }
};

// Start the listener when the application boots up
// setupTradeDealListener();
```

## 2. Cloud Function & REST API Usage Examples

### 2.1 User Authentication (Frontend JavaScript)

This example demonstrates user signup and login using direct Parse REST API calls in a web browser.

```javascript
// frontend/src/services/authService.js
const PARSE_SERVER_URL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";
const APP_ID = "YOUR_PARSE_APP_ID";
const CLIENT_KEY = "YOUR_PARSE_CLIENT_KEY"; 

export const signUpUser = async (username, password, email) => {
    try {
        const response = await fetch(`${PARSE_SERVER_URL}/users`, {
            method: 'POST',
            headers: {
                'X-Parse-Application-Id': APP_ID,
                'X-Parse-REST-API-Key': CLIENT_KEY,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ username, password, email })
        });
        const data = await response.json();
        if (!response.ok) throw new Error(data.error || "Signup failed");
        console.log("User signed up:", data);
        return data; // Contains sessionToken
    } catch (error) {
        console.error("Signup error:", error);
        throw error;
    }
};

export const loginUser = async (username, password) => {
    try {
        const response = await fetch(`${PARSE_SERVER_URL}/login?username=${username}&password=${password}`, {
            method: 'GET',
            headers: {
                'X-Parse-Application-Id': APP_ID,
                'X-Parse-REST-API-Key': CLIENT_KEY
            }
        });
        const data = await response.json();
        if (!response.ok) throw new Error(data.error || "Login failed");
        console.log("User logged in, session token:", data.sessionToken);
        // Store sessionToken securely (e.g., localStorage, secure cookies)
        localStorage.setItem('sessionToken', data.sessionToken);
        return data.sessionToken;
    } catch (error) {
        console.error("Login error:", error);
        throw error;
    }
};

export const callAuthenticatedCloudFunction = async (functionName, params) => {
    const sessionToken = localStorage.getItem('sessionToken');
    if (!sessionToken) throw new Error("No active session token found.");

    try {
        const response = await fetch(`${PARSE_SERVER_URL}/functions/${functionName}`, {
            method: 'POST',
            headers: {
                'X-Parse-Application-Id': APP_ID,
                'X-Parse-REST-API-Key': CLIENT_KEY,
                'X-Parse-Session-Token': sessionToken,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(params)
        });
        const data = await response.json();
        if (!response.ok) throw new Error(data.error || `Cloud function ${functionName} failed`);
        console.log(`Cloud function ${functionName} result:`, data.result);
        return data.result;
    } catch (error) {
        console.error(`Cloud function call error for ${functionName}:`, error);
        throw error;
    }
};

// Example usage:
// (async () => {
//     try {
//         // await signUpUser("sample_user", "secure_password", "user@example.com");
//         const token = await loginUser("sample_user", "secure_password");
//         const result = await callAuthenticatedCloudFunction("getUserProfile", { userId: "someUserId" });
//     } catch (e) {
//         console.error("Auth/Cloud Function Demo Error:", e.message);
//     }
// })();
```

### 2.2 Managing a Custom Object (Node.js Backend)

This Node.js example demonstrates CRUD operations for a custom Parse `Product` class using the Parse SDK directly.

```javascript
// backend/src/services/productService.js
const Parse = require('parse/node');

// Ensure Parse SDK is initialized (e.g., in your app's entry point)
// Parse.initialize("YOUR_PARSE_APP_ID", "YOUR_PARSE_JAVASCRIPT_KEY", "YOUR_PARSE_MASTER_KEY");
// Parse.serverURL = "YOUR_GEMFORCE_PARSE_SERVER_URL/parse";

export const createProduct = async (name, description, price) => {
    try {
        const Product = Parse.Object.extend("Product");
        const product = new Product();
        product.set("name", name);
        product.set("description", description);
        product.set("price", price);
        product.set("isInStock", true);

        // Save the object, using Master Key for backend operations
        const savedProduct = await product.save(null, { useMasterKey: true });
        console.log('New product created with objectId: ' + savedProduct.id);
        return savedProduct;
    } catch (error) {
        console.error('Error creating product:', error);
        throw error;
    }
};

export const getProduct = async (objectId) => {
    try {
        const Product = Parse.Object.extend("Product");
        const query = new Parse.Query(Product);
        // Use Master Key for backend operations
        const product = await query.get(objectId, { useMasterKey: true });
        console.log('Retrieved product:', product.toJSON());
        return product;
    } catch (error) {
        console.error('Error getting product:', error);
        throw error;
    }
};

export const updateProduct = async (objectId, updates) => {
    try {
        const Product = Parse.Object.extend("Product");
        const query = new Parse.Query(Product);
        const product = await query.get(objectId, { useMasterKey: true }); // Retrieve first
        
        for (const key in updates) {
            product.set(key, updates[key]);
        }
        // Save updates
        const updatedProduct = await product.save(null, { useMasterKey: true });
        console.log('Product updated:', updatedProduct.toJSON());
        return updatedProduct;
    } catch (error) {
        console.error('Error updating product:', error);
        throw error;
    }
};

export const deleteProduct = async (objectId) => {
    try {
        const Product = Parse.Object.extend("Product");
        const query = new Parse.Query(Product);
        const product = await query.get(objectId, { useMasterKey: true });
        await product.destroy({ useMasterKey: true });
        console.log('Product deleted successfully.');
    } catch (error) {
        console.error('Error deleting product:', error);
        throw error;
    }
};

// Example Usage:
// (async () => {
//     try {
//         const newProduct = await createProduct("Gemstone", "Rare blue sapphire", 5000);
//         const productId = newProduct.id;
//         await getProduct(productId);
//         await updateProduct(productId, { price: 4500, isInStock: false });
//         await deleteProduct(productId);
//     } catch (e) {
//         console.error("Product Service Demo Error:", e.message);
//     }
// })();
```

## 3. Advanced Integration Examples

### 3.1 DFNS Transaction Signing (Backend Node.js)

This example demonstrates how to use the DFNS SDK to securely sign an EVM transaction on your backend using a DFNS wallet.

```typescript
// backend/src/services/dfnsSigner.ts
import { DfnsApiClient, AsymmetricKeys, SignatureMechanism } from '@dfns/sdk';
import { EvmTransaction } from '@dfns/sdk/codegen/datamodel/EvmTransaction'; 
import { ethers } from 'ethers';

const DFNS_API_URL = process.env.DFNS_API_URL || "https://api.dfns.io";
const DFNS_APP_ID = process.env.DFNS_APP_ID || "YOUR_DFNS_APP_ID";
const DFNS_SIGNING_KEY_ID = process.env.DFNS_SIGNING_KEY_ID || "sk-xxxxxxxxxxxxxxxxx";
const DFNS_PRIVATE_KEY = process.env.DFNS_PRIVATE_KEY || "YOUR_DFNS_PRIVATE_KEY_PEM"; // Absolute path to PEM file or actual PEM string

let dfnsClient: DfnsApiClient;

function initializeDfnsClient() {
    if (!dfnsClient) {
        dfnsClient = new DfnsApiClient({
            baseUrl: DFNS_API_URL,
            appId: DFNS_APP_ID,
            authMethod: new AsymmetricKeys({
                privateKey: DFNS_PRIVATE_KEY,
                signingKeyId: DFNS_SIGNING_KEY_ID,
            })
        });
    }
    return dfnsClient;
}

export const signEvmTransaction = async (walletId: string, to: string, value: ethers.BigNumberish, data: string, chainId: string, nonce: number, gasLimit: ethers.BigNumberish) => {
    initializeDfnsClient();

    // Prepare the raw EVM transaction payload
    const rawTransaction: EvmTransaction = {
        to: to,
        value: ethers.BigNumber.from(value).toHexString(), // Convert to hex string
        data: data,
        gasLimit: ethers.BigNumber.from(gasLimit).toHexString(),
        chainId: ethers.BigNumber.from(chainId).toHexString(),
        nonce: ethers.BigNumber.from(nonce).toHexString(),
        type: "0x0", // Or "0x2" for EIP-1559, then include maxFeePerGas, maxPriorityFeePerGas
        // For simplicity, omitting gasPrice / EIP-1559 fields for now
    };

    try {
        const signRequest = {
            walletId: walletId,
            chainId: chainId, // Format "eip155:<chain_id>" for DFNS if needed, otherwise just number
            payload: JSON.stringify(rawTransaction),
            signatureMechanism: SignatureMechanism.JsonRpc, // Or Eip1559, etc.
            transactionType: "EvmTransaction",
        };

        const response = await dfnsClient.wallet.signTransaction(signRequest);
        console.log("Transaction signed by DFNS:", response.signedData);
        return response.signedData; // This is the fully signed hex string to broadcast
    } catch (error) {
        console.error("DFNS signing error:", error);
        throw error;
    }
};

// Example usage:
// (async () => {
//     try {
//         const dfnsWalletUuid = "YOUR_DFNS_WALLET_UUID";
//         const targetChainId = "11155111"; // Sepolia
//         const recipientAddress = "0x...";
//         const amountToSend = ethers.utils.parseEther("0.001");
//         const txData = "0x"; // Empty data for simple transfer

//         // You would typically get nonce and gasLimit from your RPC provider
//         const provider = new ethers.providers.JsonRpcProvider("https://sepolia.base.org");
//         const currentNonce = await provider.getTransactionCount(dfnsWalletUuid);
//         const gasLimit = 21000; // For simple ETH transfer

//         const signedTx = await signEvmTransaction(
//             dfnsWalletUuid,
//             recipientAddress,
//             amountToSend,
//             txData,
//             targetChainId,
//             currentNonce,
//             gasLimit
//         );

//         // Then broadcast the signed transaction
//         // const broadcastProvider = new ethers.providers.JsonRpcProvider("https://sepolia.base.org");
//         // const receipt = await broadcastProvider.sendTransaction(signedTx);
//         // await receipt.wait();
//         // console.log("Transaction broadcasted and confirmed:", receipt.hash);
//     } catch (e) {
//         console.error("DFNS Sample Demo Error:", e.message);
//     }
// })();
```

### 3.2 Consuming a Webhook (Node.js with Express)

This example provides a basic Node.js Express server to receive and process webhooks from the Gemforce platform.

```javascript
// backend/src/webhooks/webhookConsumer.ts
const express = require('express');
const bodyParser = require('body-parser');
const crypto = require('crypto');

const app = express();
const PORT = process.env.WEBHOOK_PORT || 3001; // Listen on a different port than main app
const WEBHOOK_SECRET = process.env.GEMFORCE_WEBHOOK_SECRET || "aStrongAndRandomSecretKeyForVerification"; // Must match config in Gemforce

app.use(bodyParser.json());

app.post('/gemforce-webhook', (req, res) => {
    console.log('Received HTTP POST to /gemforce-webhook');

    // 1. Verify Signature (Crucial for security)
    const signature = req.headers['x-gemforce-signature'] as string; // Header sent by Gemforce
    if (!signature) {
        console.warn('Webhook received without signature. Rejecting.');
        return res.status(401).send('Signature Missing');
    }

    try {
        const hmac = crypto.createHmac('sha256', WEBHOOK_SECRET);
        hmac.update(JSON.stringify(req.body));
        const calculatedSignature = hmac.digest('hex');

        if (signature !== calculatedSignature) {
            console.warn('Webhook signature verification failed. Possible tampering.');
            return res.status(403).send('Invalid Signature');
        }
        console.log('Webhook signature successfully verified.');

    } catch (error) {
        console.error('Error during signature verification:', error);
        return res.status(500).send('Signature Verification Error');
    }

    // 2. Process the Payload
    const eventType = req.body.event?.type;
    const eventData = req.body.data;

    if (!eventType || !eventData) {
        console.warn('Malformed webhook payload received. Missing event type or data.');
        return res.status(400).send('Malformed Payload');
    }

    console.log(`Processing event type: ${eventType}`);
    // console.log('Event Data:', JSON.stringify(eventData, null, 2));

    switch (eventType) {
        case 'user.created':
            console.log(`New User Created: ${eventData.object?.username} (ID: ${eventData.object?.objectId})`);
            // Add your logic: e.g., send welcome email, update user stats
            break;
        case 'marketplace.itemListed':
            console.log(`Marketplace Item Listed: NFT ${eventData.tokenId} at price ${ethers.utils.formatEther(eventData.price)} ETH`);
            // Add your logic: e.g., index item in a search database, notify subscribers
            break;
        case 'tradeDeal.statusUpdated':
            console.log(`Trade Deal ${eventData.tradeDealId} status changed to ${eventData.newStatus}`);
            // Update internal trade deal status, trigger further payment processing
            break;
        // Add more cases for other event types
        default:
            console.log(`Received unhandled event type: ${eventType}`);
            break;
    }

    // Acknowledge receipt
    res.status(200).send('Webhook received and processed');
});

// Start the webhook listener
// app.listen(PORT, () => {
//     console.log(`Gemforce Webhook consumer listening on port ${PORT}`);
// });

export default app; // Export the app if used in a larger project