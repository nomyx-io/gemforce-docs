# SDK & Libraries: Blockchain Utilities

This document describes the `blockchain.ts` library, a core utility module within the Gemforce SDK designed to simplify interactions with various blockchain networks. It provides a set of helper functions for common tasks such as obtaining RPC providers, managing transactions, estimating gas, and handling network-specific configurations.

## Overview

The `blockchain.ts` library offers:

-   **Provider Management**: Convenient access to `ethers.js` Provider instances for different networks.
-   **Transaction Helpers**: Functions to build, sign, and send transactions.
-   **Gas Estimation**: Utilities for accurate gas price and limit estimation.
-   **Network Configuration**: Centralized management of network-specific details (RPC URLs, chain IDs, contract addresses).
-   **Error Handling**: Standardized error handling for blockchain interactions.

This library aims to abstract away much of the boilerplate associated with direct `ethers.js` or `web3.js` usage, making blockchain interactions more developer-friendly within the Gemforce ecosystem.

## Key Functions

### 1. `getProvider(network: string)`

Retrieves an `ethers.js` `JsonRpcProvider` instance for the specified blockchain network.

**Function Signature**:
`getProvider(network: string): Promise<ethers.providers.JsonRpcProvider>`

**Parameters**:

-   `network` (String, required): The name of the blockchain network (e.g., "sepolia", "base-sepolia", "optimism-sepolia").

**Returns**:
-   `Promise<ethers.providers.JsonRpcProvider>`: A Promise that resolves to an `ethers.js` provider instance.

**Example Usage**:

```typescript
import { getProvider } from '@gemforce-sdk/blockchain'; // Assuming this import path

async function fetchBlockNumber(network: string) {
    try {
        const provider = await getProvider(network);
        const blockNumber = await provider.getBlockNumber();
        console.log(`Current block number on ${network}: ${blockNumber}`);
        return blockNumber;
    } catch (error) {
        console.error(`Error fetching block number on ${network}:`, error);
        throw error;
    }
}

// Example: fetchBlockNumber("sepolia");
```

### 2. `sendSignedTransaction(signedTx: string, network: string)`

Broadcasts a signed transaction to the specified blockchain network.

**Function Signature**:
`sendSignedTransaction(signedTx: string, network: string): Promise<ethers.providers.TransactionReceipt>`

**Parameters**:

-   `signedTx` (String, required): The raw, RLP-encoded, signed transaction hex string (e.g., "0x...").
-   `network` (String, required): The blockchain network where the transaction should be sent.

**Returns**:
-   `Promise<ethers.providers.TransactionReceipt>`: A Promise that resolves to the transaction receipt once it's mined.

**Example Usage**:

```typescript
import { sendSignedTransaction } from '@gemforce-sdk/blockchain';

// Assuming 'rawSignedTransaction' is obtained from a secure signing service like DFNS
async function broadcastTransaction(rawSignedTransaction: string, network: string) {
    try {
        console.log("Broadcasting transaction...");
        const receipt = await sendSignedTransaction(rawSignedTransaction, network);
        console.log("Transaction confirmed:", receipt.transactionHash);
        return receipt;
    } catch (error) {
        console.error("Error broadcasting transaction:", error);
        throw error;
    }
}

// Example: broadcastTransaction("0x...", "base-sepolia");
```

### 3. `getTransactionReceipt(txHash: string, network: string)`

Retrieves the transaction receipt for a given transaction hash.

**Function Signature**:
`getTransactionReceipt(txHash: string, network: string): Promise<ethers.providers.TransactionReceipt | null>`

**Parameters**:

-   `txHash` (String, required): The hash of the transaction.
-   `network` (String, required): The blockchain network where the transaction was sent.

**Returns**:
-   `Promise<ethers.providers.TransactionReceipt | null>`: A Promise that resolves to the transaction receipt or `null` if not found.

### 4. `getGasPrice(network: string)`

Estimates the current gas price for the specified network.

**Function Signature**:
`getGasPrice(network: string): Promise<ethers.BigNumber>`

**Parameters**:

-   `network` (String, required): The blockchain network.

**Returns**:
-   `Promise<ethers.BigNumber>`: A Promise that resolves to the estimated gas price in Wei.

### 5. `getNetworkConfig(network: string)`

Retrieves the configuration details for a specific blockchain network, as defined within the SDK.

**Function Signature**:
`getNetworkConfig(network: string): { rpcUrl: string; chainId: number; contractAddresses: { [key: string]: string } }`

**Parameters**:

-   `network` (String, required): The name of the blockchain network.

**Returns**:
-   `Object`: An object containing `rpcUrl`, `chainId`, and `contractAddresses` (mapping contract names to their deployed addresses).

### 6. `callContractFunction(contractAddress: string, abi: any[], functionName: string, args: any[], network: string)`

Executes a `view` or `pure` function on a smart contract. No transaction is sent, and no gas is consumed.

**Function Signature**:
`callContractFunction(contractAddress: string, abi: any[], functionName: string, args: any[], network: string): Promise<any>`

**Parameters**:

-   `contractAddress` (String, required): The address of the target smart contract.
-   `abi` (Array, required): The ABI of the contract (or relevant facet).
-   `functionName` (String, required): The name of the function to call.
-   `args` (Array, required): An array of arguments for the function.
-   `network` (String, required): The blockchain network.

**Returns**:
-   `Promise<any>`: A Promise that resolves to the result of the contract call.

## Error Handling

The `blockchain.ts` library internally handles common `ethers.js` errors and re-throws them, or wraps them with more context-specific messages. Expected errors include:

-   `NETWORK_ERROR`: Problems connecting to the RPC provider.
-   `TRANSACTION_REVERTED`: Smart contract function reverted.
-   `INSUFFICIENT_FUNDS`: Sender account lacks necessary funds for gas.
-   `INVALID_ARGUMENT`: Invalid parameters passed to a function.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Private Key Management**: This library focuses on transaction broadcasting. Private key management (signing) should always be handled by secure external services (like DFNS) or secure local key stores, NOT directly within this utility.
-   **RPC Endpoint Security**: Ensure that the RPC endpoints configured are trusted and secure to prevent data interception or manipulation.
-   **Input Validation**: Although the library uses `ethers.js` for type checking, always validate inputs to functions before calling them to prevent unexpected behavior or errors.

## Related Documentation

-   [Integrator's Guide: Smart Contracts](../integrator-guide/smart-contracts.md)
-   [Integrator's Guide: Authentication](../integrator-guide/authentication.md)
-   [Ethers.js Documentation](https://docs.ethers.org/) (External)