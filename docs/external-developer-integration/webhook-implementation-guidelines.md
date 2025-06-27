# External Developer Integration: Webhook Implementation Guidelines

This guide provides detailed best practices and considerations for external developers implementing and consuming webhooks from the Gemforce platform. Effectively handling webhooks is crucial for real-time data synchronization and building responsive applications that react to on-platform events.

## Overview

Gemforce webhooks are HTTP POST requests sent to a URL you configure, triggered by events occurring within the Gemforce ecosystem (e.g., new NFT mints, trade deal status changes, user profile updates). Proper implementation ensures reliability, security, and efficiency in processing these real-time notifications.

## 1. Webhook Endpoint Requirements & Design

Your webhook endpoint is a critical component for consuming events.

### Requirements:

-   **Public Accessibility**: Your endpoint must be publicly accessible from the internet, as Gemforce's servers will initiate the POST request.
-   **HTTPS Only**: Always use HTTPS for your webhook URL to encrypt data in transit and ensure confidentiality. Non-HTTPS endpoints may be rejected by Gemforce or pose a security risk.
-   **HTTP POST Method**: The endpoint must be configured to receive HTTP POST requests.
-   **JSON Payload Processing**: The request body will contain a JSON object with event details. Your endpoint must be able to parse this JSON.
-   **Fast Response Time**: Your endpoint should respond with an HTTP `2xx` status code (e.g., `200 OK`, `204 No Content`) **within a few seconds** (typically < 3-5 seconds). If your endpoint takes too long, Gemforce might consider the delivery failed and retry, leading to duplicate processing.

### Design Principles:

-   **Thin Endpoint, Asynchronous Processing**:
    -   **Recommendation**: Keep your webhook endpoint lean and fast. Its primary responsibility should be to validate the request (signature), acknowledge receipt with a `200 OK`, and then hand off the actual processing of the event payload to an asynchronous background job or message queue.
    -   **Why**: This prevents timeouts, allows for retry mechanisms, and ensures your main application logic doesn't block the webhook receiver.
    -   **Tools**: Message queues (e.g., RabbitMQ, Kafka, AWS SQS), background job processors (e.g., Celery for Python, Node.js worker threads).
-   **Idempotency**: Design your event processing logic to be idempotent. This means that processing the same webhook payload multiple times will produce the same result as processing it once. Webhooks can be delivered multiple times due to network issues, retries, or misconfigurations.
    -   **Strategy**: Use a unique identifier within the webhook payload (e.g., `event.id`, `transactionHash`, `objectId`) to track processed events and prevent re-processing. Store this ID in your database and check against it before performing state-changing operations.

## 2. Security Best Practices for Webhooks

Protecting your webhook endpoint from unauthorized access and ensuring data integrity is paramount.

### 2.1 Signature Verification (HMAC)

-   **Implement**: Always verify the webhook signature. Gemforce will sign the webhook payload using a shared secret key that only you and Gemforce know. This allows you to confirm that the request truly came from Gemforce and that the payload has not been tampered with in transit.
-   **Mechanism**: Gemforce will likely send an `HMAC-SHA256` signature in a custom HTTP header (e.g., `X-Gemforce-Signature`). Your application regenerates the signature using the payload and your secret, then compares it.
-   **Secret Management**:
    -   Store your `WEBHOOK_SECRET` securely (environment variables, secrets manager). **Never hardcode it or expose it publicly.**
    -   Use a unique secret for each webhook or integration.

#### Example (Node.js)

```javascript
/* Relevant snippet from webhook receiver */
const crypto = require('crypto');

const WEBHOOK_SECRET = process.env.GEMFORCE_WEBHOOK_SECRET; // Loaded securely

app.post('/gemforce-webhook', (req, res) => {
    const signature = req.headers['x-gemforce-signature']; 

    if (!signature) {
        return res.status(401).send('Signature header missing');
    }

    try {
        const hmac = crypto.createHmac('sha256', WEBHOOK_SECRET);
        hmac.update(JSON.stringify(req.body)); // Important: payload must be stringified exactly as sent
        const calculatedSignature = hmac.digest('hex');

        if (signature !== calculatedSignature) {
            console.warn('Webhook signature mismatch! Potential tampering.');
            return res.status(403).send('Invalid signature');
        }
        // Signature verified, proceed with processing
        console.log('Webhook signature verified.');
        res.status(200).send('Webhook received');
    } catch (error) {
        console.error('Error during signature verification:', error);
        res.status(500).send('Internal Server Error');
    }
});
```

### 2.2 IP Whitelisting (Optional but Recommended)

-   **Implement**: If Gemforce provides a list of static IP addresses from which its webhooks originate, configure your firewall or load balancer to only accept traffic from these IPs on your webhook endpoint. This adds another layer of defense against unauthorized requests.

### 2.3 Other Security Measures:

-   **Input Validation**: Even after signature verification, always validate the structure and content of the incoming JSON payload to prevent malicious data or unexpected formats from causing issues in your application.
-   **Role-Based Access**: If your endpoint performs privileged actions, ensure it uses a minimal set of permissions.
-   **Error Logging**: Log all webhook reception, verification, and processing errors for auditing and debugging.

## 3. Reliability and Monitoring

Ensuring your webhook consumer is always available and processing events correctly.

### Best Practices:

-   **High Availability**: Deploy your webhook endpoint in a highly available manner (e.g., multiple instances behind a load balancer).
-   **Automated Retries**: Implement retry mechanisms for any internal processes that fail after the webhook has been acknowledged (e.g., if saving to a database fails). Use exponential backoff.
-   **Dead-Letter Queues (DLQs)**: For asynchronous processing, configure a DLQ for messages that repeatedly fail processing. This prevents poison pills from blocking your queues and allows manual inspection.
-   **Monitoring and Alerting**:
    -   **Endpoint Availability**: Monitor your webhook endpoint's uptime and response times.
    -   **Processing Lag**: Monitor the lag between when a webhook is received and when it is fully processed. High lag indicates a bottleneck.
    -   **Error Rates**: Set up alerts for an increase in webhook processing errors.
-   **Logging**: Implement comprehensive logging for all stages of webhook reception and processing.

## Related Documentation

-   [Integrator's Guide: Webhooks](../integrator-guide/webhooks.md) (general overview)
-   [Integrator's Guide: Security](../integrator-guide/security.md)
-   [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)