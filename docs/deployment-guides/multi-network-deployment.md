# Deployment Guides: Multi-Network Deployment

Deploying Gemforce components across multiple blockchain networks is a common requirement for applications targeting broader reach, specific network features, or lower transaction costs. This guide details the considerations and strategies for managing deployments to various networks supported by Gemforce, including local development chains, Sepolia testnets (Base Sepolia, Optimism Sepolia, Ethereum Sepolia), and potential mainnets.

## Overview

Multi-network deployment involves:

-   **Network Configuration**: Managing RPC endpoints, chain IDs, and network-specific contract addresses.
-   **Deployment Strategy**: Automating deployments to target networks using tools like Hardhat or Foundry.
-   **Cross-Chain Interaction**: Designing applications to interact seamlessly across multiple deployed instances.
-   **Monitoring and Maintenance**: Keeping track of deployments and their health on each network.

## 1. Network Configuration

Effective multi-network deployment starts with robust network configuration management.

### Key Configuration Elements

-   **RPC URL**: The endpoint for a blockchain node (e.g., from Alchemy, Infura, Blast API). Use secure WebSocket (`wss://`) endpoints for event listening and HTTPS (`https://`) for transaction submission.
-   **Chain ID**: Unique identifier for each blockchain network.
-   **Contract Addresses**: Deployed addresses of core Gemforce contracts (Diamond, Facets, ERC-20s) will differ per network.
-   **Private Keys/Signers**: Accounts authorized to deploy and manage contracts on each network. Must be securely managed.

### Example (`hardhat.config.ts`)

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-ethers"; // For ethers.js integration
import "@nomicfoundation/hardhat-chai-matchers"; // For Chai matchers
import "dotenv/config"; // To load .env variables

const config: HardhatUserConfig = {
    solidity: {
        version: "0.8.20",
        settings: {
            optimizer: {
                enabled: true,
                runs: 200,
            },
        },
    },
    networks: {
        hardhat: {
            // Local test network
            chainId: 31337,
        },
        sepolia: {
            url: process.env.ETHEREUM_SEPOLIA_RPC_URL || "",
            accounts: process.env.DEPLOYER_PRIVATE_KEY
                ? [process.env.DEPLOYER_PRIVATE_KEY]
                : [],
            chainId: 11155111,
            gasPrice: 1000000000, // 1 Gwei
        },
        "base-sepolia": {
            url: process.env.BASE_SEPOLIA_RPC_URL || "",
            accounts: process.env.DEPLOYER_PRIVATE_KEY
                ? [process.env.DEPLOYER_PRIVATE_KEY]
                : [],
            chainId: 84532,
        },
        "optimism-sepolia": {
            url: process.env.OPTIMISM_SEPOLIA_RPC_URL || "",
            accounts: process.env.DEPLOYER_PRIVATE_KEY
                ? [process.env.DEPLOYER_PRIVATE_KEY]
                : [],
            chainId: 11155420,
        },
        // Add other networks like Polygon, Arbitrum, Mainnets etc.
    },
    etherscan: {
        // For contract verification on Etherscan-like explorers
        apiKey: {
            sepolia: process.env.ETHERSCAN_API_KEY || "",
            // Use specific API keys for each chain if needed
            "base-sepolia": process.env.BASESCAN_API_KEY || "",
            "optimism-sepolia": process.env.OPTIMISM_ETHERSCAN_API_KEY || "",
        },
        customChains: [
            // Custom chains need to be defined for verification if not standard
            {
                network: "base-sepolia",
                chainId: 84532,
                urls: {
                    apiURL: "https://api-sepolia.basescan.org/api",
                    browserURL: "https://sepolia.basescan.org"
                }
            }
        ]
    },
    gasReporter: {
        enabled: process.env.REPORT_GAS !== undefined,
        currency: "USD",
        token: "ETH",
        coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    },
};

export default config;
```

## 2. Deployment Strategy

Automating deployments is crucial for consistency across networks.

### Recommendations

-   **Scripted Deployments**: Use deployment scripts (e.g., Hardhat Deploy, Foundry Script) that can be parameterized by network.
-   **Deterministic Deployments (`CREATE2`)**: For critical contracts like the Diamond Factory, use `CREATE2` to generate predictable contract addresses across networks, simplifying address management.
-   **Contract Verification**: Automatically verify deployed contracts on block explorers. Integrate this into your deployment scripts or CI/CD pipeline (e.g., using `@nomiclabs/hardhat-etherscan`).
-   **Monorepo Structure**: If your project is a monorepo, structure deployment scripts to handle different packages or contract collections.
-   **Network-Specific Artifacts**: Store deployed contract addresses, ABIs, and transaction hashes in network-specific JSON files or databases. Keep these artifacts version-controlled.

### Example (Simplified Deploy Script)

```typescript
// scripts/deployFullGemforce.ts
import { ethers } from "hardhat";
import * as fs from 'fs';
import * as path from 'path';

