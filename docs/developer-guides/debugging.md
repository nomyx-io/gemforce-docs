# Developer Guides: Debugging

Effective debugging is a critical skill for any developer building on the Gemforce platform. This guide provides strategies and tools for identifying, isolating, and resolving issues across different layers of your application, from smart contracts on the blockchain to cloud functions and frontend interfaces.

## Overview of Debugging Challenges

Debugging in a decentralized environment introduces unique complexities:

-   **Immutability**: Once a transaction is on-chain, it cannot be changed.
-   **Transparency (but not always clarity)**: All transactions are publicly visible, but understanding why a transaction failed often requires deeper inspection.
-   **Asynchronous Operations**: Transactions are often asynchronous, requiring careful state management.
-   **Distributed Systems**: Issues can span multiple components (frontend, backend, blockchain nodes, smart contracts, external services).

## 1. Debugging Smart Contracts

Debugging smart contracts primarily involves understanding transaction execution flow, state changes, and revert reasons.

### 1.1 Hardhat & Foundry (Local Development)

Both Hardhat and Foundry provide excellent local development environments with powerful debugging capabilities.

-   **Transaction Tracing**: Hardhat local networks (and Foundry's Anvil) allow you to trace the execution of transactions, showing function calls, gas usage, and state changes step-by-step.
    -   **Hardhat**: Run tests with `npx hardhat test --verbose` or use `console.log` for Solidity. `hardhat-gas-reporter` helps with gas analysis.
    -   **Foundry**: `forge test -vvv` provides detailed call traces. `forge debug <transaction_hash>` allows interactive debugging.
-   **`console.log` for Solidity**: Hardhat and Foundry support a `console.log` equivalent in Solidity testing, enabling you to output values during test execution.
    ```solidity
    // In your Solidity test file or contract:
    import "hardhat/console.sol"; // For Hardhat
    // or import "forge-std/console.sol"; // For Foundry

    function myDebugFunction() public {
        uint256 myVar = 123;
        console.log("My var is:", myVar);
        // ...
    }
    ```
-   **Revert Reasons**: Ensure your `require()` and `revert()` statements in Solidity contracts have clear, descriptive error messages. `ethers.js` and `web3.js` can often extract these messages for better client-side error reporting.

### 1.2 Blockchain Explorers (Testnets)

For debugging on public testnets, blockchain explorers become invaluable.

-   **Tools**: Etherscan (and its forks like Basescan, Optimism Scan).
-   **Usage**:
    -   Paste your transaction hash into the explorer.
    -   Look for "Transaction failed" or "Revert" status.
    -   Examine the "Debug Trace" (or similar) tab to see the sequence of internal calls and identify where the transaction reverted.
    -   Inspect input/output data of calls, and internal transaction messages.
-   **Key Information**:
    -   **Gas Used**: High gas usage might indicate an infinite loop or unexpected complexity.
    -   **Revert Reason**: Human-readable error message.
    -   **Event Logs**: Verify expected events were emitted.

## 2. Debugging Cloud Functions & Backend Services

Gemforce's cloud functions (Parse Server) and other backend services are typically Node.js applications, which can be debugged using standard JavaScript debugging tools.

### 2.1 VS Code Debugger

VS Code has excellent built-in debugging capabilities for Node.js.

-   **Launch Configurations**: Create `launch.json` configurations in your `.vscode` folder.
    -   Example for a Cloud Function:
        ```json
        {
            "version": "0.2.0",
            "configurations": [
                {
                    "type": "node",
                    "request": "launch",
                    "name": "Debug Cloud Function (specific)",
                    "runtimeExecutable": "npm",
                    "runtimeArgs": ["run", "debug-cloud-function"], // Custom script in package.json
                    "env": {
                        "PARSE_SERVER_URL": "http://localhost:1337/parse",
                        "PARSE_APP_ID": "your_app_id",
                        // ... and other necessary env vars
                    },
                    "console": "integratedTerminal",
                    "internalConsoleOptions": "openOnSessionStart"
                }
            ]
        }
        ```
    -   Your `package.json` might have a script like: `"debug-cloud-function": "node --inspect-brk ./cloud/main.js"`.
-   **Breakpoints**: Set breakpoints directly in your `.ts` or `.js` files.
-   **Variables & Call Stack**: Inspect variable values and traverse the call stack during execution.

### 2.2 Logging

Comprehensive logging is your first line of defense for debugging backend issues.

-   **Parse Server Logs**: Configure your Parse Server instance to log detailed information. Log levels can be adjusted (e.g., `verbose`, `debug`).
-   **Cloud Function Logs**: Add `console.log()` statements directly in your Cloud Functions to inspect variable values, trace execution paths, and confirm parameters.
-   **Structured Logging**: For production, use a structured logging library (e.g., Winston, Pino) to output logs in JSON format, making them easier to ingest and analyze in log management systems.

## 3. Debugging Frontend Applications (dApps)

Debugging modern web dApps involves browser developer tools and understanding Web3 interactions.

### 3.1 Browser Developer Tools

-   **Console**: Check for JavaScript errors, network request failures, and `console.log()` outputs.
-   **Network Tab**: Monitor all HTTP requests (e.g., to Parse Server REST API), check their status codes and payloads.
-   **Sources Tab**: Set breakpoints in your React/Vue/Angular code to inspect state, props, and variable values.
-   **Performance Tab**: Analyze UI rendering performance and identify bottlenecks.

### 3.2 Web3 Wallet Extensions (MetaMask, etc.)

Your browser's Web3 wallet extension is crucial for debugging blockchain interactions directly from the frontend.

-   **Network Selector**: Ensure your wallet is connected to the correct network (testnet).
-   **Transaction History**: Inspect pending and completed transactions within the wallet UI.
-   **Gas Estimates**: Observe the gas estimates provided by the wallet for transactions.
-   **Custom RPC**: Configure custom RPC URLs if needed for local development networks.

## 4. Debugging External Service Integrations

When integrating with services like DFNS, or external APIs.

### 4.1 Request/Response Inspection

-   **HTTP Client Logging**: Enable verbose logging for your HTTP client (e.g., `axios` interceptors, `curl -v`) to see full request and response headers and bodies.
-   **DFNS Dashboard**: DFNS provides a console or API for inspecting transaction signing requests, policies applied, and audit trails.

### 4.2 Webhook Inspection

-   **Webhook Debuggers**: Use online webhook inspection services (e.g., `webhook.site`, `requestbin.com`) temporarily during development to capture and inspect incoming webhook payloads.
-   **Local Tunnels**: Use tools like `ngrok` or `localtunnel` to expose your local webhook development server to the internet, allowing external services to send webhooks to your machine.

## General Debugging Tips

-   **Reproduce the Bug**: The most important step. Understand the exact steps to consistently trigger the issue.
-   **Simplify the Problem**: Remove unnecessary components or code to create a minimal reproducible example.
-   **Divide and Conquer**: Isolate the problem to a specific layer (frontend, backend, smart contract, external service).
-   **Check Logs**: Always consult all available logs (frontend console, backend server logs, blockchain explorer traces).
-   **Version Control**: Use Git to isolate changes. Use `git bisect` to find the commit that introduced a bug.
-   **Community & Documentation**: Consult official documentation, community forums (e.g., Stack Overflow), and Gemforce's developer communities for common issues and solutions.
-   **Rubber Duck Debugging**: Explain the problem aloud to an inanimate object (or colleague). The act of explaining often reveals the solution.

## Related Documentation

-   [Developer Guides: Development Environment Setup](development-environment-setup.md)
-   [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)
-   [Integrator's Guide: Sample Code](../integrator-guide/sample-code.md)