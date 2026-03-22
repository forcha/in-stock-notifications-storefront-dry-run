# Out-of-Stock Notification — Storefront API Contract

This document defines the complete API contract for the in-stock notifications proxy, hosted on Adobe I/O Runtime. It is intended for frontend developers implementing the storefront UI integration.

---

## Table of Contents

- [Overview](#overview)
- [Base URL](#base-url)
- [Authentication](#authentication)
- [CORS](#cors)
- [Common Response Shapes](#common-response-shapes)
- [Endpoints](#endpoints)
  - [GET /check-status](#get-check-status)
  - [POST /subscribe](#post-subscribe)
  - [POST /unsubscribe](#post-unsubscribe)
- [Error Reference](#error-reference)
- [Edge Cases & Idempotency](#edge-cases--idempotency)
- [Recommended UI Flow](#recommended-ui-flow)
- [Full curl Examples](#full-curl-examples)

---

## Overview

The proxy exposes three public endpoints that the storefront calls directly from the browser. There is no authentication required from the frontend — credentials are stored server-side in the App Builder runtime action.

| Endpoint | Method | Purpose |
|---|---|---|
| `/check-status` | GET | Check whether an email is subscribed to a SKU |
| `/subscribe` | POST | Subscribe an email to back-in-stock alerts for a SKU |
| `/unsubscribe` | POST | Unsubscribe an email from alerts for a SKU |

---

## Base URL

```
https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications
```

All three endpoints live under this base URL. Append the endpoint path to form the full URL, e.g.:

```
https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/check-status
```

---

## Authentication

**No authentication is required from the frontend.** These are public web actions. Do not send any `Authorization` header — it will be ignored.

Credentials to the underlying notifications service are injected server-side and are never exposed to the client.

---

## CORS

All three endpoints support cross-origin requests from any domain.

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, OPTIONS` (check-status) / `POST, OPTIONS` (subscribe, unsubscribe) |
| `Access-Control-Allow-Headers` | `Content-Type` |

**Pre-flight requests (`OPTIONS`)** are handled automatically and return `200 OK` with an empty body. No special handling is required in your fetch client.

---

## Common Response Shapes

All responses return JSON. The top-level `success` boolean indicates whether the operation succeeded.

### Success

```json
{
  "success": true,
  "result": {
    "sku": "ADB153",
    "email": "shopper@example.com",
    "subscribed": true
  }
}
```

### Proxy validation error (missing parameters)

Returned by the proxy layer before reaching the notifications service. HTTP status `400`.

```json
{
  "success": false,
  "error": "missing parameter(s) 'email'"
}
```

### Proxy internal error

Returned when the proxy itself fails (e.g., upstream service unreachable). HTTP status `500`.

```json
{
  "success": false,
  "error": "Failed to subscribe: <reason>"
}
```

### Upstream service error

When the upstream notifications service returns an error, the proxy passes its status code and body through unchanged.

```json
{
  "success": false,
  "error": "Human-readable error message"
}
```

---

## Endpoints

---

### GET /check-status

Check whether a given email address is currently subscribed to back-in-stock alerts for a specific SKU.

Call this when the product page loads to determine whether to show the **"Notify me"** button or the **"You're subscribed"** state.

**URL**

```
GET /check-status?sku={sku}&email={email}
```

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sku` | string | Yes | The product SKU. Will be URL-encoded by the proxy. |
| `email` | string | Yes | The shopper's email address. Will be URL-encoded by the proxy. |

**Request Headers**

| Header | Value |
|---|---|
| _(none required)_ | |

**Success Response — `200 OK`**

Returned for both subscribed and non-subscribed states. An unknown SKU also returns `200` with `subscribed: false` (never a 404).

```json
{
  "success": true,
  "result": {
    "sku": "ADB153",
    "email": "shopper@example.com",
    "subscribed": false
  }
}
```

```json
{
  "success": true,
  "result": {
    "sku": "ADB153",
    "email": "shopper@example.com",
    "subscribed": true
  }
}
```

**Error Responses**

| Status | Condition | Body |
|---|---|---|
| `400` | `sku` or `email` missing from query string | `{ "success": false, "error": "missing parameter(s) 'sku,email'" }` |
| `500` | Proxy internal error (upstream unreachable) | `{ "success": false, "error": "Failed to check subscription status: <reason>" }` |

**Example fetch (JavaScript)**

```javascript
const base = 'https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications';

async function checkSubscriptionStatus(sku, email) {
  const params = new URLSearchParams({ sku, email });
  const res = await fetch(`${base}/check-status?${params}`);
  const data = await res.json();
  return data.result.subscribed; // boolean
}
```

---

### POST /subscribe

Subscribe an email address to back-in-stock notifications for a SKU. A confirmation email is sent to the shopper by the notifications service.

Subscribing the same email + SKU combination more than once is **idempotent** — the second call succeeds silently without creating duplicates or sending a second confirmation email.

**URL**

```
POST /subscribe
```

**Request Headers**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |

**Request Body**

```json
{
  "sku": "ADB153",
  "email": "shopper@example.com"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `sku` | string | Yes | The product SKU |
| `email` | string | Yes | The shopper's email address |

**Success Response — `200 OK`**

```json
{
  "success": true,
  "result": {
    "sku": "ADB153",
    "email": "shopper@example.com",
    "subscribed": true
  }
}
```

**Error Responses**

| Status | Condition | Body |
|---|---|---|
| `400` | `sku` or `email` missing from request body | `{ "success": false, "error": "missing parameter(s) 'email'" }` |
| `500` | Proxy internal error | `{ "success": false, "error": "Failed to subscribe: <reason>" }` |

**Example fetch (JavaScript)**

```javascript
const base = 'https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications';

async function subscribe(sku, email) {
  const res = await fetch(`${base}/subscribe`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sku, email }),
  });
  const data = await res.json();
  if (!data.success) throw new Error(data.error);
  return data.result;
}
```

---

### POST /unsubscribe

Unsubscribe an email address from back-in-stock notifications for a SKU. A confirmation email is sent to the shopper by the notifications service.

Unsubscribing an email that is not currently subscribed is **idempotent** — the call succeeds silently.

**URL**

```
POST /unsubscribe
```

**Request Headers**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |

**Request Body**

```json
{
  "sku": "ADB153",
  "email": "shopper@example.com"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `sku` | string | Yes | The product SKU |
| `email` | string | Yes | The shopper's email address |

**Success Response — `200 OK`**

```json
{
  "success": true,
  "result": {
    "sku": "ADB153",
    "email": "shopper@example.com",
    "subscribed": false
  }
}
```

**Error Responses**

| Status | Condition | Body |
|---|---|---|
| `400` | `sku` or `email` missing from request body | `{ "success": false, "error": "missing parameter(s) 'email'" }` |
| `500` | Proxy internal error | `{ "success": false, "error": "Failed to unsubscribe: <reason>" }` |

**Example fetch (JavaScript)**

```javascript
const base = 'https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications';

async function unsubscribe(sku, email) {
  const res = await fetch(`${base}/unsubscribe`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sku, email }),
  });
  const data = await res.json();
  if (!data.success) throw new Error(data.error);
  return data.result;
}
```

---

## Error Reference

| HTTP Status | Source | Condition | `success` |
|---|---|---|---|
| `200` | Proxy + upstream | All successful operations | `true` |
| `400` | Proxy | `sku` or `email` missing from request | `false` |
| `500` | Proxy | Upstream service unreachable or proxy crashed | `false` |

> **Note:** When the upstream notifications service itself returns an error (e.g., a `400` from a malformed upstream request), the proxy passes the upstream status code and body through unchanged. Always check `data.success` rather than relying solely on the HTTP status code.

---

## Edge Cases & Idempotency

### Unknown SKU returns `subscribed: false`, not `404`

Any SKU string can be queried via `check-status`. If no record exists for it, the API returns `200` with `subscribed: false`. You do not need to handle `404` for this endpoint.

### Subscribe is idempotent

Calling `/subscribe` with the same `sku` + `email` more than once:
- Returns `200` with `subscribed: true`
- Does **not** create duplicate subscription records
- Does **not** send a second confirmation email

### Unsubscribe is idempotent

Calling `/unsubscribe` for an email that is not subscribed:
- Returns `200` with `subscribed: false`
- Does **not** return an error

### Email delivery

A confirmation email is sent to the shopper on both subscribe and unsubscribe. This is handled by the notifications service — no action is required from the frontend.

---

## Recommended UI Flow

```
Product page loads
  │
  ├─ Product is IN STOCK
  │    └─ Show normal "Add to cart" button. No notification UI needed.
  │
  └─ Product is OUT OF STOCK
       │
       ├─ User is NOT logged in / email unknown
       │    └─ Show email input + "Notify me when available" button
       │
       └─ User email is known
            │
            ├─ Call GET /check-status?sku={sku}&email={email}
            │
            ├─ result.subscribed === false
            │    └─ Show "Notify me when available" button
            │         └─ On click → POST /subscribe
            │              ├─ Success → Show "You'll be notified!" confirmation
            │              └─ Error   → Show error message, allow retry
            │
            └─ result.subscribed === true
                 └─ Show "You're subscribed" state with "Unsubscribe" link
                      └─ On click → POST /unsubscribe
                           ├─ Success → Revert to "Notify me" button
                           └─ Error   → Show error message, allow retry
```

### Loading states

- Show a loading/spinner state while any API call is in flight.
- Disable the button during the in-flight request to prevent duplicate submissions.
- On network error (fetch throws), show a generic "Something went wrong, please try again" message.

### Email input validation

Validate the email format client-side before calling the API to avoid unnecessary round trips. A simple regex or the browser's built-in `<input type="email">` validation is sufficient.

---

## Full curl Examples

### Check status (not subscribed)

```bash
curl -s "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/check-status?sku=ADB153&email=shopper@example.com" | jq
```

### Check status (subscribed)

```bash
curl -s "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/check-status?sku=ADB153&email=subscriber@example.com" | jq
```

### Subscribe

```bash
curl -s -X POST \
  "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/subscribe" \
  -H "Content-Type: application/json" \
  -d '{"sku":"ADB153","email":"shopper@example.com"}' | jq
```

### Unsubscribe

```bash
curl -s -X POST \
  "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/unsubscribe" \
  -H "Content-Type: application/json" \
  -d '{"sku":"ADB153","email":"shopper@example.com"}' | jq
```

### Missing parameter — error path

```bash
curl -s -X POST \
  "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/subscribe" \
  -H "Content-Type: application/json" \
  -d '{"sku":"ADB153"}' | jq
```

Expected: `400` with `{ "success": false, "error": "missing parameter(s) 'email'" }`

### CORS pre-flight

```bash
curl -s -X OPTIONS \
  "https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications/subscribe" \
  -H "Origin: https://your-storefront.com" \
  -H "Access-Control-Request-Method: POST" \
  -I
```

Expected: `200 OK` with `Access-Control-Allow-Origin: *`
