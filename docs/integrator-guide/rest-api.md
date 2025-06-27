# Integrator's Guide: REST API

The Gemforce platform exposes much of its functionality through a rich RESTful API, powered by a Parse Server backend. This guide will walk you through how to interact with these endpoints, covering basic requests, common operations, and best practices for consuming the Gemforce REST API.

## Overview

The Gemforce REST API provides:

-   **Data Management**: CRUD (Create, Read, Update, Delete) operations for various data objects (e.g., users, projects, custom classes).
-   **Cloud Function Invocation**: Execute custom server-side logic defined as Cloud Functions.
-   **File Storage**: Upload and retrieve files.
-   **User Management**: Sign up, log in, manage user sessions, and reset passwords.
-   **ACLs and CLPs**: Access Control Lists (ACLs) and Class Level Permissions (CLPs) to secure your data.

## Base URL

All REST API requests should be made to your Gemforce Parse Server instance's `/parse` endpoint.

`YOUR_GEMFORCE_PARSE_SERVER_URL/parse`

## Authentication

Every request to the Parse Server REST API requires specific headers for authentication.

-   `X-Parse-Application-Id`: Your Parse Application ID.
-   `X-Parse-REST-API-Key`: Your REST API Key. For client-side applications, this is generally safe to include. For server-side, you might use your `Master Key` (see section on Authentication).
-   `X-Parse-Master-Key`: (Server-side only) Your Parse Master Key. Grants full access and bypasses security. Use with extreme caution.
-   `X-Parse-Session-Token`: (Optional) Required for requests on behalf of an authenticated user.

For more details checkout the [Authentication Guide](authentication.md).

## Making Requests

You can use any HTTP client (e.g., `curl`, `Postman`, `axios` in JavaScript, `requests` in Python) to interact with the Gemforce REST API.

Let's look at some common operations.

### 1. Invoking Cloud Functions

Cloud Functions allow you to execute server-side logic remotely.

**Endpoint**: `/functions/{functionName}`
**Method**: `POST`
**Body**: JSON object containing parameters for the Cloud Function.

```bash
curl -X POST \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/functions/helloWorld \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
        "message": "Hello from API!"
      }'
```

**Response Example:**

```json
{
  "result": "Hello from API! You sent: Hello from API!"
}
```

### 2. Creating an Object (e.g., a "Project" in a custom class)

You can create new objects in any custom Parse Class.

**Endpoint**: `/classes/{ClassName}`
**Method**: `POST`
**Body**: JSON object representing the object data.

```bash
curl -X POST \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
        "name": "My First Gemforce Project",
        "status": "pending",
        "budget": 100000
      }'
```

**Response Example:**

```json
{
  "objectId": "oZ5jV6QJ59",
  "createdAt": "2025-06-26T14:30:00.000Z"
}
```

### 3. Retrieving Objects

Retrieve objects from a Parse Class. You can use query parameters for filtering, sorting, and pagination.

**Endpoint**: `/classes/{ClassName}` or `/classes/{ClassName}/{objectId}`
**Method**: `GET`
**Query Parameters**:
-   `where`: JSON encoded query constraints (e.g., `{"status": "active"}`).
-   `limit`: Maximum number of results to return.
-   `skip`: Number of results to skip (for pagination).
-   `order`: Field to sort by (e.g., `-createdAt` for descending).
-   `keys`: Comma-separated list of keys to return (for projection).
-   `include`: Comma-separated list of pointer fields to include the full object.

#### Get all Projects

```bash
curl -X GET \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY'
```

#### Get a specific Project by objectId

```bash
curl -X GET \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project/oZ5jV6QJ59 \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY'
```

#### Query Projects with a specific status

```bash
curl -X GET \
  'YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project?where={"status": "pending"}' \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY'
```

**Response Example:**

```json
{
  "results": [
    {
      "name": "My First Gemforce Project",
      "status": "pending",
      "budget": 100000,
      "objectId": "oZ5jV6QJ59",
      "createdAt": "2025-06-26T14:30:00.000Z",
      "updatedAt": "2025-06-26T14:30:00.000Z"
    }
  ]
}
```

### 4. Updating an Object

Modify existing objects in a Parse Class.

**Endpoint**: `/classes/{ClassName}/{objectId}`
**Method**: `PUT`
**Body**: JSON object containing the fields to update.

```bash
curl -X PUT \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project/oZ5jV6QJ59 \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-REST-API-Key: YOUR_PARSE_REST_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
        "status": "approved",
        "approvedBy": "admin_user_id"
      }'
```

**Response Example:**

```json
{
  "updatedAt": "2025-06-26T14:45:00.000Z"
}
```

### 5. Deleting an Object

Remove an object from a Parse Class.

**Endpoint**: `/classes/{ClassName}/{objectId}`
**Method**: `DELETE`

```bash
curl -X DELETE \
  YOUR_GEMFORCE_PARSE_SERVER_URL/parse/classes/Project/oZ5jV6QJ59 \
  -H 'X-Parse-Application-Id: YOUR_PARSE_APP_ID' \
  -H 'X-Parse-Master-Key: YOUR_PARSE_MASTER_KEY' # Often requires Master Key for delete
```

**Response Example:**

```json
{}
```

## Best Practices

-   **Error Handling**: Always check the response status and gracefully handle errors returned by the API. Parse Server returns JSON objects with `code` and `error` fields for failures. (See [Error Handling](error-handling.md) guide)
-   **Session Tokens**: For user-facing applications, manage session tokens securely. Store them in `localStorage` (Web) or `Keychain` (Mobile) and include them in all authenticated requests.
-   **Master Key Security**: Never expose your Master Key in client-side code. Use it only in secure, server-side environments.
-   **Rate Limiting**: Be aware of any rate limits imposed by your Parse Server instance to prevent abuse and ensure fair usage.
-   **Parse SDKs**: For rich client applications (Web, iOS, Android, etc.), consider using the official Parse SDKs (JavaScript, Java, Swift/Objective-C) which simplify API interactions, handle session management, and provide offline capabilities.
-   **Cloud Functions for Complex Logic**: For complex business logic, data transformations, or interactions with external services, prefer creating and invoking Cloud Functions. This keeps your client code cleaner and your business logic secure on the server.
-   **Security**: Implement appropriate Access Control Lists (ACLs) and Class Level Permissions (CLPs) on your Parse Server schema to protect your data.

## Related Documentation

-   [ integrators-guide/authentication.md ](../integrator-guide/authentication.md)
-   [Cloud Functions: Overview](../cloud-functions/index.md)
-   [Parse Server REST API Guide](https://docs.parseplatform.org/rest/guide/) (External)