# SDK & Libraries: Contract Utilities

This document describes the `contract.ts` library, a utility module within the Gemforce SDK designed to streamline common interactions with arbitrary smart contracts. It builds upon `ethers.js` by providing helper functions for instantiating contract objects, handling generic function calls, and managing contract-specific configurations.

## Overview

The `contract.ts` library offers:

-   **Contract Instantiation**: Simplified creation of `ethers.Contract` instances.
-   **Generic Call/Transaction**: Functions to execute read (view, pure) and write (state-changing) operations on any contract given its address and ABI.
-   **ABI Management**: Helper for loading or providing contract ABIs.
-   **Type-Safe Defaults**: Can be used with pre-defined Gemforce contract ABIs for type safety and convenience.

This library is particularly useful for interacting with external ERC- стандартов or custom contracts beyond the core Gemforce Diamond setup.

## Key Functions

### 1. `getContractInstance(contractAddress: string, abi: any[], network: string, signer?: ethers.Signer)`

Retrieves an `ethers.js` `Contract` instance tailored for the specified contract.

**Function Signature**:
`getContractInstance(contractAddress: string, abi: any[], network: string, signer?: ethers.Signer): Promise<ethers.Contract>`

**Parameters**:

-   `contractAddress` (String, required): The address of the target smart contract.
-   `abi` (Array, required): The ABI (Application Binary Interface) of the contract.
-   `network` (String, required): The blockchain network.
-   `signer` (ethers.Signer, optional): An `ethers.Signer` object if state-changing functions are to be called or if a specific account is needed for read calls. If not provided, a `Provider` is used for read-only access.

**Returns**:
-   `Promise<ethers.Contract>`: A Promise that resolves to an `ethers.js` `Contract` instance.

**Example Usage**:

```typescript
import { getContractInstance } from '@gemforce-sdk/contract'; // Assuming this import path
import { ethers } from 'ethers';

// Example: Interacting with a generic ERC-20 token
const ERC20_ABI = [
    "function name() view returns (string)",
    "function symbol() view returns (string)",
    "function balanceOf(address owner) view returns (uint256)",
    "function transfer(address to, uint256 amount) returns (bool)",
    "event Transfer(address indexed from, address indexed to, uint256 value)"
];

async function getTokenInfo(tokenAddress: string, userAddress: string, network: string) {
    try {
        const tokenContract = await getContractInstance(tokenAddress, ERC20_ABI, network);
        
        const name = await tokenContract.name();
        const symbol = await tokenContract.symbol();
        const balance = await tokenContract.balanceOf(userAddress);

        console.log(`Token: ${name} (${symbol})`);
        console.log(`Balance of ${userAddress}: ${ethers.utils.formatUnits(balance, 18)} ${symbol}`); // Assuming 18 decimals
        return { name, symbol, balance };
    } catch (error) {
        console.error(`Error fetching token info for ${tokenAddress}:`, error);
        throw error;
    }
}

// Example: getTokenInfo("0x...USDC_Address...", "0xYourWalletAddress", "sepolia");
```

### 2. `callContractRead(contract: ethers.Contract, functionName: string, args?: any[])`

Executes a `view` or `pure` function (read-only) on an already instantiated contract.

**Function Signature**:
`callContractRead(contract: ethers.Contract, functionName: string, args?: any[]): Promise<any>`

**Parameters**:

-   `contract` (ethers.Contract, required): An `ethers.js` `Contract` instance (obtained via `getContractInstance` or similar).
-   `functionName` (String, required): The name of the function to call.
-   `args` (Array, optional): An array of arguments for the function.

**Returns**:
-   `Promise<any>`: A Promise that resolves to the result of the contract call.

### 3. `executeContractWrite(contract: ethers.Contract, functionName: string, args: any[], txOverrides?: ethers.Overrides)`

Executes a state-changing function (write operation) on an already instantiated contract, typically requiring a `signer` in the contract instance.

**Function Signature**:
`executeContractWrite(contract: ethers.Contract, functionName: string, args: any[], txOverrides?: ethers.Overrides): Promise<ethers.providers.TransactionReceipt>`

**Parameters**:

-   `contract` (ethers.Contract, required): An `ethers.js` `Contract` instance that includes a `Signer`.
-   `functionName` (String, required): The name of the function to call.
-   `args` (Array, required): An array of arguments for the function.
-   `txOverrides` (ethers.Overrides, optional): An object containing transaction overrides like `gasLimit`, `gasPrice`, `value` (for payable functions).

**Returns**:
-   `Promise<ethers.providers.TransactionReceipt>`: A Promise that resolves to the transaction receipt once it's mined.

**Example Usage (Continuing ERC-20 example)**:

```typescript
import { getContractInstance, executeContractWrite } from '@gemforce-sdk/contract';
import { ethers } from 'ethers';

async function transferTokens(tokenAddress: string, fromSigner: ethers.Signer, toAddress: string, amount: ethers.BigNumberish, network: string) {
    try {
        const ERC20_ABI_TRANSFER = [
            "function transfer(address to, uint256 amount) returns (bool)",
        ];
        // Get instance with a signer
        const tokenContract = await getContractInstance(tokenAddress, ERC20_ABI_TRANSFER, network, fromSigner);
        
        console.log(`Transferring ${ethers.utils.formatUnits(amount, 18)} tokens from ${await fromSigner.getAddress()} to ${toAddress}...`);
        
        const receipt = await executeContractWrite(tokenContract, "transfer", [toAddress, amount]);
        console.log("Transfer successful! Transaction hash:", receipt.transactionHash);
        return receipt;
    } catch (error) {
        console.error("Error transferring tokens:", error);
        throw error;
    }
}

// Example:
// const signer = // ... obtain a connected ethers.Signer (e.g., from MetaMask or a Wallet);
// transferTokens("0x...USDC_Address...", signer, "0xAnotherWalletAddress", ethers.utils.parseUnits("10", 6), "sepolia"); // Assuming 6 decimals for USDC
```

## Error Handling

The `contract.ts` library propagates `ethers.js` errors, providing a consistent way to handle blockchain-related exceptions. Common errors include those related to network issues, transaction reverts, insufficient funds, and invalid contract interactions.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for detailed error handling strategies.

## Security Considerations

-   **ABI Trust**: Always ensure the ABI provided to `getContractInstance` is correct and from a trusted source. A malicious ABI could lead to unexpected behavior.
-   **Signer Security**: Any function that performs write operations relies heavily on the `signer` provided. Ensure private key management is handled securely outside this library (e.g., via DFNS or user's browser wallet).
-   **Input Validation**: Thoroughly validate all arguments passed to contract functions to prevent common smart contract vulnerabilities.

## Related Documentation

-   [SDK & Libraries: Blockchain Utilities](blockchain.md)
-   [Integrator's Guide: Smart Contracts](../integrator-guide/smart-contracts.md)
-   [Ethers.js Documentation](https://docs.ethers.org/) (External)