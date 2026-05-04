# Razorpay Integration Guide

Complete reference for integrating Razorpay payments into a backend service, derived from the local Razorpay MCP server (`razorpay/mcp` Docker image).

> **Important**: All amounts in Razorpay are in the **smallest currency sub-unit** (paisa for INR). For ₹295, send `29500`. For ₹1, send `100`.

---

## Table of Contents

1. [Authentication](#authentication)
2. [Standard Payment Flow](#standard-payment-flow)
3. [Orders API](#1-orders-api)
4. [Payments API](#2-payments-api)
5. [Payment Links API](#3-payment-links-api)
6. [Refunds API](#4-refunds-api)
7. [QR Codes API](#5-qr-codes-api-upi)
8. [Common Field Reference](#common-field-reference)

---

## Authentication

All API calls use HTTP Basic Auth with your Razorpay Key ID and Key Secret.

```
Authorization: Basic base64(KEY_ID:KEY_SECRET)
```

- **Test keys** start with `rzp_test_...` — sandbox, no real money
- **Live keys** start with `rzp_live_...` — real transactions

---

## Standard Payment Flow

```
[Frontend]                [Backend]                 [Razorpay]
    |                         |                          |
    |---- 1. Initiate ------> |                          |
    |                         |--- 2. Create Order ----->|
    |                         |<--- order_id ------------|
    |<--- 3. order_id --------|                          |
    |                                                    |
    |---- 4. Open Checkout (Razorpay JS) ----------------|
    |                                                    |
    |<--- 5. payment_id + razorpay_signature ------------|
    |                                                    |
    |---- 6. Verify --------> |                          |
    |                         |--- 7. Verify signature ->|
    |                         |--- 8. Fetch payment ---->|
    |                         |<--- payment status ------|
    |<--- 9. Success ---------|                          |
```

**Steps explained:**
1. User clicks "Pay" on frontend
2. Backend calls `Create Order` — returns `order_id`
3. Backend sends `order_id` + amount to frontend
4. Frontend opens Razorpay Checkout JS with `order_id`
5. After payment, Razorpay returns `payment_id`, `order_id`, `razorpay_signature`
6. Frontend posts these to backend
7. Backend verifies HMAC signature: `HMAC_SHA256(order_id + "|" + payment_id, KEY_SECRET)`
8. Backend optionally calls `Fetch Payment` to confirm status
9. Mark order as paid in DB

---

## Detailed Order Flow (with API calls)

Below is the complete end-to-end flow showing exactly which API is called at each step, what payload moves between systems, and the DB state changes.

### Step 1 — User initiates payment (Frontend → Backend)

```
POST /api/payments/create-order
{
  "amount": 50000,
  "currency": "INR",
  "purpose": "subscription_renewal"
}
```

### Step 2 — Backend creates Razorpay order (Backend → Razorpay)

API used: **`POST /v1/orders`** (MCP tool: `create_order`)

```python
# services/razorpay_service.py
import razorpay
from config import settings

client = razorpay.Client(auth=(settings.RAZORPAY_KEY_ID, settings.RAZORPAY_KEY_SECRET))

async def create_order(amount: int, currency: str, user_id: str, purpose: str):
    order = client.order.create({
        "amount": amount,
        "currency": currency,
        "receipt": f"rcpt_{uuid.uuid4().hex[:12]}",
        "notes": {
            "user_id": user_id,
            "purpose": purpose
        }
    })
    return order
```

DB write: insert into `payments` table with status `pending`:

```sql
INSERT INTO payments (payment_id, order_id, user_id, amount, currency, status, created_at)
VALUES (NULL, $1, $2, $3, $4, 'pending', NOW());
```

### Step 3 — Backend returns order_id to frontend

```json
{
  "success": true,
  "data": {
    "order_id": "order_SlIfzOnmocJHFf",
    "key_id": "rzp_test_SlI9mS2kmWEjlL",
    "amount": 50000,
    "currency": "INR"
  }
}
```

> **Note**: Always return `key_id` (NOT `key_secret`) so the frontend can initialize Checkout.

### Step 4 — Frontend opens Razorpay Checkout

```javascript
const options = {
  key: response.data.key_id,
  amount: response.data.amount,
  currency: response.data.currency,
  order_id: response.data.order_id,
  name: "Your Company",
  description: "Subscription Renewal",
  handler: async (rzpResponse) => {
    // rzpResponse contains: razorpay_payment_id, razorpay_order_id, razorpay_signature
    await fetch("/api/payments/verify", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(rzpResponse)
    });
  },
  prefill: { name: user.name, email: user.email, contact: user.phone },
  theme: { color: "#3399cc" }
};
const rzp = new Razorpay(options);
rzp.open();
```

### Step 5 — User pays on Razorpay UI

Razorpay handles card/UPI/netbanking entry, OTP, 3DS auth — all in their hosted iframe. On success, the `handler` callback fires with three values:

- `razorpay_payment_id` — e.g., `pay_IluGXgWmHNoTr8`
- `razorpay_order_id` — same as the order you created
- `razorpay_signature` — HMAC-SHA256 of `order_id|payment_id` signed with your `KEY_SECRET`

### Step 6 — Frontend posts verification to Backend

```
POST /api/payments/verify
{
  "razorpay_payment_id": "pay_IluGXgWmHNoTr8",
  "razorpay_order_id": "order_SlIfzOnmocJHFf",
  "razorpay_signature": "9ef4dffaf81..."
}
```

### Step 7 — Backend verifies signature (CRITICAL — never skip this)

```python
import hmac, hashlib
from config import settings

def verify_payment_signature(order_id: str, payment_id: str, signature: str) -> bool:
    body = f"{order_id}|{payment_id}".encode()
    expected = hmac.new(
        settings.RAZORPAY_KEY_SECRET.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

If signature is invalid → reject the request with `400 BAD REQUEST`. Someone is spoofing a successful payment.

### Step 8 — Backend confirms payment via Razorpay (defense in depth)

API used: **`GET /v1/payments/:id`** (MCP tool: `fetch_payment`)

```python
payment = client.payment.fetch(payment_id)
assert payment["status"] == "captured"
assert payment["order_id"] == order_id
assert payment["amount"] == expected_amount
```

This is your **second line of defense** against signature replay — verifying the actual payment status with Razorpay before granting access.

### Step 9 — Backend updates DB and unlocks the feature

```sql
UPDATE payments
SET payment_id = $1,
    status = 'paid',
    paid_at = NOW()
WHERE order_id = $2;
```

Then trigger downstream actions (send receipt email, activate subscription, etc.) — preferably via a `BackgroundTask`.

---

## Webhook Integration

Webhooks are **the source of truth** for payment events. The Step 6–9 flow above can fail (user closes the tab, network drops, etc.) — webhooks ensure you ALWAYS know the final state.

### Why Webhooks Matter

| Scenario | Without Webhook | With Webhook |
|----------|----------------|--------------|
| User closes tab after paying | DB still shows `pending` | Webhook updates to `paid` |
| Network failure on `/verify` | Order looks failed | Webhook reconciles |
| Async refund processed | No update | Webhook fires `refund.processed` |
| Payment fails after auth | Stale state | Webhook fires `payment.failed` |
| Settlement done | Manual check needed | Webhook fires `settlement.processed` |

### Setting Up a Webhook

1. Go to https://dashboard.razorpay.com/app/webhooks
2. Click **Add New Webhook**
3. Set **Webhook URL**: `https://yourdomain.com/api/payments/webhook`
4. Set **Webhook Secret**: generate a strong random string — save it as `RAZORPAY_WEBHOOK_SECRET` in `.env`
5. Select events to subscribe to (see table below)
6. Save

### Key Webhook Events

| Event | When it fires | Typical action |
|-------|--------------|----------------|
| `payment.authorized` | Payment authorized but not captured | Wait for capture, or capture manually |
| `payment.captured` | Payment captured (money moves to your account) | Mark order as `paid`, send receipt |
| `payment.failed` | Payment failed (user error, bank decline, etc.) | Mark order as `failed`, notify user |
| `order.paid` | Order fully paid (sum of payments = order amount) | Final state for the order |
| `refund.created` | Refund initiated | Mark refund as `pending` |
| `refund.processed` | Refund settled to customer | Mark refund as `completed` |
| `refund.failed` | Refund failed | Notify ops team |
| `payment_link.paid` | Payment link successfully paid | Mark invoice as paid |
| `payment_link.expired` | Payment link expired unpaid | Trigger reminder |
| `subscription.charged` | Recurring charge succeeded | Extend subscription validity |
| `settlement.processed` | Funds settled to bank | Update accounting |

### Webhook Payload Structure

Every webhook follows this shape:

```json
{
  "entity": "event",
  "account_id": "acc_BFQ7uQEaa7j2zG",
  "event": "payment.captured",
  "contains": ["payment"],
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_IluGXgWmHNoTr8",
        "entity": "payment",
        "amount": 50000,
        "currency": "INR",
        "status": "captured",
        "order_id": "order_SlIfzOnmocJHFf",
        "method": "card",
        "captured": true,
        "email": "user@example.com",
        "contact": "+919876543210",
        "fee": 1180,
        "tax": 180,
        "notes": { "user_id": "user_abc123" },
        "created_at": 1700000050
      }
    }
  },
  "created_at": 1700000051
}
```

### Webhook Signature Verification

Razorpay signs every webhook with HMAC-SHA256 using your **webhook secret** (NOT your API key secret). The signature is in the `X-Razorpay-Signature` header.

```python
import hmac, hashlib
from fastapi import Header, Request, HTTPException
from config import settings

def verify_webhook_signature(payload_bytes: bytes, signature: str) -> bool:
    expected = hmac.new(
        settings.RAZORPAY_WEBHOOK_SECRET.encode(),
        payload_bytes,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### Full Webhook Endpoint (FastAPI Pattern)

```python
# routers/webhooks.py
from fastapi import APIRouter, Request, Header, HTTPException, BackgroundTasks
from limiter import limiter
from logger import get_logger
from services.razorpay_service import verify_webhook_signature
from models.payment import Payment

logger = get_logger(__name__)
router = APIRouter(prefix="/api/payments", tags=["Webhooks"])


@router.post("/webhook", status_code=200)
@limiter.limit("100/minute")
async def razorpay_webhook(
    request: Request,
    background_tasks: BackgroundTasks,
    x_razorpay_signature: str = Header(...),
    x_razorpay_event_id: str = Header(None),
):
    """
    Receive and process Razorpay webhook events.

    - Signature verification via X-Razorpay-Signature header
    - Idempotent: dedup by X-Razorpay-Event-Id
    - Always returns 200 if signature is valid (even on processing errors)
    """
    raw_body = await request.body()

    if not verify_webhook_signature(raw_body, x_razorpay_signature):
        logger.warning(f"Invalid webhook signature from {request.client.host}")
        raise HTTPException(status_code=400, detail="Invalid signature")

    payload = await request.json()
    event = payload.get("event")
    event_id = x_razorpay_event_id

    if await is_event_already_processed(event_id):
        logger.info(f"Skipping duplicate webhook: {event_id}")
        return {"status": "duplicate"}

    background_tasks.add_task(process_webhook_event, event, payload, event_id)

    return {"status": "received"}


async def process_webhook_event(event: str, payload: dict, event_id: str):
    """Route the event to its handler."""
    try:
        if event == "payment.captured":
            await handle_payment_captured(payload["payload"]["payment"]["entity"])
        elif event == "payment.failed":
            await handle_payment_failed(payload["payload"]["payment"]["entity"])
        elif event == "order.paid":
            await handle_order_paid(payload["payload"]["order"]["entity"])
        elif event == "refund.processed":
            await handle_refund_processed(payload["payload"]["refund"]["entity"])
        else:
            logger.info(f"Unhandled webhook event: {event}")

        await mark_event_processed(event_id)
    except Exception as e:
        logger.error(f"Webhook processing failed for {event_id}: {e}", exc_info=True)


async def handle_payment_captured(payment: dict):
    """Mark order as paid in DB."""
    await Payment.update_status(
        order_id=payment["order_id"],
        payment_id=payment["id"],
        status="paid",
        amount_paid=payment["amount"]
    )
```

### Critical Webhook Rules

1. **Always return 200 within 5 seconds** — Razorpay retries failed webhooks. Heavy processing must go to a background task.
2. **Verify signature first, before parsing** — protects against malicious replay.
3. **Use raw body bytes** for signature, NOT the parsed JSON — JSON re-serialization changes byte order.
4. **Idempotency is mandatory** — Razorpay may deliver the same event multiple times. Track `X-Razorpay-Event-Id` in a `processed_webhooks` table.
5. **Whitelist Razorpay IPs** at the firewall (optional but recommended).
6. **Never trust the payload alone** — webhooks confirm "an event happened," but for high-value operations, fetch the entity again via API to verify current state.
7. **Use a separate webhook secret** — different from your API key secret. Rotate it periodically.

### Idempotency Table Schema

```sql
CREATE TABLE processed_webhooks (
    event_id VARCHAR(255) PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    received_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP
);

CREATE INDEX idx_processed_webhooks_received_at ON processed_webhooks(received_at);
```

### Order State Machine (with Webhook Events)

```
                       ┌─────────────────┐
   Create Order ──────>│   created       │
                       └────────┬────────┘
                                │ (user pays via Checkout)
                                ▼
                       ┌─────────────────┐
                       │   attempted     │ (payment.authorized fires)
                       └────────┬────────┘
                                │ (auto-capture or manual capture)
                                ▼
                       ┌─────────────────┐
                       │     paid        │ (payment.captured + order.paid fire)
                       └────────┬────────┘
                                │ (optional: refund initiated)
                                ▼
                       ┌─────────────────┐
                       │    refunded     │ (refund.processed fires)
                       └─────────────────┘

   Failure path:
   created → attempted → (payment fails) → payment.failed event
   The order stays in "attempted" status; user can retry payment on the same order.
```

---

## 1. Orders API

The **Order** is the foundation of every Razorpay payment. Create it first, then use the `order_id` in the checkout flow.

### 1.1 Create Order

**MCP Tool**: `create_order`

**Purpose**: Create a new order before initiating payment.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | Yes | Amount in smallest currency unit (paisa). Min: 100 |
| `currency` | string | Yes | ISO 4217 code (e.g., `INR`, `USD`, `SGD`). 3 uppercase letters |
| `receipt` | string | No | Internal receipt number. Max 40 chars, must be unique |
| `notes` | object | No | Key-value metadata. Max 15 pairs, 256 chars each |
| `partial_payment` | boolean | No | Allow partial payments. Default: `false` |
| `first_payment_min_amount` | number | No | Min first partial payment (only if `partial_payment=true`) |
| `transfers` | array | No | Route payment to linked accounts (Razorpay Route) |
| `method` | string | No | Required only for mandate orders. Use `"upi"` |
| `customer_id` | string | No | Required for mandate orders. Format: `cust_xxx` |
| `token` | object | No | Required for mandate orders (recurring) |

**Sample Request (Regular Order)**:
```json
{
  "amount": 50000,
  "currency": "INR",
  "receipt": "order_rcptid_11",
  "notes": {
    "user_id": "user_abc123",
    "purpose": "subscription_renewal"
  }
}
```

**Sample Response**:
```json
{
  "id": "order_IluGWxBm9U8zJ8",
  "entity": "order",
  "amount": 50000,
  "amount_paid": 0,
  "amount_due": 50000,
  "currency": "INR",
  "receipt": "order_rcptid_11",
  "status": "created",
  "attempts": 0,
  "notes": {
    "user_id": "user_abc123",
    "purpose": "subscription_renewal"
  },
  "created_at": 1700000000
}
```

**Order Status Values**: `created` → `attempted` → `paid`

---

### 1.2 Fetch Order

**MCP Tool**: `fetch_order`

**Purpose**: Retrieve order details by ID.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | Yes | Order ID (starts with `order_`) |

**Sample Request**:
```json
{ "order_id": "order_IluGWxBm9U8zJ8" }
```

**Sample Response**: Same shape as `Create Order` response, with updated `status`, `amount_paid`, `attempts`.

---

### 1.3 Fetch Order Payments

**MCP Tool**: `fetch_order_payments`

**Purpose**: List all payment attempts made against an order.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | Yes | Order ID (starts with `order_`) |

**Sample Response**:
```json
{
  "entity": "collection",
  "count": 1,
  "items": [
    {
      "id": "pay_IluGXgWmHNoTr8",
      "entity": "payment",
      "amount": 50000,
      "currency": "INR",
      "status": "captured",
      "order_id": "order_IluGWxBm9U8zJ8",
      "method": "card",
      "captured": true,
      "email": "user@example.com",
      "contact": "+919876543210",
      "fee": 1180,
      "tax": 180,
      "created_at": 1700000050
    }
  ]
}
```

---

### 1.4 Update Order

**MCP Tool**: `update_order`

**Purpose**: Update only the `notes` field of an order.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | Yes | Order ID (starts with `order_`) |
| `notes` | object | Yes | Key-value pairs. Max 15 pairs, 256 chars per value |

---

### 1.5 Fetch All Orders

**MCP Tool**: `fetch_all_orders`

**Purpose**: List orders with filters and pagination.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | number | No | Items per page. Default: 10, Max: 100 |
| `skip` | number | No | Items to skip (offset). Default: 0 |
| `from` | number | No | Unix timestamp — start date filter |
| `to` | number | No | Unix timestamp — end date filter |
| `receipt` | string | No | Filter by receipt value |
| `authorized` | number | No | `0` for unauthorized, `1` for authorized |
| `expand` | array | No | Expand nested data: `payments`, `payments.card`, `transfers`, `virtual_account` |

---

## 2. Payments API

Once a payment is made via Checkout, you'll receive a `payment_id`. Use these APIs to verify, capture, or inspect it.

### 2.1 Fetch Payment

**MCP Tool**: `fetch_payment`

**Purpose**: Get full details of a payment.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID (starts with `pay_`) |

**Sample Response**:
```json
{
  "id": "pay_IluGXgWmHNoTr8",
  "entity": "payment",
  "amount": 50000,
  "currency": "INR",
  "status": "captured",
  "order_id": "order_IluGWxBm9U8zJ8",
  "invoice_id": null,
  "international": false,
  "method": "card",
  "amount_refunded": 0,
  "refund_status": null,
  "captured": true,
  "description": null,
  "card_id": "card_IluGXgnh7kDPVT",
  "bank": null,
  "wallet": null,
  "vpa": null,
  "email": "user@example.com",
  "contact": "+919876543210",
  "notes": {},
  "fee": 1180,
  "tax": 180,
  "error_code": null,
  "error_description": null,
  "created_at": 1700000050
}
```

**Payment Status Values**: `created` → `authorized` → `captured` | `refunded` | `failed`

---

### 2.2 Capture Payment

**MCP Tool**: `capture_payment`

**Purpose**: Capture a previously authorized payment. Only `authorized` payments can be captured.

> **Note**: By default Razorpay auto-captures payments. Manual capture is only used when you set `payment_capture: 0` during order creation.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID (starts with `pay_`) |
| `amount` | number | Yes | Amount to capture in paisa (must equal authorized amount) |
| `currency` | string | Yes | ISO code (e.g., `INR`) |

**Sample Request**:
```json
{
  "payment_id": "pay_IluGXgWmHNoTr8",
  "amount": 50000,
  "currency": "INR"
}
```

**Sample Response**: Same shape as `Fetch Payment`, with `status: "captured"` and `captured: true`.

---

### 2.3 Update Payment

**MCP Tool**: `update_payment`

**Purpose**: Update only the `notes` field of a payment.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID (starts with `pay_`) |
| `notes` | object | Yes | Key-value metadata (string or integer values) |

---

### 2.4 Fetch Payment Card Details

**MCP Tool**: `fetch_payment_card_details`

**Purpose**: Retrieve card metadata (last 4, network, type) for a card payment. Only works for `method: card`.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID (starts with `pay_`) |

**Sample Response**:
```json
{
  "id": "card_IluGXgnh7kDPVT",
  "entity": "card",
  "name": "Vivek V",
  "last4": "1111",
  "network": "Visa",
  "type": "credit",
  "issuer": "HDFC",
  "international": false,
  "emi": false,
  "sub_type": "consumer"
}
```

---

### 2.5 Fetch All Payments

**MCP Tool**: `fetch_all_payments`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | number | No | Items per page. Default: 10, Max: 100 |
| `skip` | number | No | Offset |
| `from` | number | No | Unix timestamp — start filter |
| `to` | number | No | Unix timestamp — end filter |

---

## 3. Payment Links API

Payment Links are shareable URLs your customers can pay directly — useful when you don't have a checkout UI (email/SMS invoicing, WhatsApp, etc.).

### 3.1 Create Payment Link (Standard)

**MCP Tool**: `create_payment_link`

**Purpose**: Generate a generic payment link supporting all methods (card, UPI, netbanking, wallet).

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | Yes | Amount in paisa. Min: 100 |
| `currency` | string | Yes | ISO code (e.g., `INR`) |
| `description` | string | No | Purpose of payment shown to customer |
| `customer_name` | string | No | Customer's name |
| `customer_email` | string | No | Customer's email |
| `customer_contact` | string | No | Customer's phone |
| `notify_email` | boolean | No | Send email notification |
| `notify_sms` | boolean | No | Send SMS notification |
| `reminder_enable` | boolean | No | Auto-remind customer if unpaid |
| `expire_by` | number | No | Unix timestamp — expiry. Default: 6 months |
| `accept_partial` | boolean | No | Allow partial payments. Default: `false` |
| `first_min_partial_amount` | number | No | Min first partial amount. Default: 100 |
| `reference_id` | string | No | Your internal ref. Must be unique. Max 40 chars |
| `callback_url` | string | No | Redirect URL after payment |
| `callback_method` | string | No | Must be `"get"` if `callback_url` is set |
| `notes` | object | No | Metadata. Max 15 pairs, 256 chars each |

**Sample Request**:
```json
{
  "amount": 100000,
  "currency": "INR",
  "description": "Premium subscription",
  "customer_name": "Vivek V",
  "customer_email": "vivek@example.com",
  "customer_contact": "+919876543210",
  "notify_email": true,
  "notify_sms": true,
  "reference_id": "INV-2025-001",
  "callback_url": "https://yoursite.com/payment-success",
  "callback_method": "get"
}
```

**Sample Response**:
```json
{
  "id": "plink_IluGYjKqV5z0nM",
  "entity": "payment_link",
  "amount": 100000,
  "amount_paid": 0,
  "currency": "INR",
  "status": "created",
  "short_url": "https://rzp.io/i/abcd1234",
  "description": "Premium subscription",
  "reference_id": "INV-2025-001",
  "customer": {
    "name": "Vivek V",
    "email": "vivek@example.com",
    "contact": "+919876543210"
  },
  "notify": {
    "sms": true,
    "email": true
  },
  "reminder_enable": false,
  "created_at": 1700000000,
  "expire_by": 1715552000
}
```

**Payment Link Status**: `created` → `partially_paid` → `paid` | `expired` | `cancelled`

---

### 3.2 Create UPI Payment Link

**MCP Tool**: `payment_link_upi_create`

**Purpose**: Generate a UPI-only payment link (INR only). Same fields as standard link.

**Input Payload**: Identical to `create_payment_link`, but `currency` MUST be `INR`.

---

### 3.3 Fetch Payment Link

**MCP Tool**: `fetch_payment_link`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_link_id` | string | Yes | Link ID (starts with `plink_`) |

---

### 3.4 Notify Payment Link

**MCP Tool**: `payment_link_notify`

**Purpose**: Resend the payment link via SMS or email.

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_link_id` | string | Yes | Link ID (starts with `plink_`) |
| `medium` | string | Yes | `"sms"` or `"email"` |

---

## 4. Refunds API

### 4.1 Create Refund

**MCP Tool**: `create_refund`

**Purpose**: Refund a captured payment (full or partial).

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID to refund (starts with `pay_`) |
| `amount` | number | Yes | Refund amount in paisa. Min: 100 |
| `speed` | string | No | `"normal"` (default, T+5 days) or `"optimum"` (instant) |
| `receipt` | string | No | Your internal reference |
| `notes` | object | No | Metadata. Max 15 pairs |

**Sample Request**:
```json
{
  "payment_id": "pay_IluGXgWmHNoTr8",
  "amount": 50000,
  "speed": "optimum",
  "receipt": "refund_rcpt_001",
  "notes": {
    "reason": "customer_requested"
  }
}
```

**Sample Response**:
```json
{
  "id": "rfnd_IluGZkR8Q3xPv6",
  "entity": "refund",
  "amount": 50000,
  "currency": "INR",
  "payment_id": "pay_IluGXgWmHNoTr8",
  "speed_processed": "instant",
  "speed_requested": "optimum",
  "status": "processed",
  "receipt": "refund_rcpt_001",
  "notes": {
    "reason": "customer_requested"
  },
  "created_at": 1700000200
}
```

**Refund Status**: `pending` → `processed` | `failed`

---

### 4.2 Fetch Refund

**MCP Tool**: `fetch_refund`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refund_id` | string | Yes | Refund ID (starts with `rfnd_`) |

---

### 4.3 Fetch Specific Refund for Payment

**MCP Tool**: `fetch_specific_refund_for_payment`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_id` | string | Yes | Payment ID (starts with `pay_`) |
| `refund_id` | string | Yes | Refund ID (starts with `rfnd_`) |

---

## 5. QR Codes API (UPI)

Generate dynamic UPI QR codes for in-store, kiosk, or POS payments.

### 5.1 Create QR Code

**MCP Tool**: `create_qr_code`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"upi_qr"` |
| `usage` | string | Yes | `"single_use"` or `"multiple_use"` |
| `name` | string | No | Display label (e.g., "Counter 1") |
| `description` | string | No | QR description |
| `fixed_amount` | boolean | No | If `true`, only `payment_amount` accepted. Default: `false` |
| `payment_amount` | number | No | Required if `fixed_amount=true`. Amount in paisa |
| `customer_id` | string | No | Link to a customer (`cust_xxx`) |
| `close_by` | number | No | Unix timestamp — auto-close time. Min 2 mins from now |
| `notes` | object | No | Metadata. Max 15 pairs |

**Sample Request**:
```json
{
  "type": "upi_qr",
  "usage": "multiple_use",
  "name": "Storefront Display",
  "fixed_amount": true,
  "payment_amount": 30000,
  "description": "Pay ₹300 for menu item",
  "close_by": 1715552000
}
```

**Sample Response**:
```json
{
  "id": "qr_IluGavM4xQp9R2",
  "entity": "qr_code",
  "created_at": 1700000000,
  "name": "Storefront Display",
  "usage": "multiple_use",
  "type": "upi_qr",
  "image_url": "https://rzp.io/i/qr_image.png",
  "payment_amount": 30000,
  "status": "active",
  "description": "Pay ₹300 for menu item",
  "fixed_amount": true,
  "payments_amount_received": 0,
  "payments_count_received": 0,
  "close_by": 1715552000,
  "close_reason": null
}
```

**QR Status**: `active` → `closed`

---

### 5.2 Fetch QR Code

**MCP Tool**: `fetch_qr_code`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `qr_code_id` | string | Yes | QR ID (starts with `qr_`) |

---

### 5.3 Fetch Payments for QR Code

**MCP Tool**: `fetch_payments_for_qr_code`

**Input Payload**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `qr_code_id` | string | Yes | QR ID (starts with `qr_`) |
| `count` | number | No | Items per page. Default: 10, Max: 100 |
| `skip` | number | No | Offset |
| `from` | number | No | Unix timestamp — start filter |
| `to` | number | No | Unix timestamp — end filter |

---

## Common Field Reference

### ID Prefixes

| Prefix | Entity |
|--------|--------|
| `order_` | Order |
| `pay_` | Payment |
| `rfnd_` | Refund |
| `plink_` | Payment Link |
| `qr_` | QR Code |
| `cust_` | Customer |
| `card_` | Card |
| `acc_` | Linked Account |

### Currency Codes

Use ISO 4217 3-letter codes: `INR`, `USD`, `EUR`, `GBP`, `SGD`, `AED`, etc.

### Amount Conversion

| Display | API Value |
|---------|-----------|
| ₹1.00 | `100` |
| ₹100.00 | `10000` |
| ₹2,950.00 | `295000` |

```python
# Helper
def to_paisa(rupees: float) -> int:
    return int(round(rupees * 100))

def to_rupees(paisa: int) -> float:
    return paisa / 100
```

### Webhook Signature Verification

After receiving a webhook from Razorpay, ALWAYS verify the signature:

```python
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### Payment Verification After Checkout

```python
import hmac
import hashlib

def verify_payment_signature(
    order_id: str,
    payment_id: str,
    signature: str,
    secret: str
) -> bool:
    body = f"{order_id}|{payment_id}"
    expected = hmac.new(
        secret.encode(),
        body.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

---

## Quick Integration Checklist

- [ ] Add `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` to `config.py` (via pydantic-settings)
- [ ] Create `services/razorpay_service.py` for SDK wrapper
- [ ] Add a `payments` table: `id`, `order_id`, `payment_id`, `amount`, `currency`, `status`, `user_id`, `created_at`
- [ ] Endpoint: `POST /api/payments/create-order` → returns `order_id`
- [ ] Endpoint: `POST /api/payments/verify` → verifies signature, marks order paid
- [ ] Endpoint: `POST /api/payments/webhook` → handles `payment.captured`, `payment.failed`, `refund.processed`
- [ ] Set up webhook in Razorpay Dashboard pointing to your `/webhook` endpoint
- [ ] Test with cards: `4111 1111 1111 1111` (success), `5104 0600 0000 0008` (failure)
- [ ] Test UPI ID: `success@razorpay`
- [ ] Switch keys from `rzp_test_*` to `rzp_live_*` only in production env

---

## Resources

- Official docs: https://razorpay.com/docs/api/
- Test cards: https://razorpay.com/docs/payments/payments/test-card-details/
- Webhook events: https://razorpay.com/docs/webhooks/payloads/
- MCP server source: https://github.com/razorpay/razorpay-mcp-server
- MCP server image: `razorpay/mcp` on Docker Hub
