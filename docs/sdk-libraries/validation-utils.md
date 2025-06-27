# SDK & Libraries: Validation Utilities

This document describes the `validation-utils.ts` library, a key utility module within the Gemforce SDK providing a collection of helper functions for data validation. These utilities are essential for ensuring the integrity and correctness of inputs across various layers of your application, from user interfaces to backend logic before interacting with smart contracts or APIs.

## Overview

The `validation-utils.ts` library offers:

-   **Input Data Validation**: Functions to validate common data types and formats (e.g., addresses, amounts, dates, strings).
-   **Error Reporting**: Consistent methods for reporting validation failures.
-   **Blockchain-Specific Checks**: Utilities for validating parameters relevant to blockchain interactions (e.g., valid ERC-20 amounts, non-zero addresses).

This library helps improve application robustness, prevent common integration errors, and enhance security by ensuring that only valid data is processed.

## Key Functions

### 1. `isValidAddress(address: string)`

Checks if a given string is a valid Ethereum-like blockchain address (e.g., 0x-prefixed, 40 hex characters).

**Function Signature**:
`isValidAddress(address: string): boolean`

**Parameters**:

-   `address` (String, required): The string to validate as an address.

**Returns**:
-   `boolean`: `true` if the address is valid, `false` otherwise.

**Example Usage**:

```typescript
import { isValidAddress } from '@gemforce-sdk/validation-utils'; // Assuming this import path

const address1 = "0x742d35Cc6634C0539Ff34fB5BdCfd2C300f72868"; // Valid
const address2 = "0xinvalidAddress"; // Invalid

console.log(`'${address1}' is valid: ${isValidAddress(address1)}`); // true
console.log(`'${address2}' is valid: ${isValidAddress(address2)}`); // false
```

### 2. `isValidNonZeroAmount(amount: ethers.BigNumberish)`

Checks if a given amount (e.g., `ethers.BigNumber`, string, number) is valid and non-zero. Useful for preventing transactions with zero value.

**Function Signature**:
`isValidNonZeroAmount(amount: ethers.BigNumberish): boolean`

**Parameters**:

-   `amount` (ethers.BigNumberish, required): The amount to validate.

**Returns**:
-   `boolean`: `true` if the amount is valid and greater than zero, `false` otherwise.

### 3. `isPositiveNumber(value: number | string)`

Checks if a given value is a positive number (greater than zero).

**Function Signature**:
`isPositiveNumber(value: number | string): boolean`

**Parameters**:

-   `value` (Number | String, required): The value to validate.

**Returns**:
-   `boolean`: `true` if the value is a positive number, `false` otherwise.

### 4. `isHexString(value: string, length?: number)`

Checks if a given string is a hexadecimal string, optionally verifying its length.

**Function Signature**:
`isHexString(value: string, length?: number): boolean`

**Parameters**:

-   `value` (String, required): The string to validate as hex.
-   `length` (Number, optional): The expected length of the hex string (excluding "0x" prefix), e.g., 64 for a `bytes32`.

**Returns**:
-   `boolean`: `true` if the value is a valid hex string of the specified length (if provided), `false` otherwise.

### 5. `validateEmail(email: string)`

Performs a basic validation check on an email string using a regular expression.

**Function Signature**:
`validateEmail(email: string): boolean`

**Parameters**:

-   `email` (String, required): The email string to validate.

**Returns**:
-   `boolean`: `true` if the email format is valid, `false` otherwise.

### 6. `throwIfInvalid(condition: boolean, errorMessage: string)`

A general helper function to throw an error if a condition is `false`. Useful for consolidating validation logic.

**Function Signature**:
`throwIfInvalid(condition: boolean, errorMessage: string): void`

**Parameters**:

-   `condition` (Boolean, required): The condition to check. If `false`, an error is thrown.
-   `errorMessage` (String, required): The error message to throw.

**Example Usage**:

```typescript
import { isValidAddress, throwIfInvalid } from '@gemforce-sdk/validation-utils';

function processUserRequest(userAddress: string, amount: ethers.BigNumberish) {
    throwIfInvalid(isValidAddress(userAddress), "Invalid recipient address provided.");
    throwIfInvalid(amount > 0, "Amount must be greater than zero.");
    // Further processing...
}

// Example: processUserRequest("0xValidAddress", 100);
// Example: processUserRequest("invalid", 0); // Throws error
```

## Error Handling

Validation utilities primarily throw errors when validation fails. These errors should be caught by the calling application layer and translated into user-friendly messages.

Refer to the [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md) for general error handling strategies.

## Security Considerations

-   **Client-Side vs. Server-Side**: While client-side validation using this library enhances user experience, **always perform server-side validation** as well. Client-side checks can be bypassed.
-   **Canonical Checks**: For critical blockchain-related parameters (like addresses), ensure that the library performs canonical checks (e.g., checksumming for addresses) if applicable.
-   **Regular Expressions**: Be cautious with overly complex regular expressions for validation, as they can sometimes be vulnerable to ReDoS (Regular expression Denial of Service) attacks.

## Related Documentation

-   [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)