# Developer Guides: Testing Frameworks

Comprehensive testing is fundamental to developing reliable and secure applications on the Gemforce platform. This guide delves deeper into the specific testing frameworks and methodologies recommended for different layers of the Gemforce ecosystem: smart contracts, cloud functions, and frontend applications.

## Overview

Effective testing for Gemforce-integrated applications requires a multi-faceted approach, employing specialized tools for:

-   **Smart Contract Testing**: Ensuring on-chain logic is sound, secure, and gas-efficient.
-   **Cloud Function/Backend Testing**: Validating server-side business logic and API interactions.
-   **Frontend/dApp Testing**: Verifying user interfaces and end-to-end user flows.

This document serves as a practical reference for setting up and utilizing these testing frameworks.

## 1. Smart Contract Testing (Hardhat / Foundry)

Hardhat and Foundry are the leading development environments for Solidity smart contracts, offering robust testing capabilities.

### 1.1 Hardhat

Hardhat comes with its own testing environment, built on top of Mocha and Chai. It provides an `ethers.js` wrapper for easy contract interaction in tests.

-   **Installation**: Part of Hardhat project setup.
-   **Key Features**:
    -   Built-in Ethereum network for fast local testing.
    -   Support for JavaScript and TypeScript tests.
    -   Easy interaction with deployed contracts and accounts.
    -   `hardhat-ethers` and `hardhat-waffle` plugins for powerful assertions.
    -   Gas reporting and code coverage tools.
-   **Example (JavaScript / Hardhat)**:

    ```javascript
    // test/MyFacet.js
    const { expect } = require("chai");

    describe("MyFacet", function () {
        let MyFacet;
        let myFacet;
        let owner;
        let addr1;

        beforeEach(async function () {
            [owner, addr1] = await ethers.getSigners();
            MyFacet = await ethers.getContractFactory("MyFacet");
            myFacet = await MyFacet.deploy(); // Replace with Diamond deployment logic for integration tests
            await myFacet.deployed();
        });

        it("Should return the new value once it's changed", async function () {
            await myFacet.connect(owner).setValue(50);
            expect(await myFacet.getValue()).to.equal(50);
        });

        it("Should not allow non-owner to set value", async function () {
            await expect(myFacet.connect(addr1).setValue(20))
                .to.be.revertedWith("Ownable: caller is not the owner");
        });
    });
    ```

### 1.2 Foundry

Foundry is a Rust-based toolkit for Ethereum application development, known for its speed and developer experience, especially for writing tests in Solidity itself (`Forge`).

-   **Installation**: `curl -L https://foundry.paradigm.xyz | bash` and then `foundryup`.
-   **Key Features**:
    -   Tests written directly in Solidity (`.t.sol` files).
    -   Extremely fast execution due to native Rust compilation.
    -   Rich debugging experience (stack traces, call traces).
    -   `forge test` for running tests, `forge coverage` for coverage.
    -   Fuzzing capabilities.
-   **Example (Solidity / Forge)**:

    ```solidity
    // test/MyContract.t.sol
    pragma solidity ^0.8.0;

    import "forge-std/Test.sol";
    import "../contracts/MyContract.sol"; // Assuming MyContract.sol is your contract

    contract MyContractTest is Test {
        MyContract public myContract;

        function setUp() public {
            myContract = new MyContract();
        }

        function testSetValue() public {
            myContract.setValue(123);
            assertEq(myContract.getValue(), 123);
        }

        function testRevertWhenNotOwner() public {
            vm.expectRevert("Ownable: caller is not the owner");
            vm.prank(address(0x1)); // Pretend to be address 0x1
            myContract.setValue(456);
        }

        function testFuzz_SetValue(uint256 newValue) public {
            // Test with arbitrary inputs
            myContract.setValue(newValue);
            assertEq(myContract.getValue(), newValue);
        }
    }
    ```

## 2. Cloud Function & Backend Testing (Jest / Supertest)

For testing Gemforce Cloud Functions and other backend services (e.g., API endpoints, background jobs), Jest is a popular choice for unit and integration tests, often paired with `supertest` for HTTP API testing.

### 2.1 Jest

A delightful JavaScript testing framework with a focus on simplicity.

-   **Features**: Fast, built-in assertion library, mocking, coverage reporting.
-   **Installation**: `npm install --save-dev jest @types/jest ts-jest` (for TypeScript).
-   **Configuration**: Add `jest.config.js` or `package.json` entry.
-   **Example (Mocking Cloud Functions)**: (See [Integrator's Guide: Error Handling - Cloud Function Example](../integrator-guide/error-handling.md) for more details on mocking Parse SDK).

### 2.2 Supertest

A library for testing Node.js HTTP servers that allows you to make requests to your API and assert on the responses.

-   **Installation**: `npm install --save-dev supertest @types/supertest`.
-   **Approach**: Create a test-specific instance of your Parse Server Express app (if managed in code) or point to a running local instance.
-   **Example (API Endpoint Test)**: (See [Integrator's Guide: Sample Code - Supertest Example](../integrator-guide/sample-code.md) for more details).

## 3. Frontend / dApp Testing (Cypress / Playwright)

For testing the user interface and end-to-end user flows, including interactions with Web3 wallets and smart contracts. These frameworks run in real browsers.

### 3.1 Cypress

A fast, easy, and reliable testing for anything that runs in a browser.

-   **Features**: Time travel debugging, automatic waiting, real-time reloading.
-   **Installation**: `npm install cypress --save-dev`.
-   **Approach**: Write tests that simulate user actions (clicks, inputs, navigation) and assert on UI state, network requests, and eventually, blockchain interactions (though direct wallet interaction in E2E tests is complex).
-   **Considerations**: Direct Web3 wallet interaction in E2E tests can be challenging; often involves mocking the `window.ethereum` object or using specialized plugins.

### 3.2 Playwright

Enables reliable end-to-end testing for modern web apps. Developed by Microsoft, it supports Chromium, Firefox, and WebKit (Safari).

-   **Features**: Auto-waits for elements, supports multiple browsers, parallel execution, codegen (record tests).
-   **Installation**: `npm install --save-dev playwright @playwright/test`.
-   **Approach**: Similar to Cypress, but with broader browser support and often considered more stable for CI/CD.

## General Testing Principles

-   **Test Pyramid**: Focus on a high ratio of fast unit tests, a medium number of integration tests, and a small number of slower E2E tests.
-   **CI/CD Integration**: Automate tests to run in your continuous integration/continuous deployment pipeline.
-   **Code Coverage**: Use coverage tools (e.g., Jest `collectCoverage`, Hardhat/Foundry coverage reports) to identify untested parts of your codebase.
-   **Mocking**: Master mocking external services and complex dependencies to enable isolated and faster unit tests.
-   **Test Data Management**: Have clear strategies for setting up and tearing down test data for each test run.
-   **Testnets**: Always perform integration and E2E blockchain tests on dedicated testnets (e.g., Sepolia, Base Sepolia, Optimism Sepolia). Never on mainnet.

## Related Documentation

-   [Developer Guides: Development Environment Setup](development-environment-setup.md)
-   [Integrator's Guide: Testing](../integrator-guide/testing.md) (general testing strategies)
-   [Integrator's Guide: Sample Code](../integrator-guide/sample-code.md)