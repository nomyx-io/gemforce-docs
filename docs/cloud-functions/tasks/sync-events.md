# Cloud Functions: Sync Events Tasks

This document details the cloud functions dedicated to synchronizing blockchain events with the Parse Server backend. These functions are crucial for building responsive applications that react to on-chain activities like token transfers, NFT mints, marketplace listings, and trade deal state changes, by storing event data in a queryable off-chain database.

## Overview

Sync Events tasks enable:

-   **Event Ingestion**: Reading events from blockchain nodes and writing them to Parse collections.
-   **Historical Sync**: Backfilling events from past blocks.
-   **Real-time Processing**: Monitoring new blocks for live event indexing.
-   **Data Consistency**: Ensuring that the off-chain representation of events matches the on-chain source of truth.

These functions are fundamental for any application feature that requires querying or displaying historical blockchain data or reacting to new on-chain actions.

## Key Functions

### 1. `syncEvent`

Synchronizes events of a specific type from a blockchain contract.

**Function Name**: `syncEvent`
**Method**: `POST`

**Parameters**:

-   `contractAddress` (String, required): The address of the smart contract emitting the events.
-   `eventName` (String, required): The name of the event to synchronize (e.g., "Transfer", "ItemListed", "TradeDealCreated").
-   `network` (String, required): The blockchain network where the contract is deployed.
-   `abi` (Object, required): The ABI of the contract, specifically containing the event definition.
-   `fromBlock` (Number, optional): The starting block number for the synchronization. If omitted, it syncs from the last recorded block for this event or a predefined default.
-   `toBlock` (Number, optional): The ending block number. If omitted, syncs up to the latest block.
-   `chunkSize` (Number, optional): The number of blocks to process in each batch to avoid RPC timeouts or memory limits. Defaults to a reasonable value.
-   `reindex` (Boolean, optional): If `true`, forces a full re-indexing of the event, deleting existing records before starting. Use with caution.
-   `queryFilters` (Object, optional): An object containing indexed event parameters to filter events (e.g., `{ from: "0xSenderAddress" }`).

**Returns**:

-   `eventsIndexed` (Number): The total number of events indexed during the process.
-   `lastBlockProcessed` (Number): The last block number successfully processed.

**Example Request**:

```json
{
    "functionName": "syncEvent",
    "parameters": {
        "contractAddress": "0xYourNFTContractAddress",
        "eventName": "Transfer",
        "network": "base-sepolia",
        "abi": {
            "anonymous": false,
            "inputs": [
                { "indexed": true, "internalType": "address", "name": "from", "type": "address" },
                { "indexed": true, "internalType": "address", "name": "to", "type": "address" },
                { "indexed": false, "internalType": "uint256", "name": "tokenId", "type": "uint256" }
            ],
            "name": "Transfer",
            "type": "event"
        },
        "fromBlock": 12345000,
        "queryFilters": {
            "to": "0xYourAddress"
        }
    }
}
```

**Example Response**:

```json
{
    "result": {
        "eventsIndexed": 50,
        "lastBlockProcessed": 12345678
    }
}
```

**Workflow**:

1.  Client application (backend service) calls `syncEvent` Cloud Function.
2.  Cloud Function connects to the specified blockchain network.
3.  Queries the contract for events within the given block range and filters.
4.  Processes each event, parses its data, and saves it to a designated Parse collection (e.g., `BlockchainEvent`).
5.  Updates the last processed block to ensure continuity for the next sync.

### 2. `syncAllRelevantEvents`

A higher-level function that triggers the synchronization of a predefined set of critical Gemforce events across various core contracts.

**Function Name**: `syncAllRelevantEvents`
**Method**: `POST`

**Parameters**:

-   `network` (String, required): The blockchain network to synchronize events from.
-   `fromBlock` (Number, optional): Global starting block.
-   `toBlock` (Number, optional): Global ending block.

**Returns**: `summary` (Object with details of all synced events)

## Error Handling

Sync Events tasks can encounter errors such as:

-   **RPC Node Issues**: Connectivity problems, rate limits, or outmoded RPC nodes.
-   **Invalid ABI/Contract Address**: The provided ABI does not match the contract, or the address is incorrect.
-   **Event Parsing Errors**: Issues decoding event data due to ABI mismatch or corrupted logs.
-   **Database Write Errors**: Problems saving processed events to Parse.
-   **Timeout**: Long-running syncs exceeding Cloud Function execution limits, especially for large block ranges.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Access Control**: These functions often require privileged access (`Master Key`) if they perform large-scale data writes to the Parse database. This access should be strictly controlled.
-   **Data Integrity**: Implement robust validation on ingested event data to prevent corruption or malicious injection into the off-chain database.
-   **Resource Management**: Be mindful of the computational and network resources consumed by large-scale event synchronization. Implement rate limiting and smart chunking.
-   **RPC Endpoint Security**: Ensure that the RPC endpoints used for fetching events are secure and trusted.

## Related Documentation

-   [Smart Contracts: Overview](../../smart-contracts/index.md)
-   [Integrator's Guide: Integration Patterns](../../integrator-guide/integration-patterns.md) (specifically event-driven architecture)
-   [Parse Server Cloud Code Guide]([https://docs.parseplatform.org/cloudcode/guide/](https://docs.parseplatform.org/cloudcode/guide/)) (External)