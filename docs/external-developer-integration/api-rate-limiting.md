# External Developer Integration: API Rate Limiting and Quotas

To ensure fair usage, prevent abuse, and maintain platform stability, Gemforce implements API rate limiting and usage quotas for external developer integrations. This guide explains these mechanisms and provides best practices for designing your applications to conform to them gracefully.

## Overview of Rate Limiting and Quotas

-   **Rate Limiting**: Restricts the number of API requests your application can make within a defined time window (e.g., requests per second, requests per minute). Requests exceeding the limit are typically throttled or rejected.
-   **Usage Quotas**: Define the maximum total number of API calls or resource consumption (e.g., data transfer, storage) allowed over a longer period (e.g., per day, per month).

These measures are crucial for protecting Gemforce's infrastructure and ensuring a high quality of service for all users.

## Understanding Gemforce's Rate Limits

While specific rate limit values might vary based on your integration tier or current platform load, here are common patterns:

-   **Authentication Endpoints**: May have stricter limits to prevent brute-force attacks.
-   **Read Operations (GET)**: Generally have higher limits.
-   **Write Operations (POST, PUT, DELETE)**: Typically have lower limits due to higher resource consumption.
-   **Burst Limits**: Allow for short bursts of higher request volume, but sustained high rates will be throttled.

### Expected HTTP Response Headers

When you hit a rate limit, the Gemforce REST API (Parse Server) will typically return a `429 Too Many Requests` HTTP status code. Look for these headers in the response:

-   `Retry-After`: (Recommended) Indicates how long to wait before making a new request (in seconds).
-   `X-RateLimit-Limit`: The total number of requests allowed in the current time window.
-   `X-RateLimit-Remaining`: The number of requests remaining in the current time window.
-   `X-RateLimit-Reset`: The timestamp (Unix epoch seconds) when the current rate limit window resets.

## Best Practices for Handling Rate Limits

Designing your application with rate limits in mind is crucial for a robust integration.

### 1. Implement Retry Mechanisms with Exponential Backoff

When you receive a `429 Too Many Requests` response, do not immediately retry the request. Implement a strategy that waits before retrying.

-   **Respect `Retry-After` Header**: If the `Retry-After` header is present, always honor it. This is the most direct way to handle the situation.
-   **Exponential Backoff**: If `Retry-After` is not present, use an exponential backoff algorithm. This means waiting a progressively longer period after each failed retry attempt.
    -   Example: Wait 1 second, then 2, then 4, then 8, etc., with some randomized jitter to prevent all clients from retrying simultaneously.
-   **Maximum Retries**: Define a maximum number of retry attempts and fail permanently after that.

#### Example (Conceptual JavaScript)

```javascript
async function callGemforceAPIWithRetry(url, options, retries = 3, delay = 1000) {
    try {
        const response = await fetch(url, options);

        if (response.status === 429) {
            if (retries > 0) {
                const retryAfter = response.headers.get('Retry-After');
                const waitTime = retryAfter ? parseInt(retryAfter, 10) * 1000 : delay + Math.random() * 500; // Add jitter
                console.warn(`Rate limit hit. Retrying in ${waitTime / 1000} seconds. Retries left: ${retries - 1}`);
                await new Promise(resolve => setTimeout(resolve, waitTime));
                return callGemforceAPIWithRetry(url, options, retries - 1, delay * 2); // Double delay for exponential backoff
            } else {
                throw new Error("Maximum retries reached for rate limit.");
            }
        }

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(`API Error: ${errorData.error || response.statusText}`);
        }

        return response.json();
    } catch (error) {
        console.error("API call failed:", error);
        throw error;
    }
}

// Example usage:
// callGemforceAPIWithRetry(
//     "YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project",
//     {
//         method: 'POST',
//         headers: {
//             'X-Parse-Application-Id': 'YOUR_APP_ID',
//             'X-Parse-REST-API-Key': 'YOUR_API_KEY',
//             'Content-Type': 'application/json'
//         },
//         body: JSON.stringify({ name: "Rate Limit Test Project" })
//     }
// );
```

### 2. Implement Client-Side Caching

Cache API responses on your client (frontend or backend) where appropriate to reduce the number of requests to Gemforce.

-   Cache non-volatile data for a period (e.g., configuration settings, static asset URLs).
-   Use HTTP caching headers (`Cache-Control`, `ETag`) when possible.

### 3. Batch Requests

Where the Gemforce API supports it, batch multiple operations into a single request.

-   Parse Server supports batch operations (e.g., `POST /batch`). This significantly reduces the number of HTTP requests and is more efficient than individual calls.

### 4. Monitor Your Usage

Actively monitor your API usage to anticipate hitting quotas or rate limits.

-   Review logs for `429` errors.
-   If Gemforce provides a dashboard or metrics endpoint for API usage, monitor it regularly.

### 5. Optimize Your API Calls

-   **Minimize Data Transfer**: Request only the data you need (using `keys` parameter for GET requests).
-   **Efficient Queries**: Design Parse queries to be as specific as possible to reduce server load.
-   **Webhooks over Polling**: For real-time updates, use [Webhooks](../sdk-libraries/webhooks.md) instead of repeatedly polling APIs.

### 6. Consider Your Integration Tier

Understand if your needs align with any specific integration tiers offered by Gemforce. Higher tiers might offer increased rate limits or dedicated resources.

## Handling Quotas

Unlike rate limits (which are time-windowed), quotas represent an overall usage cap.

-   **Alerting**: Set up internal alerts to notify you when you are approaching your usage quotas. This allows you to adjust your application's behavior or contact Gemforce support for an increase.
-   **Cost Management**: Monitor your consumption against metered usage if applicable.

## Related Documentation

-   [Integrator's Guide: REST API](../integrator-guide/rest-api.md)
-   [Integrator's Guide: Webhooks](../integrator-guide/webhooks.md)
-   [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)