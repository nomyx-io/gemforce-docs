# Database Schema Overview

The Gemforce platform leverages a flexible database architecture, primarily built around MongoDB for Parse Server and potentially PostgreSQL for analytical or specialized data storage. This document provides an overview of the key data models and their relationships, offering insights into how data is structured and managed within the Gemforce ecosystem.

## 1. Primary Database: MongoDB (Parse Server)

Parse Server utilizes MongoDB as its default database, providing a schema-less JSON document model that offers flexibility and scalability. While MongoDB is schema-less by nature, Parse Server enforces a soft schema, where `Classes` are analogous to tables and `Objects` are analogous to rows, with defined types for each `Key` (column).

### 1.1. Key Classes and Their Relationships

Below are some of the critical Parse Server Classes and their typical relationships:

*   **`_User` (Users)**:
    *   Represents platform users with fields like `username`, `email`, `password`, `sessionToken`, and custom fields for user profiles.
    *   **Relationships**:
        *   One-to-One with `IdentityProfile` (via `identityProfilePointer` or similar).
        *   One-to-Many with `Parse.Session` (via `sessionToken`).
        *   Many-to-Many with `Role` (via `_Role` class).

*   **`_Role` (Roles)**:
    *   Defines user roles and permissions within the Parse Server system.
    *   **Relationships**: Many-to-Many with `_User`.

*   **`IdentityProfile`**:
    *   Stores detailed profile information for users' on-chain identities, linked to an `_User` account.
    *   Fields may include `identityAddress` (ERC-735 identity contract address), `walletAddress`, `privateDataPointer` (ACL-protected to owner), `publicProfileData`.
    *   **Relationships**: One-to-One with `_User`.

*   **`SmartContractTransaction`**:
    *   Logs critical on-chain transactions initiated or observed by the platform, including details like `transactionHash`, `blockNumber`, `contractAddress`, `methodName`, `parameters`, `status`.
    *   **Relationships**: Potentially links to `_User` (if initiated by a user).

*   **`WebhookEvent`**:
    *   Stores incoming or outgoing webhook events, used for off-chain notifications and integrations.
    *   Fields may include `eventType`, `payload`, `status`, `targetUrl`, `retryCount`.

*   **`CarbonCredit`**:
    *   Represents carbon credit NFTs, with fields like `tokenId`, `issuer`, `amount`, `vintage`, `state` (e.g., `issued`, `retired`), `metadataURI`, `smartContractAddress`.
    *   **Relationships**: One-to-One with the actual ERC-721A NFT on-chain.

*   **`TradeDeal`**:
    *   Represents the off-chain metadata and state of a trade deal, mirroring or complementing on-chain data.
    *   Fields may include `tradeDealId` (bytes32 from chain), `borrowerUserPointer`, `lenderUserPointer`, `status`, `invoiceNFTId`, `collateralAmount`, `loanAmount`, `repaymentAmount`, `settlementTime`, `gracePeriod`.
    *   **Relationships**: One-to-One with `TradeDeal` on-chain. Links to `_User` for borrower/lender.

*   **`AuditLog`**:
    *   Records significant operational and security events, including `timestamp`, `actorUserPointer`, `actionType`, `targetObjectPointer`, `details`.
    *   **Relationships**: Links to `_User` (for the actor) and other relevant `Parse.Objects`.

*   **`SystemConfig`**:
    *   Stores dynamic platform configurations that are managed via the database rather than static code (e.g., feature flags, global settings).

### 1.2. Parse Server ACLs and Pointers

*   **Access Control Lists (ACLs)**: Parse Server's security model heavily relies on ACLs to control read and write access to `Parse.Objects` down to the field level. Each object can have ACLs restricting access to specific `_User`s or `_Role`s.
*   **Pointers**: Relationships between `Parse.Objects` are typically established using `Pointers`, which store a reference to another object's `objectId` and `className`.

## 2. Potential Secondary Database: PostgreSQL

While MongoDB handles most of Parse Server's data, PostgreSQL (or other relational databases) might be used for specific purposes requiring strong ACID compliance, complex relational queries, or integration with traditional BI tools.

### 2.1. Potential Use Cases

*   **Analytical Data**: De-normalized or aggregated data for business intelligence and reporting.
*   **Financial Ledgers**: For high-integrity financial transaction records that benefit from relational structure and ACID properties.
*   **Complex Relations**: Scenarios where a highly normalized relational model is genuinely advantageous over MongoDB's document model.

### 2.2. Example Tables (if used with PostgreSQL)

*   **`transactions_history`**:
    *   `id`, `transaction_hash`, `from_address`, `to_address`, `amount`, `token_address`, `timestamp`, `status`.
    *   **Relationships**: Foreign key to `users` table (if user-related).
*   **`user_balances`**:
    *   `user_id`, `token_address`, `balance`, `last_updated`.

## 3. Data Relationships and Constraints

*   **One-to-One**: E.g., `_User` to `IdentityProfile`. Enforced by application logic or unique constraints on pointer fields.
*   **One-to-Many**: E.g., a `_User` can have many `SmartContractTransaction` records. Enforced by storing a pointer to the `_User` in the `SmartContractTransaction` class.
*   **Many-to-Many**: E.g., `_User` to `_Role` through the `_Role` default class.
*   **Data Integrity**: Achieved through a combination of:
    *   **Parse Server Hooks**: `beforeSave`, `afterSave`, `beforeDelete`, etc., can implement server-side validation and business logic.
    *   **Client-side Validation**: Initial validation in front-end or mobile applications.
    *   **Cloud Function Logic**: Orchestrating complex relationships and ensuring consistency.
    *   **On-chain Data Mirroring**: For blockchain data, the Parse Server classes often mirror and index on-chain state, benefiting from the blockchain's own integrity mechanisms.

## 4. Schema Evolution

*   **MongoDB's Flexibility**: The schema-less nature of MongoDB makes it easier to evolve the database schema compared to relational databases, as new fields can be added to documents without requiring an immediate database migration.
*   **Parse Server Migrations**: For significant changes (e.g., renaming fields, changing data types for existing data), Parse Server provides tools and best practices for running data migrations.
*   **Versioning**: Application logic should be designed to be backward compatible with older data schemas during migration periods, if necessary.

Understanding these database schemas and their management principles is crucial for developers working on data-driven features and ensuring the long-term integrity and scalability of the Gemforce platform.