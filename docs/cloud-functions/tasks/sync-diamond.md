# Cloud Functions: Sync Diamond Tasks

This document details the cloud functions responsible for synchronizing data related to Gemforce Diamond contracts between the blockchain and the Parse Server backend. These functions ensure that the off-chain environment accurately reflects the current state of on-chain Diamond deployments and configurations.

## Overview

Sync Diamond tasks enable:

-   **State Reconciliation**: Automatically updating Parse models (e.g., `Diamond`, `Facet`) to match their on-chain counterparts.
-   **Event Processing**: Indexing historical and real-time smart contract events to derive off-chain state.
-   **Configuration Mirroring**: Ensuring that deployed Diamond configurations (facets, function selectors) are available in the backend.

These functions are crucial for maintaining consistency, providing fast lookup capabilities, and building user interfaces that rely on comprehensive Diamond data.

## Key Functions

### 1. `syncAllDiamonds`

Initiates a full synchronization process for all known Gemforce Diamonds, fetching their current configurations and relevant events.

**Function Name**: `syncAllDiamonds`
**Method**: `POST`

**Parameters**:

-   `network` (String, required): The blockchain network to synchronize from.
-   `fromBlock` (Number, optional): The starting block number for event synchronization. If omitted, starts from a predefined historical block or the last synced block.
-   `toBlock` (Number, optional): The ending block number for event synchronization. If omitted, syncs up to the latest block.
-   `reindexAll` (Boolean, optional): If `true`, forces a complete re-indexing of all events and configurations, discarding previous state. Use with caution.

**Returns**:

-   `summary` (Object): A summary of the synchronization process, including number of Diamonds processed, events indexed, and any errors encountered.

**Example Request**:

```json
{
    "functionName": "syncAllDiamonds",
    "parameters": {
        "network": "base-sepolia",
        "reindexAll": false
    }
}
```

**Example Response**:

```json
{
    "result": {
        "diamondsSynced": 15,
        "facetsUpdated": 50,
        "eventsIndexed": 1200,
        "lastSyncedBlock": 12345678,
        "status": "success"
    }
}
```

### 2. `syncDiamondByAddress`

Synchronizes data for a specific Diamond contract.

**Function Name**: `syncDiamondByAddress`
**Method**: `POST`

**Parameters**:

-   `diamondAddress` (String, required): The address of the Diamond contract to synchronize.
-   `network` (String, required): The blockchain network.
-   `fromBlock` (Number, optional): Starting block for events.

**Returns**:

-   `summary` (Object): Summary for the specific Diamond sync.

### 3. `syncDiamondEvents`

Specifically focuses on indexing events for a Diamond contract (e.g., `DiamondCut`, `OwnershipTransferred`).

**Function Name**: `syncDiamondEvents`
**Method**: `POST`

**Parameters**:

-   `diamondAddress` (String, required): Address of the Diamond.
-   `network` (String, required): Blockchain network.
-   `eventType` (String, optional): Specific event type to sync (e.g., "DiamondCut").
-   `fromBlock` (Number, optional): Starting block for events.

**Returns**: `eventsIndexed` (Number)

## Error Handling

Sync Diamond tasks can face errors due to:

-   **Blockchain Connectivity**: Problems connecting to the RPC node, timeouts during large data fetches.
-   **Invalid Addresses/Networks**: Non-existent Diamond addresses or unsupported networks.
-   **Data Consistency Issues**: Discrepancies between on-chain data and expected Parse schema.
-   **Gas Limits/Timeouts**: Long-running sync processes hitting Cloud Function execution limits.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Access Control**: Synchronization functions are typically privileged. Limit access to trusted backend services or administrators that have a clear purpose for triggering these syncs.
-   **Data Integrity**: Implement robust validation on received blockchain data to prevent corruption of the off-chain database.
-   **Resource Consumption**: Be mindful of the computational resources (CPU, memory, database writes) required for large-scale synchronization, especially for `reindexAll`. Implement rate limiting or schedule these functions during off-peak hours.
-   **RPC Provider Security**: Ensure your RPC provider connection (e.g., API keys) is secure.

## Related Documentation

-   [Smart Contracts: Diamond Contract](../../smart-contracts/diamond.md)
-   [Smart Contracts: Diamond Cut Facet](../../smart-contracts/facets/diamond-cut-facet.md)
-   [Integrator's Guide: Integration Patterns](../../integrator-guide/integration-patterns.md) (especially event-driven architecture)