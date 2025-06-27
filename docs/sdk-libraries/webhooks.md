# SDK & Libraries: Webhook Utilities

This document describes the `webhooks.ts` library, a utility module within the Gemforce SDK designed to assist with handling and verifying incoming webhooks from the Gemforce platform. It provides functions to parse webhook payloads, verify their authenticity (via signatures), and standardize webhook processing within your application.

## Overview

The `webhooks.ts` library offers:

-   **Payload Parsing**: Standardized parsing of incoming JSON webhook payloads.
-   **Signature Verification**: Crucial security checks to ensure webhook authenticity using shared secrets (HMAC).
-   **Event Type Identification**: Helpers to easily identify the type of event contained within the webhook.

This library is essential for any application that consumes webhooks from the Gemforce platform, guaranteeing data integrity and security.

## Key Functions

### 1. `verifyWebhookSignature(payload: any, signature: string, secret: string)`

Verifies the HMAC signature of an incoming webhook payload against a shared secret.

**Function Signature**:
`verifyWebhookSignature(payload: any, signature: string, secret: string): boolean`

**Parameters**:

-   `payload` (Any, required): The raw JSON payload object received from the webhook.
-   `signature` (String, required): The signature string, typically found in a custom HTTP header (e.g., `X-Gemforce-Signature`).
-   `secret` (String, required): The shared secret key configured both in your application and on the Gemforce platform for this webhook.

**Returns**:
-   `boolean`: `true` if the signature is valid, `false` otherwise.

**Example Usage**:

```typescript
import { verifyWebhookSignature } from '@gemforce-sdk/webhooks'; // Assuming this import path
import express from 'express';
import bodyParser from 'body-parser';
import crypto from 'crypto';

const app = express();
const WEBHOOK_SECRET = process.env.GEMFORCE_WEBHOOK_SECRET || "your_super_secret_webhook_key";

app.use(bodyParser.json());

app.post('/gemforce-webhook-listener', (req, res) => {
    const signature = req.headers['x-gemforce-signature'] as string; // Adjust header name as per Gemforce config

    if (!signature) {
        return res.status(401).send('Signature missing');
    }

    try {
        const isVerified = verifyWebhookSignature(req.body, signature, WEBHOOK_SECRET);
        
        if (!isVerified) {
            console.warn('Webhook signature mismatch. Potential tampering.');
            return res.status(403).send('Invalid signature');
        }

        console.log('Webhook signature verified successfully.');
        // Process webhook payload
        const eventType = req.body.event?.type;
        const eventData = req.body.data;
        console.log(`Received event: ${eventType}`, eventData);

        res.status(200).send('Webhook processed');
    } catch (error) {
        console.error('Error verifying or processing webhook:', error);
        res.status(500).send('Internal server error'); // Respond with 500 for internal errors
    }
});

// app.listen(3000, () => console.log('Webhook listener running on port 3000'));
```

### 2. `getWebhookEventType(payload: any)`

Extracts and returns the type of event from a given webhook payload.

**Function Signature**:
`getWebhookEventType(payload: any): string | null`

**Parameters**:

-   `payload` (Any, required): The JSON payload object received from the webhook.

**Returns**:
-   `string | null`: The event type string (e.g., "user.created", "marketplace.itemListed") or `null` if the type cannot be determined.

### 3. `getWebhookEventData(payload: any)`

Extracts and returns the data object associated with the webhook event from a given payload.

**Function Signature**:
`getWebhookEventData(payload: any): any | null`

**Parameters**:

-   `payload` (Any, required): The JSON payload object received from the webhook.

**Returns**:
-   `any | null`: The event data object or `null` if the data cannot be extracted.

## Error Handling

Webhook utilities primarily throw errors if essential parameters are missing or if signature verification fails. These errors indicate a potential security issue or a malformed webhook, and should be handled by your application to log the incident and potentially alert administrators.

## Security Considerations

-   **Secret Management**: The `secret` used for `verifyWebhookSignature` MUST be kept absolutely secret and should be stored in environment variables or a secure secrets management system (e.g., AWS Secrets Manager, HashiCorp Vault), never hardcoded.
-   **Replay Attacks**: While HMAC signatures prevent tampering, they don't inherently prevent replay attacks. Consider adding a timestamp or a unique nonce to your webhook payloads and verify these on your server to prevent an attacker from re-sending old webhooks.
-   **Idempotency**: Your webhook processing logic should be **idempotent**, meaning that processing the same webhook multiple times has the same effect as processing it once. This is crucial for resilience against retries and accidental duplicate deliveries.
-   **SSL/TLS**: Ensure your webhook endpoint is served over HTTPS to protect the payload in transit.
-   **Input Validation**: Even after signature verification, always validate the contents of the `payload` to protect against logic flaws or unexpected data structures.
-   **Response Time**: Aim for very fast response times (within a few seconds) for your webhook endpoint. Long response times can cause the sender to retry, leading to duplicate deliveries. Process heavy logic asynchronously.

## Related Documentation

-   [Integrator's Guide: Webhooks](../integrator-guide/webhooks.md)
-   [Integrator's Guide: Security](../integrator-guide/security.md)