# Developer Guides: Development Environment Setup

Setting up a robust and efficient development environment is the first step towards building on the Gemforce platform. This guide provides comprehensive instructions and recommendations for configuring your local machine and tools to ensure a smooth development experience, from smart contracts to cloud functions and client applications.

## Overview

A typical Gemforce development environment will involve:

-   **Operating System**: macOS, Linux (Ubuntu, Debian), or Windows (with WSL2).
-   **Node.js & npm/Yarn**: For JavaScript/TypeScript projects (Cloud Functions, SDK, client apps).
-   **Docker & Docker Compose**: For containerized backend services (Parse Server, MongoDB).
-   **Solidity Development Environment**: Tools like Hardhat or Foundry for smart contract development.
-   **Code Editor**: VS Code with recommended extensions.
-   **Terminal**: A robust terminal emulator (e.g., iTerm2, Windows Terminal, Zsh/Oh My Zsh).

## 1. Operating System Setup

### macOS / Linux

-   **Homebrew (macOS)** or **apt/dnf/yum (Linux)**: Package managers are essential.
-   **Git**: For version control. Install: `git --version`
-   **Node.js**: Use a version manager like `nvm` (Node Version Manager) to easily switch Node.js versions.
    -   Install `nvm`: Follow instructions on [nvm-sh/nvm](https://github.com/nvm-sh/nvm#installing-and-updating)
    -   Install Node.js (recommended LTS version): `nvm install --lts`
    -   Set default: `nvm use --lts`
    -   Install `yarn` (optional, but recommended): `npm install -g yarn`

### Windows (recommended: WSL2)

-   **Install WSL2 (Windows Subsystem for Linux 2)**: This provides a Linux environment directly within Windows, offering better compatibility and performance for development tools. Follow Microsoft's official guide.
-   **Install a Linux Distribution**: Ubuntu is a common choice.
-   Proceed with Linux setup steps within your WSL2 distribution.

## 2. Docker & Docker Compose

Gemforce's backend services (Parse Server, MongoDB) are designed to run efficiently in Docker containers, simplifying setup and ensuring consistent environments.

-   **Install Docker Desktop**: Download and install from [Docker's Official Website](https://www.docker.com/products/docker-desktop).
-   **Verify Installation**:
    ```bash
    docker --version
    docker compose version
    ```
-   **Ensure Docker is Running**: Docker Desktop application should be running in your system tray.

## 3. Solidity Development Environment (Hardhat)

Hardhat is the recommended environment for developing, compiling, testing, and deploying Gemforce smart contracts.

-   **Install Hardhat**: If you haven't already, install Hardhat globally or within your project.
    ```bash
    npm install --save-dev hardhat
    # or globally: npm install -g hardhat
    ```
-   **Project Setup**: If starting a new smart contract project:
    ```bash
    npx hardhat init
    ```
-   **Hardhat Configuration**: Gemforce smart contract projects will have a `hardhat.config.ts` file configured with specific networks (Sepolia, Base Sepolia, etc.), Solidity compilers, and network providers.
    -   **RPC URLs & Private Keys**: Obtain RPC URLs for testnets (e.g., from Alchemy, Infura, BlastAPI). Manage private keys for deployment accounts securely (use `.env` files and `dotenv`, or a secrets manager).
-   **Recommended Hardhat Plugins**:
    -   `@nomiclabs/hardhat-ethers`: For Ethers.js integration.
    -   `hardhat-gas-reporter`: For gas usage analysis.
    -   `solidity-coverage`: For smart contract code coverage.

## 4. Code Editor (VS Code)

Visual Studio Code is a popular and highly extensible editor suitable for all aspects of Gemforce development.

-   **Download VS Code**: From [Visual Studio Code Official Website](https://code.visualstudio.com/).
-   **Recommended Extensions**:
    -   **Solidity**: For syntax highlighting, linting, and formatting of Solidity files.
    -   **ESLint**: For JavaScript/TypeScript linting.
    -   **Prettier - Code formatter**: For consistent code formatting.
    -   **Docker**: For Dockerfile and Docker Compose syntax highlighting.
    -   **DotENV**: For `.env` file syntax highlighting.
    -   **GitLens**: Enhances Git capabilities within VS Code.
    -   **YAML**: For `mkdocs.yml` and other YAML file editing.

## 5. Terminal Setup

A good terminal experience improves productivity.

-   **macOS**: iTerm2 with Zsh and Oh My Zsh.
-   **Linux**: Your distribution's default terminal with Zsh/Oh My Zsh or Bash.
-   **Windows**: Windows Terminal (use with WSL2) and PowerShell.
-   **Powerline Fonts**: Install a Powerline-compatible font for better display of terminal themes (e.g., Meslo Nerd Font).
-   **Basic Terminal Commands**: Familiarize yourself with `cd`, `ls`, `mkdir`, `rm`, `mv`, `cp`, `grep`.

## 6. Project-Specific Setup (Gemforce Repository)

Once your base environment is ready, clone the Gemforce repository and install project-specific dependencies.

```bash
# Clone the repository
git clone https://github.com/gemforce/gemforce-platform.git
cd gemforce-platform

# Install root dependencies (Node.js/npm)
npm install # or yarn install

# Navigate to specific sub-projects if they have their own package.json
# cd cloud-functions && npm install
# cd some-frontend-app && npm install
# cd smart-contracts && npm install
```
### Environment Variables
-   Gemforce projects will typically use `.env` files to manage environment-specific variables (e.g., API keys, RPC URLs, database connection strings).
-   **DO NOT** commit `.env` files to version control. They should be in your `.gitignore`.
-   **Example `.env` structure**:
    ```dotenv
    # Network RPCs
    SEPOLIA_RPC_URL="https://sepolia.infura.io/v3/YOUR_INFURA_KEY"
    BASE_SEPOLIA_RPC_URL="https://base-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY"

    # Wallet Private Keys (for deployment/admin accounts)
    DEPLOYER_PRIVATE_KEY="0x..." # Highly sensitive, treat with care!

    # Parse Server Configuration
    PARSE_APP_ID="myappid"
    PARSE_MASTER_KEY="mymasterkey"
    PARSE_SERVER_URL="http://localhost:1337/parse" # Or your deployed URL

    # DFNS Integration
    DFNS_API_URL="https://api.dfns.io"
    DFNS_APP_ID="app_xxxx"
    DFNS_SIGNING_KEY_ID="sk_yyyy"
    DFNS_PRIVATE_KEY_PEM="/path/to/dfns-private-key.pem" # Recommended for production
    ```

## 7. PostgreSQL Database for Historical Data

While MongoDB is used by Parse Server, for historical and analytical data, PostgreSQL is also used.

-   **Install PostgreSQL**: Installation varies by OS. On macOS, Homebrew (`brew install postgresql`). On Linux, system package manager (`sudo apt install postgresql`).
-   **Configure**: Ensure PostgreSQL is running and you have a user/database configured for development.
-   **Clients**: Consider using a GUI client like TablePlus, DBeaver, or pgAdmin for database management.

## Troubleshooting Common Issues

-   **Node.js Version Mismatch**: Use `nvm use` to switch to the correct Node.js version specified in the project.
-   **Docker Issues**: Ensure Docker Desktop is running. Check Docker logs for errors (`docker logs <container_name>`).
-   **Network Connectivity**: Verify your internet connection and that RPC URLs are correct and accessible.
-   **Invalid Private Key**: Double-check private keys in your `.env` file; ensure they are 0x-prefixed for `ethers.js`.
-   **Missing Dependencies**: Run `npm install` (or `yarn install`) in the root and any sub-project directories.

By following this guide, you should have a well-equipped development environment ready to tackle Gemforce integrations.

## Related Documentation

-   [Integrator's Guide: Smart Contracts](../integrator-guide/smart-contracts.md)
-   [Integrator's Guide: REST API](../integrator-guide/rest-api.md)
-   [Integrator's Guide: Authentication](../integrator-guide/authentication.md)