async function deployContracts(network: string) {
    const [deployer] = await ethers.getSigners();
    console.log(`Deploying to ${network} with account: ${deployer.address}`);

    const networkConfig = await ethers.getJsonRpcProvider(network).getNetwork();
    const chainId = networkConfig.chainId;

    const deployedAddresses: { [key: string]: string } = {};

    // 1. Deploy LibDiamond
    const LibDiamond = await ethers.getContractFactory("LibDiamond");
    const libDiamond = await LibDiamond.deploy();
    await libDiamond.deployed();
    deployedAddresses.LibDiamond = libDiamond.address;
    console.log(`LibDiamond deployed to: ${libDiamond.address}`);

    // 2. Deploy Diamond
    const Diamond = await ethers.getContractFactory("Diamond", {
        libraries: {
            LibDiamond: libDiamond.address,
        },
    });
    const diamond = await Diamond.deploy(deployer.address, true); // Owner, add owner as facet
    await diamond.deployed();
    deployedAddresses.Diamond = diamond.address;
    console.log(`Diamond deployed to: ${diamond.address}`);

    // 3. Deploy and Add Facets
    const facetNames = ["MarketplaceFacet", "CarbonCreditFacet", "TradeDealManagementFacet"]; // Example facets
    const deployedFacets: { [key: string]: string } = {};
    const facetCuts = [];

    for (const name of facetNames) {
        const Facet = await ethers.getContractFactory(name);
        const facet = await Facet.deploy();
        await facet.deployed();
        deployedFacets[name] = facet.address;
        console.log(`${name} deployed to: ${facet.address}`);

        // Prepare cut for facets
        facetCuts.push({
            facetAddress: facet.address,
            action: 0, // AddFacet
            functionSelectors: facet.interface.getSighash("function1()") // Placeholder
        });
    }

    // Perform the Diamond Cut
    const diamondCutFacet = await ethers.getContractAt("IDiamondCut", diamond.address);
    const diamondCutTx = await diamondCutFacet.diamondCut(
        facetCuts,
        ethers.constants.AddressZero,
        "0x"
    );
    await diamondCutTx.wait();
    console.log("Diamond Cut performed.");

    // Save deployed addresses to a file
    const outputDir = path.join(__dirname, `../deployments/${network}`);
    if (!fs.existsSync(outputDir)) {
        fs.mkdirSync(outputDir, { recursive: true });
    }
    fs.writeFileSync(
        path.join(outputDir, 'addresses.json'),
        JSON.stringify({ ...deployedAddresses, ...deployedFacets }, null, 2)
    );
    console.log(`Deployment addresses saved to deployments/${network}/addresses.json`);

    // Optionally verify contracts (requires ETHERSCAN_API_KEY in .env)
    for (const name in deployedAddresses) {
        try {
            await ethers.run("verify:verify", {
                address: deployedAddresses[name],
                constructorArguments: name === "Diamond" ? [deployer.address, true] : [],
                libraries: name === "Diamond" ? { LibDiamond: deployedAddresses.LibDiamond } : {}
            });
            console.log(`Verified ${name} on Etherscan.`);
        } catch (error) {
            console.error(`Verification failed for ${name}:`, error);
        }
    }
    
    return deployedAddresses;
}

// To run: npx hardhat run scripts/deployFullGemforce.ts --network base-sepolia
// deployContracts(process.env.HARDHAT_NETWORK || "hardhat")
//     .then(() => process.exit(0))
//     .catch((error) => {
//         console.error(error);
//         process.exit(1);
//     });
```

## 3. Cross-Chain Interaction Patterns

Designing your application to interact with components deployed on different networks.

### Centralized Backend Orchestration

-   **Pattern**: A single backend service (e.g., your Gemforce Parse Server, or a dedicated microservice) acts as the coordinator. It manages RPC connections to various chains and directs transactions to the appropriate network.
-   **Use Case**: Ideal for automated processes, admin tools, or when you need a single point of truth for multi-chain state.
-   **Advantages**: Simplifies client-side logic, centralizes secure key management, provides better control over transaction management.
-   **Disadvantages**: Centralized bottleneck, introduces latency for cross-chain operations depending on backend processing.

### Frontend Direct Interaction (Multi-Wallet)

-   **Pattern**: Frontend dApp directly connects to multiple networks via the user's wallet (e.g., MetaMask network switching, WalletConnect).
-   **Use Case**: User-driven interactions on a specific network (e.g., voting on a governance contract on Ethereum, then minting an NFT on Polygon).
-   **Advantages**: Fully decentralized from the backend, leverages user's existing wallet setup.
-   **Disadvantages**: Can be complex for users to manage multiple networks, requires explicit network switching, potentially higher client-side resource usage.

### Hybrid Approaches

-   Combine backend orchestration for automated, critical functions (e.g., indexing, cross-chain bridges) with frontend direct interaction for user-facing actions.

## 4. Monitoring and Maintenance

Post-deployment, continuous monitoring and maintenance are essential for healthy multi-network deployments.

### Recommendations

-   **Blockchain Explorers**: Use block explorers for each chain to monitor transaction status, gas prices, and contract events.
-   **RPC Node Monitoring**: Monitor the health and latency of your RPC providers.
-   **Custom Dashboards**: Build dashboards (e.g., using Grafana, Dune Analytics) to track key metrics like transaction volume, active users per chain, and contract interactions.
-   **Alerting**: Set up alerts for failed transactions, high gas costs, contract errors, or unusual activity.
-   **Upgrade Procedures**: Have clear, tested procedures for upgrading contracts on each network.
-   **Emergency Playbooks**: Define playbooks for responding to network outages or security incidents on a specific chain.

## Related Documentation

-   [SDK & Libraries: Deploy Utilities](../sdk-libraries/deploy.md)
-   [SDK & Libraries: Blockchain Utilities](../sdk-libraries/blockchain.md)
-   [Integrator's Guide: Smart Contracts](../integrator-guide/smart-contracts.md)
-   [Developer Guides: Performance Optimization](../developer-guides/performance-optimization.md)