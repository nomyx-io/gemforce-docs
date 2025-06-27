# Cloud Functions: Admin Utility Tasks

This document details the cloud functions designed for administrative utilities within the Gemforce platform. These functions provide privileged access to perform maintenance, configuration, and data management tasks, primarily intended for platform administrators and operators.

## Overview

Admin Utility tasks enable:

-   **Data Management**: Bulk operations on the Parse Server database (e.g., cleaning up old data, modifying user roles).
-   **Configuration Updates**: Changing platform-wide settings.
-   **System Health Checks**: Performing diagnostics and integrity checks.
-   **User Management**: Advanced user operations not available through standard APIs (e.g., forced password resets, account locking).
-   **Smart Contract Configuration**: Updating certain smart contract settings (e.g., fee percentages, trusted addresses) via backend-signed transactions.

These functions are critical for platform maintenance and operational control.

## Key Functions

### 1. `cleanDatabase`

Performs cleanup operations on specified Parse collections, primarily for test environments or old data.

**Function Name**: `cleanDatabase`
**Method**: `POST`

**Parameters**:

-   `collections` (Array of Strings, required): Names of Parse collections to clean (e.g., ["_User", "Project", "Transactions"]).
-   `query` (Object, optional): A Parse query object (JSON string) to specify which records to delete. If omitted, all records in the specified collections are deleted.
-   `dryRun` (Boolean, optional): If `true`, the function will only report what *would* be deleted without performing the deletion. Defaults to `false`.

**Returns**:

-   `deletedCount` (Number): The number of records deleted (or to be deleted in dry run).
-   `message` (String): A summary of the operation.

**Example Request**:

```json
{
    "functionName": "cleanDatabase",
    "parameters": {
        "collections": ["Project", "TradeDeal"],
        "query": { "status": "completed", "createdAt": { "$lt": { "__type": "Date", "iso": "2024-01-01T00:00:00.000Z" } } },
        "dryRun": false
    }
}
```

### 2. `updatePlatformSetting`

Updates a global platform setting stored in a Parse configuration class.

**Function Name**: `updatePlatformSetting`
**Method**: `POST`

**Parameters**:

-   `settingKey` (String, required): The key of the setting to update (e.g., "marketplaceFeePercentage", "maxTradeDealDuration").
-   `settingValue` (Any, required): The new value for the setting.

**Returns**:

-   `success` (Boolean): `true` if the update was successful.

### 3. `syncBlockchainData`

Triggers a synchronization process for specific blockchain data, such as refreshing cached smart contract states or re-indexing events. (Note: A more detailed version might be `sync-events`).

**Function Name**: `syncBlockchainData`
**Method**: `POST`

**Parameters**:

-   `dataType` (String, required): The type of data to sync (e.g., "marketplaceListings", "userWalletBalances").
-   `network` (String, optional): The blockchain network to sync from.
-   `options` (Object, optional): Sync-specific options (e.g., `fromBlock`, `toBlock`).

**Returns**:

-   `syncSummary` (Object): Details about the synchronization, including number of items synced.

### 4. `forceUserLogout`

Forces a user's session to expire, effectively logging them out from all devices.

**Function Name**: `forceUserLogout`
**Method**: `POST`

**Parameters**:

-   `userId` (String, required): The objectId of the Parse User to log out.

**Returns**: `success` (Boolean)

## Error Handling

Admin Utility tasks require robust error handling:

-   **Permission Denied**: The most common error. Only users with the Parse Master Key or specific `admin` roles should be able to call these functions.
-   **Invalid Input**: Incorrect collection names, malformed queries, or invalid setting values.
-   **Operational Errors**: Failures during database operations or smart contract interactions.

Refer to the [Integrator's Guide: Error Handling](../../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Extreme Privilege**: These functions provide significant control over the platform. **They should be guarded extremely carefully.**
-   **Master Key Usage**: Access to these functions MUST require the Parse Master Key or a highly restricted administrative role with strict ACLs/CLPs.
-   **Logging and Auditing**: Every execution of these functions must be thoroughly logged, including who called it, when, and with what parameters. Implement alerts for unauthorized attempts.
-   **Rate Limiting**: Apply aggressive rate limiting to prevent abuse.
-   **Least Privilege Principle**: When developing these functions, ensure they only have the necessary permissions to perform their specific task and nothing more.
-   **Test Environment**: Never test these functions on production environments. Use dedicated staging or test environments.

## Related Documentation

-   [Integrator's Guide: Authentication](../../integrator-guide/authentication.md)
-   [Parse Server Cloud Code Guide](https://docs.parseplatform.org/cloudcode/guide/) (External)