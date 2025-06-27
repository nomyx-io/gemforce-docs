# SDK & Libraries: Deployment Utilities

This document describes the `deploy.ts` library, a utility module within the Gemforce SDK designed to simplify and automate smart contract deployment processes. It provides functions to deploy new contract instances, verify deployments on block explorers, and manage deployment configurations across various blockchain networks.

## Overview

The `deploy.ts` library offers:

-   **Contract Deployment**: Programmatic deployment of compiled smart contracts.
-   **Deployment Verification**: Tools to verify deployed contracts on block explorers (e.g., Etherscan).
-   **Configuration Management**: Integration with deployment configurations (network-specific settings, contract linkages).
-   **Gas Management**: Automated gas estimation and handling during deployment.

This library is particularly useful for CI/CD pipelines, automated testing environments, and any scenario requiring reliable, repeatable contract deployments.

## Key Functions

### 1. `deployContract(contractFactory: ethers.ContractFactory, args: any[], network: string)`

Deploys a new instance of a smart contract. Throws an error if deployment fails.

**Function Signature**:
`deployContract(contractFactory: ethers.ContractFactory, args: any[], network: string): Promise<{ address: string; transactionHash: string; }> `

**Parameters**:

-   `contractFactory` (ethers.ContractFactory, required): An `ethers.ContractFactory` instance for the contract to be deployed (obtained from Hardhat, Truffle, or `new ethers.ContractFactory(abi, bytecode, signer)`).
-   `args` (Array, required): An array of constructor arguments for the contract.
-   `network` (String, required): The blockchain network to deploy to.

**Returns**:
-   `Promise<{ address: string; transactionHash: string; }>`: A Promise that resolves to an object containing the deployed contract's address and the deployment transaction hash.

**Example Usage**:

```typescript
import { deployContract } from '@gemforce-sdk/deploy'; // Assuming this import path
import { ethers } from 'ethers';
import { getProvider } from '@gemforce-sdk/blockchain'; // From blockchain utilities
import MyContractABI from './MyContract.json'; // Compiled contract ABI
import MyContractBytecode from './MyContract.bytecode.json'; // Compiled contract bytecode

async function deployMyNewContract(network: string, constructorArg: string) {
    try {
        const provider = await getProvider(network);
        // Assuming a signer is available (e.g., from a connected wallet or private key)
        const signer = new ethers.Wallet(process.env.PRIVATE_KEY!, provider); // Securely manage private keys

        const MyContractFactory = new ethers.ContractFactory(MyContractABI, MyContractBytecode, signer);
        
        console.log(`Deploying MyContract on ${network}...`);
        const deployment = await deployContract(MyContractFactory, [constructorArg], network);
        
        console.log(`MyContract deployed to: ${deployment.address}`);
        console.log(`Deployment transaction hash: ${deployment.transactionHash}`);
        return deployment.address;
    } catch (error) {
        console.error(`Error deploying MyContract on ${network}:`, error);
        throw error;
    }
}

// Example: deployMyNewContract("sepolia", "initialValue");
```

### 2. `verifyContract(contractAddress: string, constructorArgs: any[], network: string, options?: { compilerVersion?: string; libraries?: { [name: string]: string } })`

Verifies a deployed smart contract's source code on a block explorer (e.g., Etherscan, Basescan). This function typically integrates with block explorer APIs.

**Function Signature**:
`verifyContract(contractAddress: string, constructorArgs: any[], network: string, options?: { compilerVersion?: string; libraries?: { [name: string]: string } }): Promise<boolean>`

**Parameters**:

-   `contractAddress` (String, required): The address of the deployed contract.
-   `constructorArgs` (Array, required): Array of original constructor arguments.
-   `network` (String, required): The blockchain network.
-   `options` (Object, optional):
    -   `compilerVersion` (String): Specific Solidity compiler version.
    -   `libraries` (Object): An object mapping linked library names to their deployed addresses.

**Returns**:
-   `Promise<boolean>`: `true` if verification was successful, `false` otherwise.

**Example Usage**:

```typescript
import { verifyContract } from '@gemforce-sdk/deploy';

async function verifyDeployedContract(deployedAddress: string, initialArg: string) {
    try {
        console.log(`Verifying contract ${deployedAddress}...`);
        const success = await verifyContract(deployedAddress, [initialArg], "sepolia");
        if (success) {
            console.log("Contract verified successfully on Etherscan!");
        } else {
            console.warn("Contract verification failed.");
        }
    } catch (error) {
        console.error("Error during contract verification:", error);
    }
}

// Example: verifyDeployedContract("0xDeployedContractAddress", "initialValue");
```

### 3. `getDeployedAddress(contractName: string, network: string)`

Retrieves the pre-configured deployed address of a known Gemforce contract by name and network. This relies on an internal configuration.

**Function Signature**:
`getDeployedAddress(contractName: string, network: string): string | undefined`

**Parameters**:

-   `contractName` (String, required): The name of the contract (e.g., "Diamond", "MarketplaceFacet").
-   `network` (String, required): The blockchain network.

**Returns**:
-   `String | undefined`: The deployed address if found, otherwise `undefined`.

## Error Handling

Deployment utilities can encounter errors such as:

-   **Transaction Failures**: Due to insufficient gas, reverts in constructor, or network issues.
-   **Verification Failures**: Explorer API issues, incorrect constructor arguments, or mismatched source code.
-   **Configuration Errors**: Missing network configurations or contract addresses.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Private Key Management**: Deployment requires a `Signer` with access to a private key. Ensure this private key is managed securely (e.g., through environment variables, secret management services, or dedicated deployment services). Never hardcode private keys.
-   **Constructor Arguments**: Carefully review and validate constructor arguments for production deployments. Errors here can lead to security vulnerabilities or misconfigured contracts.
-   **Verification**: Always strive to verify your deployed contracts on block explorers. This provides transparency and allows others to audit the deployed bytecode.
-   **Network Trust**: Ensure the RPC endpoints used for deployment are trusted and reliable.

## Related Documentation

-   [SDK & Libraries: Blockchain Utilities](blockchain.md)
-   [Deployment Guides: Multi-Network Deployment](../deployment-guides/multi-network-deployment.md)
-   [Hardhat Documentation](https://hardhat.org/docs) (External)