# GDS Agent API

A complete, secure API that lets your system talk directly to GDS application — no browser, no manual login required. Create orders, check status, upload documents, and receive real-time notifications, all programmatically.

**Base URL:** `https://api.geodesiq.co/api/v1`  

---

## Table of Contents

- [What Is This?](#what-is-this)
- [How Authentication Works](#how-authentication-works)
- [Configuration Lookups](#configuration-lookups)
- [Clients](#clients)
- [Rates](#rates)
- [Quotes](#quotes)
- [Documents](#documents)
- [Orders](#orders)
- [Webhooks](#webhooks)
- [End-to-End Order Flow](#end-to-end-order-flow)
- [Order Status Lifecycle](#order-status-lifecycle)
- [Webhook Events](#webhook-events)
- [Error Codes](#error-codes)

---

## What Is This?

Normally, agents use the GDS web portal to create remittance orders. The Agent API allows your software system to do everything the portal can do — automatically, without a human clicking buttons.

- **Automate Orders** — Create, monitor, and cancel remittance orders from your own system without logging into the portal.
- **Real-Time Notifications** — Register a webhook URL to receive instant push notifications when an order is paid, delivered, or fails.
- **Secure by Design** — Every request is signed with a secret key. Even if someone intercepts the request, they cannot forge it.

---

## How Authentication Works

Every API call requires **three headers**. Think of it like a tamper-proof envelope: the API key says who you are, the timestamp proves the message is fresh, and the signature proves nobody altered the contents.

| Header | Description |
|---|---|
| `X-Api-Key` | Your identity — starts with `gds_agt_` |
| `X-Timestamp` | Current time (Unix seconds, must be within ±5 min of server time) |
| `X-Signature` | HMAC-SHA256 proof that the request wasn't tampered |

### How the Signature Is Built

Join these four parts with a newline character (`\n`), then sign:

```
METHOD
/api/v1/path
TIMESTAMP
SHA256(body_bytes_as_hex)
```

- `METHOD` — e.g. `POST`, `GET`
- `/api/v1/path` — the URL path (no query string)
- `TIMESTAMP` — same value as `X-Timestamp`
- `SHA256(body)` — lowercase hex hash of the raw request body bytes
  - For `GET`/`DELETE` requests: use `SHA256("")` (hash of empty string)
  - For `multipart/form-data` (document uploads): also use `SHA256("")`
  - For `POST`/`PATCH` with a JSON body: hash the raw body bytes

Then sign the assembled string with your HMAC secret:

```
X-Signature: HMAC-SHA256(hmac_secret, canonical_string_as_hex)
```

The `sha256=` prefix is accepted but optional.

> **HMAC Secret is shown once only.** When you generate an API key from the agent portal (`https://dashboard.geodesiq.co`), both the API key and the HMAC signing secret are displayed in a one-time modal. Save them immediately — the secret cannot be retrieved again.

### What Happens on Each Request

1. **Timestamp check** — Is the request within ±5 minutes of server time? If not → `401 TIMESTAMP_EXPIRED`. This blocks replay attacks.
2. **Key lookup** — The first 8 characters of your API key body (after `gds_agt_`) are used to find your record in the database quickly.
3. **Full key verification** — A SHA-256 hash of your full API key is compared to what's stored. If it doesn't match → `401 KEY_INVALID`.
4. **Signature verification** — The server recomputes the HMAC signature using your stored secret and compares it to `X-Signature`. If different → `401 SIGNATURE_INVALID`.
5. **Revocation check** — Is this key revoked or expired? If yes → `401 KEY_REVOKED`.

---

## Configuration Lookups

These read-only endpoints return the lists you need to fill in an order — which payment methods are available, which purposes are enabled for your account, and which institutions can receive funds.

| Method | Path | What It Returns |
|---|---|---|
| `GET` | `/config/payment-purposes` | List of purposes enabled for your agent account (e.g. Education Fee, Living Expenses). Filter by `?country_code=CN`. |
| `GET` | `/config/institutions` | Universities or institutions that can receive funds. Filter by `?purpose_id=...` to get institutions linked to a specific purpose. |
| `GET` | `/config/payment-methods` | Available payment channels (e.g. WeChat Pay) and their fee structure for your account. Filter by `?country_code=CN`. |
| `GET` | `/ping` | Health check — confirms your credentials are valid and the API is reachable. Returns `{ ok: true, agent_id: "..." }`. |

---

## Clients

A "client" is the person sending the remittance (e.g. a student paying tuition). Clients are identified by their ID document type and number. The same client record is shared across the platform — if two agents have transacted with the same passport, they share the same client record, but **you can only look up clients you have previously transacted with**.

| Method | Path | What It Does |
|---|---|---|
| `GET` | `/clients` | Find a client by their ID document. Required query params: `?id_type=PASSPORT&id_num=A1234567`. Returns `404` if not found or not associated with your account. |
| `GET` | `/clients/{client_id}` | Get full details of a client by their internal UUID (the `client_id` returned from a previous order). Useful for building a "repeat client" UX where you cache the UUID. |

### Client Response Fields

| Field | Description |
|---|---|
| `id` | Internal UUID — cache this for repeat orders |
| `name` | Full name |
| `email` | Email address |
| `phone_country` | Dial code, e.g. `+60` |
| `phone_num` | Phone number without dial code |
| `nationality` | ISO country code, e.g. `CN` |
| `id_type` | Document type, e.g. `PASSPORT` |
| `id_num` | Document number |
| `payment_reference` | Your reference stored on the client — e.g. student ID, invoice number |

---

## Rates

`GET /rates` returns live exchange rates. No reservation is made — this is for display purposes only.

| Query Param | Required | Description |
|---|---|---|
| `req_currency` | No | The base currency (default: `MYR`) |
| `payment_currency` | No | A specific destination currency to calculate a single rate against. If omitted, all available rates vs `req_currency` are returned. |

**Single-rate response** (when `payment_currency` is provided):

```json
{
  "updated_at": "2026-06-06T10:00:00Z",
  "payment_currency": "CNY",
  "req_currency": "MYR",
  "ex_rate": "0.71428571"
}
```

**All-rates response** (no `payment_currency`):

```json
{
  "updated_at": "2026-06-06T10:00:00Z",
  "req_currency": "MYR",
  "rates": {
    "CNY": "0.71428571",
    "USD": "4.72000000"
  }
}
```

---

## Quotes

Before creating an order, use **Quotes** to *lock in* an exchange rate while your client confirms. The lock duration is configurable on the server (default: **10 minutes**).

| Method | Path | What It Does |
|---|---|---|
| `POST` | `/quotes` | Lock in a rate. Returns a `quote_id` that you must include when creating the order. If the quote expires or is already used, the order will be rejected. |
| `GET` | `/quotes/{quote_id}` | Check the status of a quote — whether it has expired or been used, and the locked amounts. |

### POST /quotes — Request Body

| Field | Required | Description |
|---|---|---|
| `payment_method_code` | Yes | e.g. `WECHAT` |
| `payment_purpose_id` | Yes | UUID from `/config/payment-purposes` |
| `req_amount` | Yes | Amount in MYR |
| `req_country` | Yes | Destination country ISO code, e.g. `CN` |
| `institution_id` | No | UUID from `/config/institutions` — required for non-personal purposes |

### Quote Response

```json
{
  "quote_id": "uuid...",
  "payment_method_code": "WECHAT",
  "req_country": "CN",
  "req_amount": "1500.00",
  "req_currency": "MYR",
  "payment_currency": "CNY",
  "ex_rate": "0.71428571",
  "payment_amount": "1082.14",
  "expires_at": "2026-06-06T10:10:00Z",
  "validity_seconds": 600
}
```

> **Why lock a rate?** Exchange rates change constantly. By locking a quote, your client is guaranteed to pay exactly the amount shown at confirmation time. If the quote expires before the client confirms, request a new one.

---

## Documents

Some orders require supporting documents — an Offer Letter (`OF`) and an Invoice (`IV`). You can also upload KYC identity documents (`PP`, `IF`, `IB`) tied to a client.

Upload documents *before* creating the order, then reference them by ID in the order request. This "pre-upload" approach makes order creation fast and atomic.

| Method | Path | What It Does |
|---|---|---|
| `POST` | `/documents` | Upload a file (`multipart/form-data`). Returns a `doc_id` — pass this in `doc_ids` when creating the order. |
| `GET` | `/orders/{order_id}/documents/CB/download` | Download the **Cross-Border Payment Note** (CB) — available once the order is `P` (Paid) or `D` (Delivered). |
| `GET` | `/orders/{order_id}/documents/PC/download` | Download the **Payment Confirmation Receipt** (PC) — available only after the order is `D` (Delivered). |

### POST /documents — Form Fields

| Field | Required | Description |
|---|---|---|
| `file` | Yes | The file to upload. Accepted formats: `jpg`, `jpeg`, `png`, `pdf`. |
| `doc_type` | Yes | One of `PP`, `IF`, `IB`, `OF`, `IV` — see table below. |
| `client_id` | Conditional | Required for KYC document types (`PP`, `IF`, `IB`). Must be a client you have a prior order with. |

### Document Types

| Code | Name | Description | Requires `client_id` |
|---|---|---|---|
| `PP` | Passport | Client's passport scan | Yes |
| `IF` | ID Front | Client's national ID (front) | Yes |
| `IB` | ID Back | Client's national ID (back) | Yes |
| `OF` | Offer Letter | Order form / offer letter for the institution | No |
| `IV` | Invoice | Invoice from the institution | No |

**Response:**

```json
{
  "doc_id": "uuid...",
  "doc_type": "OF"
}
```

> KYC documents (`PP`, `IF`, `IB`) are uploaded to the payment gateway in the background after this call returns. Upload them before creating an order — the order will fail at `422 XC_FILES_NOT_READY` if no client documents have been successfully sent to the gateway.

---

## Orders

| Method | Path | What It Does |
|---|---|---|
| `POST` | `/orders` | **Create a new remittance order.** Returns the new `order_id`, `client_id`, and initial status. |
| `GET` | `/orders` | List all your orders. Filter by `status`, `client_id`, date range. Paginate with `limit` (max 100) and `offset`. |
| `GET` | `/orders/{order_id}` | Full detail of a single order: amounts, exchange rate, client info, documents, payment instructions, and availability of CB/PC documents. |
| `POST` | `/orders/{order_id}/cancel` | Cancel an order. Only possible while status is **N, R, or U**. Returns `409 INVALID_STATE` if the order cannot be cancelled. |

### Creating an Order — Client Identity

When creating an order, identify the client in one of two ways:

**Form A — Repeat Client** (use when you already have the client's UUID from a previous order):

| Field | Required |
|---|---|
| `client_id` | Yes |

**Form B — New / Update Client** (for first-time clients or to update their details; the system upserts by `id_type` + `id_num`):

| Field | Required |
|---|---|
| `id_type` | Yes |
| `id_num` | Yes |
| `client_name` | Yes |
| `client_email` | Yes |
| `client_phone_country` | Yes |
| `client_phone_num` | Yes |
| `nationality` | Yes |

Provide Form A **or** Form B, not both.

### Creating an Order — Always Required Fields

The payment method, country, purpose, institution, and amounts are drawn from the referenced `quote_id`. You do not re-specify them in the order.

| Field | Required | Description |
|---|---|---|
| `quote_id` | Yes | The locked rate — must be unused and not expired |
| `payment_reference` | Yes | Your reference number, e.g. student ID or invoice no. Max 100 chars. Stored on the client record. |
| `payer_relation` | Yes | Relationship between account holder and client, e.g. `Self`, `Parent` |
| `doc_ids` | Yes | Array of pre-uploaded doc IDs — **must include at least one `OF` and one `IV`** |
| `req_desc` | No | Remittance description / purpose text |
| `realpayer_name` | Conditional | Required when `payer_relation` is not `Self` |
| `realpayer_id_type` | Conditional | Required when `payer_relation` is not `Self` |
| `realpayer_id_num` | Conditional | Required when `payer_relation` is not `Self` |

### Order Creation Response

A successful `POST /orders` returns `201` with:

```json
{
  "order_id": "uuid...",
  "client_id": "uuid...",
  "order_no": "abc123456789",
  "status": "U",
  "req_amount": "1500.00",
  "req_currency": "MYR",
  "payment_currency": "CNY",
  "payment_amount": "1082.14",
  "ex_rate": "0.71428571",
  "payment_details": {
    "bank_name": "...",
    "account_name": "...",
    "account_number": "...",
    "reference": "..."
  }
}
```

If document uploads to the payment gateway failed but other documents succeeded, the response includes a `warnings` array with details. If **no** documents reached the gateway, the order is immediately marked `F` and a `422 XC_FILES_NOT_READY` is returned.

---

## Webhooks

Instead of continuously polling `GET /orders/{id}`, register a **webhook** — a URL in your system that GDS system will call automatically whenever an order status changes.

Only one active webhook subscription is allowed per agent at a time. Registering a new one automatically deactivates the previous active subscription.

| Method | Path | What It Does |
|---|---|---|
| `GET` | `/webhooks` | List all webhook subscriptions (URL, events, active status, failure count). |
| `POST` | `/webhooks` | Register a new webhook. Deactivates any existing active subscription first. Returns a `secret` — **shown once only** — used to verify incoming calls. |
| `DELETE` | `/webhooks/{webhook_id}` | Deactivate a webhook subscription. Returns `204 No Content`. The subscription record is retained but marked inactive. |

### POST /webhooks — Request Body

| Field | Required | Description |
|---|---|---|
| `url` | Yes | A valid HTTPS URL (max 500 characters) |
| `events` | Yes | Non-empty array of event names to subscribe to |

**Response (`201`):**

```json
{
  "webhook_id": "uuid...",
  "url": "https://your-system.example.com/webhooks/gds",
  "events": ["order.paid", "order.delivered"],
  "secret": "<base64-encoded 32-byte secret>"
}
```

The `secret` is a base64-encoded 32-byte random value, shown once only. Decode it from base64 to raw bytes when computing the HMAC to verify incoming payloads.

---

## End-to-End Order Flow

1. **Check configuration** — Call `GET /config/payment-purposes`, `/config/institutions`, and `/config/payment-methods` to populate your order form dropdowns. These rarely change — safe to cache.

2. **Upload KYC documents** — Call `POST /documents` with `doc_type=PP` (or `IF`/`IB`) and the client's `client_id` for each identity document. These are pre-uploaded to the payment gateway in the background.

3. **Look up or collect client details** — Call `GET /clients?id_type=PASSPORT&id_num=...` to check if the client exists. If found, use Form A with their `client_id`. If not, collect their details for Form B.

4. **Upload order documents** — Call `POST /documents` for the Offer Letter (`OF`) and Invoice (`IV`). Save the returned `doc_id` values.

5. **Get a rate quote** — Call `POST /quotes` with the payment method, purpose, amount, and destination country to lock in the exchange rate. Show the locked amounts to your client for confirmation.

6. **Create the order** — Call `POST /orders` with the `quote_id`, client identity (Form A or B), `payment_reference`, `payer_relation`, and `doc_ids`. The system atomically consumes the quote, upserts the client record, creates the order, and submits it to the payment gateway. The order arrives in status **U (Underway)** on success.

7. **Monitor for updates** — Poll `GET /orders/{order_id}` or use a registered webhook to receive push notifications when status changes to Paid, Delivered, or Failed.

8. **Download completion documents** — Once status is **Paid** (`P`), download the CB note. Once **Delivered** (`D`), download the PC receipt as well.

---

## Order Status Lifecycle

An order moves through the following statuses. The normal happy path is **N → R → U → P → D**.

```
N (New) → R (Ready) → U (Underway) → P (Paid) → D (Delivered)
                              ↘ F (Failed — terminal)
N / R / U → C (Cancelled — terminal)
```

| Code | Name | What It Means | Can Cancel? | Documents Available |
|---|---|---|---|---|
| `N` | New | Order just created, awaiting gateway submission | Yes | — |
| `R` | Ready | Submitted to payment gateway, waiting for payer | Yes | — |
| `U` | Underway | Payment link active — returned after successful order creation | Yes | — |
| `P` | Paid | Payment confirmed; payout to institution in progress | No | CB note |
| `D` | Delivered | Institution payout completed — fully settled | No | CB note + PC receipt |
| `C` | Cancelled | Order was cancelled | — | — |
| `F` | Failed | Payment failed — no funds collected | — | — |

---

## Webhook Events

| Event Name | When It Fires |
|---|---|
| `order.paid` | Payment confirmed by gateway — funds received |
| `order.delivered` | Institution payout completed |
| `order.failed` | Payment failed |
| `order.cancelled` | Order was cancelled |

### Incoming Payload Structure

```json
{
  "event": "order.paid",
  "timestamp": "2026-06-06T10:05:00.000Z",
  "data": {
    "order_id": "uuid...",
    "order_no": "abc123456789",
    "status": "P",
    "client_name": "张三",
    "req_amount": "1500.00",
    "req_currency": "MYR",
    "payment_amount": "1082.14",
    "payment_currency": "CNY"
  }
}
```

Every incoming webhook request includes an `X-Gds-Signature` header:

```
X-Gds-Signature: sha256=<lowercase_hex>
```

Compute: `HMAC-SHA256(raw_webhook_secret_bytes, raw_payload_bytes)` where `raw_webhook_secret_bytes` is your webhook `secret` decoded from base64. Always verify this header before processing the event.

### Retry Schedule

If your webhook URL returns anything other than a `2xx` response, GDS system will retry up to **5 attempts total**:

| Attempt | Delay |
|---|---|
| 1st (initial) | Immediate |
| 2nd | +1 minute |
| 3rd | +5 minutes |
| 4th | +30 minutes |
| 5th | +2 hours |

After 5 consecutive failures, the subscription is automatically **deactivated**. Retries are abandoned immediately if the subscription is deactivated between attempts.

---

## Error Codes

All errors follow the same shape:

```json
{ "error": "ERROR_CODE", "message": "Human-readable description." }
```

| HTTP | Code | Plain-English Meaning |
|---|---|---|
| 400 | `INVALID_REQUEST` | A required field is missing or has an invalid value |
| 400 | `INVALID_JSON` | Request body is not valid JSON |
| 400 | `INVALID_FILE` | File format not allowed, extension doesn't match content, or content is unreadable |
| 400 | `FILE_TOO_LARGE` | File exceeds the configured maximum size |
| 400 | `AMOUNT_TOO_LOW` | Requested amount is below the minimum for this payment method |
| 400 | `AMOUNT_TOO_HIGH` | Requested amount exceeds the maximum for this payment method |
| 401 | `TIMESTAMP_EXPIRED` | Request timestamp is more than ±5 minutes from server time |
| 401 | `KEY_INVALID` | API key missing, malformed, or hash doesn't match |
| 401 | `SIGNATURE_INVALID` | The HMAC signature doesn't match — request may have been altered |
| 401 | `KEY_REVOKED` | API key has been revoked or has expired |
| 403 | `FORBIDDEN` | The resource exists but belongs to another agent |
| 404 | `NOT_FOUND` | The requested order, client, quote, or webhook was not found |
| 404 | `CURRENCY_NOT_FOUND` | The requested currency code is not in the exchange rate data |
| 409 | `QUOTE_EXPIRED` | The quote has passed its expiry time — create a new one |
| 409 | `QUOTE_USED` | The quote has already been consumed by another order |
| 409 | `CONFLICT` | A document was claimed by a concurrent request — retry with new documents |
| 409 | `INVALID_STATE` | The action isn't allowed in the order's current status (e.g. cancelling a Paid order) |
| 422 | `XC_FILES_NOT_READY` | No documents have been successfully pre-uploaded to the payment gateway |
| 422 | `XC_ORDER_FAILED` | The payment gateway rejected the order — see `message` for details |
| 503 | `RATE_UNAVAILABLE` | Exchange rate data is not available |
| 500 | `INTERNAL_ERROR` | Unexpected server error — contact support |

---

*GDS Agent API — Implementation Overview · v1 · Last updated 2026-06-08 · Full machine-readable spec: `spec/v1.0.0.yaml` (OpenAPI 3.1)*
