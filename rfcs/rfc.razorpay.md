# RFC: Razorpay Payment Integration — Agentic Commerce Protocol

**Status:** Draft  
**Version:** 2025-09-29  
**Scope:** Razorpay order creation and payment processing for ChatGPT-driven checkout flows

This RFC defines the integration of **Razorpay's payment APIs** with the Agentic Commerce Protocol. It specifies how merchants can leverage Razorpay's order and payment infrastructure while maintaining compatibility with agent-driven checkout sessions.

---

## 1. Scope & Goals

- Enable **seamless integration** of Razorpay payment processing within the Agentic Checkout flow.
- Provide **order creation** capabilities that link to checkout sessions.
- Support **multiple payment methods** including cards, UPI, netbanking, wallets, and EMI.
- Maintain **merchant control** over payment authorization, capture, and settlement.
- Ensure **compatibility** with the Agentic Commerce Protocol version `2025-09-29`.

**Out of scope:** Refunds, subscriptions, payment links, QR codes, international payments (non-INR), settlement schedules.

### 1.1 Normative Language

The key words **MUST**, **MUST NOT**, **SHOULD**, **MAY** follow RFC 2119/8174.

---

## 2. Protocol Integration Overview

### 2.1 Agentic Checkout + Razorpay Flow

The typical flow integrates Razorpay with the Agentic Checkout API:

1. **Agent initiates checkout** → `POST /checkout_sessions` (Agentic Checkout API)
2. **Merchant creates Razorpay order** → `POST /v1/orders` (Razorpay API)
3. **Agent collects payment details** → User provides card/UPI information
4. **Merchant creates payment** → `POST /v1/payments` (Razorpay API)
5. **Merchant captures payment** → `POST /v1/payments/{id}/capture` (Razorpay API)
6. **Agent completes checkout** → `POST /checkout_sessions/{id}/complete` (Agentic Checkout API)

### 2.2 Version Compatibility

- **API Version:** `2025-09-29` (Agentic Commerce Protocol)
- **Razorpay API:** v1 (stable)
- Clients **SHOULD** send `API-Version: 2025-09-29` header for protocol compliance.

### 2.3 Authentication

Razorpay uses **HTTP Basic Authentication**:
- Format: `Authorization: Basic <base64(key_id:key_secret)>`
- Clients **MUST** obtain API keys from Razorpay Dashboard.
- Keys **MUST** be kept secure and **MUST NOT** be exposed client-side.

---

## 3. HTTP Interface

### 3.1 Common Requirements

All Razorpay endpoints **MUST**:
- Use HTTPS (TLS 1.2+)
- Accept and return `application/json`
- Use **integer amounts in currency subunits** (e.g., paise for INR)
- Support idempotency via `Idempotency-Key` header (RECOMMENDED)

**Request Headers:**
- `Authorization: Basic <credentials>` (**REQUIRED**)
- `Content-Type: application/json` (**REQUIRED**)
- `Accept-Language: <locale>` (OPTIONAL, e.g., `en-US`)
- `User-Agent: <string>` (OPTIONAL)
- `Idempotency-Key: <string>` (RECOMMENDED)
- `Request-Id: <string>` (RECOMMENDED for tracing)
- `API-Version: 2025-09-29` (RECOMMENDED for protocol compliance)

**Response Headers:**
- `Request-Id: <string>` — echo correlation ID when provided

---

## 4. Endpoints

### 4.1 Create Order

**Endpoint:** `POST /v1/orders`

**Purpose:** Creates an order in Razorpay. An order is a prerequisite for payment creation and serves as a linkage between the checkout session and payment processing.

**Request Body:**

| Field             | Type                 | Req | Description                                                    |
| ----------------- | -------------------- | :-: | -------------------------------------------------------------- |
| `amount`          | integer              | ✅  | Amount in currency subunits (e.g., 50000 = ₹500.00)           |
| `currency`        | string               | ✅  | ISO 4217 currency code, uppercase (e.g., `"INR"`)             |
| `receipt`         | string               | ❌  | Receipt number for reference (max 40 characters)               |
| `notes`           | object               | ❌  | Key-value pairs for metadata (max 15 pairs)                    |
| `partial_payment` | boolean              | ❌  | Enable partial payments (default: `false`)                     |

**Notes Field Integration:**
The `notes` object **SHOULD** include:
- `checkout_session_id`: Links to Agentic Checkout session
- `merchant_id`: Merchant identifier
- `customer_email`: Customer email for reference
- Additional merchant-specific metadata

**Response (201 Created):**

| Field          | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| `id`           | string  | Unique order identifier (e.g., `order_...`)  |
| `entity`       | string  | Always `"order"`                             |
| `amount`       | integer | Order amount in subunits                     |
| `amount_paid`  | integer | Amount paid against order                    |
| `amount_due`   | integer | Remaining amount to be paid                  |
| `currency`     | string  | Currency code                                |
| `receipt`      | string  | Receipt number                               |
| `status`       | string  | Order status: `created`, `attempted`, `paid` |
| `attempts`     | integer | Number of payment attempts                   |
| `notes`        | object  | Metadata key-value pairs                     |
| `created_at`   | integer | Unix timestamp                               |

**Example Request:**

```json
{
  "amount": 50000,
  "currency": "INR",
  "receipt": "receipt#123",
  "notes": {
    "checkout_session_id": "csn_01HV3P3...",
    "merchant_id": "acme",
    "customer_email": "john@example.com"
  }
}
```

**Example Response:**

```json
{
  "id": "order_RB58MiP5SPFYyM",
  "entity": "order",
  "amount": 50000,
  "amount_paid": 0,
  "amount_due": 50000,
  "currency": "INR",
  "receipt": "receipt#123",
  "status": "created",
  "attempts": 0,
  "notes": {
    "checkout_session_id": "csn_01HV3P3...",
    "merchant_id": "acme"
  },
  "created_at": 1756455561
}
```

### 4.2 Create Payment

**Endpoint:** `POST /v1/payments`

**Purpose:** Initiates a payment for a given order using various payment methods.

**Request Body:**

| Field         | Type         | Req | Description                                               |
| ------------- | ------------ | :-: | --------------------------------------------------------- |
| `amount`      | integer      | ✅  | Payment amount in subunits                                |
| `currency`    | string       | ✅  | ISO 4217 currency code (uppercase)                        |
| `order_id`    | string       | ✅  | Order ID from create order response                       |
| `email`       | string       | ✅  | Customer email address                                    |
| `contact`     | string       | ✅  | Customer phone with country code (e.g., `+919876543210`) |
| `method`      | string       | ✅  | Payment method: `card`, `upi`, `netbanking`, `wallet`     |
| `card`        | CardDetails  | ❌  | Card details (required if method is `card`)               |
| `upi`         | UpiDetails   | ❌  | UPI details (required if method is `upi`)                 |
| `description` | string       | ❌  | Payment description (max 255 characters)                  |
| `notes`       | object       | ❌  | Key-value pairs for metadata                              |

**CardDetails Object:**

| Field          | Type   | Req | Description                        |
| -------------- | ------ | :-: | ---------------------------------- |
| `number`       | string | ✅  | Card number (13-19 digits)         |
| `name`         | string | ✅  | Cardholder name                    |
| `expiry_month` | string | ✅  | Expiry month (MM format: `"01"`-`"12"`) |
| `expiry_year`  | string | ✅  | Expiry year (YYYY format)          |
| `cvv`          | string | ✅  | Card CVV (3-4 digits)              |

**UpiDetails Object:**

| Field  | Type   | Req | Description                          |
| ------ | ------ | :-: | ------------------------------------ |
| `flow` | string | ✅  | UPI flow: `collect` or `intent`      |
| `vpa`  | string | ❌  | Virtual Payment Address (for collect) |

**Response (200 OK):**

| Field               | Type     | Description                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `id`                | string   | Payment identifier (e.g., `pay_...`)             |
| `entity`            | string   | Always `"payment"`                               |
| `amount`            | integer  | Payment amount in subunits                       |
| `currency`          | string   | Currency code                                    |
| `status`            | string   | Payment status: `created`, `authorized`, `captured`, `failed` |
| `order_id`          | string   | Associated order ID                              |
| `method`            | string   | Payment method used                              |
| `captured`          | boolean  | Whether payment has been captured                |
| `card`              | CardInfo | Card information (if applicable)                 |
| `email`             | string   | Customer email                                   |
| `contact`           | string   | Customer phone                                   |
| `notes`             | object   | Metadata                                         |
| `error_code`        | string   | Error code (if failed)                           |
| `error_description` | string   | Error description (if failed)                    |
| `created_at`        | integer  | Unix timestamp                                   |

**Example Request (Card Payment):**

```json
{
  "amount": 50000,
  "currency": "INR",
  "order_id": "order_RB58MiP5SPFYyM",
  "email": "john@example.com",
  "contact": "+919876543210",
  "method": "card",
  "card": {
    "number": "4111111111111111",
    "name": "John Doe",
    "expiry_month": "12",
    "expiry_year": "2025",
    "cvv": "123"
  },
  "notes": {
    "checkout_session_id": "csn_01HV3P3..."
  }
}
```

**Example Response:**

```json
{
  "id": "pay_29QQoUBi66xm2f",
  "entity": "payment",
  "amount": 50000,
  "currency": "INR",
  "status": "authorized",
  "order_id": "order_RB58MiP5SPFYyM",
  "method": "card",
  "captured": false,
  "card": {
    "id": "card_29QQoUBi66xm2g",
    "entity": "card",
    "name": "John Doe",
    "last4": "1111",
    "network": "Visa",
    "type": "credit"
  },
  "email": "john@example.com",
  "contact": "+919876543210",
  "created_at": 1756455561
}
```

### 4.3 Capture Payment

**Endpoint:** `POST /v1/payments/{payment_id}/capture`

**Purpose:** Captures an authorized payment. Used in two-step payment flows where authorization and capture are separate.

**Request Body:**

| Field      | Type    | Req | Description                                  |
| ---------- | ------- | :-: | -------------------------------------------- |
| `amount`   | integer | ✅  | Amount to capture (≤ authorized amount)      |
| `currency` | string  | ✅  | ISO 4217 currency code                       |

**Response (200 OK):**

Returns a `Payment` object (same structure as Create Payment response) with:
- `status`: `"captured"`
- `captured`: `true`
- `fee`: Platform fee charged (if applicable)
- `tax`: Tax on fee (if applicable)

**Example Request (Full Capture):**

```json
{
  "amount": 50000,
  "currency": "INR"
}
```

**Example Request (Partial Capture):**

```json
{
  "amount": 30000,
  "currency": "INR"
}
```

---

## 5. Error Handling

Razorpay returns errors in a nested `error` object format.

**Error Response Structure:**

```json
{
  "error": {
    "code": "BAD_REQUEST_ERROR",
    "description": "Human-readable error message",
    "field": "field.name",
    "source": "business",
    "step": "payment_initiation",
    "reason": "input_validation_failed"
  }
}
```

**Common Error Codes:**

| Code                  | HTTP Status | Description                    |
| --------------------- | ----------- | ------------------------------ |
| `BAD_REQUEST_ERROR`   | 400         | Invalid request parameters     |
| `GATEWAY_ERROR`       | 502         | Payment gateway error          |
| `SERVER_ERROR`        | 500         | Internal server error          |
| `UNAUTHORIZED_ERROR`  | 401         | Authentication failed          |

**Field-Specific Errors:**

The `field` property indicates which parameter caused the error (e.g., `"amount"`, `"card.number"`).

**Error Sources:**

- `business`: Validation error or business rule violation
- `bank`: Bank declined the transaction
- `customer`: Customer action required
- `gateway`: Payment gateway issue

---

## 6. Integration with Agentic Checkout

### 6.1 Order Linkage

When creating a Razorpay order, the `notes` field **SHOULD** include:

```json
{
  "notes": {
    "checkout_session_id": "csn_abc123",
    "merchant_id": "merchant_xyz"
  }
}
```

This enables:
- Tracking the relationship between checkout sessions and orders
- Webhook correlation
- Audit trail reconstruction

### 6.2 Payment Method Mapping

Map Agentic Checkout payment methods to Razorpay:

| Agentic Protocol | Razorpay Method | Notes                           |
| ---------------- | --------------- | ------------------------------- |
| `card`           | `card`          | Requires full card details      |
| `upi`            | `upi`           | India-specific                  |
| `wallet`         | `wallet`        | Paytm, PhonePe, etc.            |
| `netbanking`     | `netbanking`    | Bank account transfer           |

### 6.3 Amount Representation

Both protocols use **integer amounts in minor units**:

- Agentic Checkout: Minor units (e.g., $20.00 → 2000 cents)
- Razorpay: Paise for INR (e.g., ₹500.00 → 50000 paise)

**Conversion is 1:1** when using the same currency representation.

### 6.4 Status Mapping

| Razorpay Order Status | Agentic Checkout Status    |
| --------------------- | -------------------------- |
| `created`             | `not_ready_for_payment`    |
| `attempted`           | `in_progress`              |
| `paid`                | `completed`                |

| Razorpay Payment Status | Description                      |
| ----------------------- | -------------------------------- |
| `created`               | Payment initiated                |
| `authorized`            | Authorized, awaiting capture     |
| `captured`              | Successfully captured            |
| `failed`                | Payment failed                   |
| `refunded`              | Payment refunded                 |

---

## 7. Security Considerations

### 7.1 Authentication

- **API Keys MUST** be stored securely (e.g., environment variables, secret managers).
- **NEVER** expose API keys in client-side code or public repositories.
- Rotate keys periodically as per security policy.

### 7.2 PCI DSS Compliance

- **Card data MUST NOT** be logged or stored by merchants.
- Use Razorpay's tokenization for recurring payments.
- Implement proper access controls for payment processing systems.

### 7.3 Transport Security

- All API calls **MUST** use HTTPS with TLS 1.2 or higher.
- Validate SSL/TLS certificates.
- Implement certificate pinning for mobile applications (RECOMMENDED).

### 7.4 Webhook Security

When implementing webhooks:
- Verify webhook signatures using Razorpay's webhook secret.
- Validate event authenticity before processing.
- Implement idempotency handling for duplicate events.

---

## 8. Idempotency & Retries

### 8.1 Idempotency Key

Clients **SHOULD** include `Idempotency-Key` header for:
- Order creation
- Payment creation
- Payment capture

Format: Unique string (UUID recommended)

**Behavior:**
- Same key with identical parameters → Returns original result
- Same key with different parameters → May return error (implementation-dependent)

### 8.2 Retry Strategy

For transient failures (5xx errors, network timeouts):
- Implement exponential backoff (e.g., 1s, 2s, 4s, 8s)
- Maximum retry attempts: 3-5
- Use same `Idempotency-Key` across retries

---

## 9. Currency Support

Razorpay primarily supports **INR (Indian Rupee)**.

**Minor Units:**
- 1 INR = 100 paise
- Example: ₹299.00 = 29900 paise

**Future Support:**
Additional currencies may be supported. Clients **SHOULD** validate currency support before order creation.

---

## 10. Validation Rules

### 10.1 Amount Validation

- **Minimum amount:** 100 paise (₹1.00)
- Amounts **MUST** be positive integers
- Capture amount **MUST NOT** exceed authorized amount

### 10.2 Field Length Limits

- `receipt`: Max 40 characters
- `description`: Max 255 characters
- `notes`: Max 15 key-value pairs
- Card `name`: Max 255 characters

### 10.3 Format Validation

- `currency`: 3-letter uppercase ISO 4217 (e.g., `"INR"`)
- `contact`: E.164 format with country code (e.g., `"+919876543210"`)
- `email`: Valid email address format
- `card.expiry_month`: `"01"` to `"12"`
- `card.expiry_year`: 4-digit year
- `card.number`: 13-19 digits
- `card.cvv`: 3-4 digits

---

## 11. Complete Integration Example

### Step 1: Create Checkout Session (Agentic API)

```http
POST /checkout_sessions HTTP/1.1
Authorization: Bearer api_key_123
Content-Type: application/json

{
  "items": [{"id": "item_456", "quantity": 1}],
  "fulfillment_address": {...}
}
```

**Response:** `checkout_session_id: csn_abc123`

### Step 2: Create Razorpay Order

```http
POST /v1/orders HTTP/1.1
Authorization: Basic a2V5X2lkOmtleV9zZWNyZXQ=
Content-Type: application/json
Idempotency-Key: idem_001

{
  "amount": 50000,
  "currency": "INR",
  "receipt": "receipt#123",
  "notes": {
    "checkout_session_id": "csn_abc123",
    "merchant_id": "acme"
  }
}
```

**Response:** `order_id: order_RB58MiP5SPFYyM`

### Step 3: Create Payment

```http
POST /v1/payments HTTP/1.1
Authorization: Basic a2V5X2lkOmtleV9zZWNyZXQ=
Content-Type: application/json
Idempotency-Key: idem_002

{
  "amount": 50000,
  "currency": "INR",
  "order_id": "order_RB58MiP5SPFYyM",
  "email": "john@example.com",
  "contact": "+919876543210",
  "method": "card",
  "card": {
    "number": "4111111111111111",
    "name": "John Doe",
    "expiry_month": "12",
    "expiry_year": "2025",
    "cvv": "123"
  }
}
```

**Response:** `payment_id: pay_29QQoUBi66xm2f`, `status: authorized`

### Step 4: Capture Payment

```http
POST /v1/payments/pay_29QQoUBi66xm2f/capture HTTP/1.1
Authorization: Basic a2V5X2lkOmtleV9zZWNyZXQ=
Content-Type: application/json
Idempotency-Key: idem_003

{
  "amount": 50000,
  "currency": "INR"
}
```

**Response:** `status: captured`

### Step 5: Complete Checkout Session (Agentic API)

```http
POST /checkout_sessions/csn_abc123/complete HTTP/1.1
Authorization: Bearer api_key_123
Content-Type: application/json

{
  "payment_data": {
    "token": "pay_29QQoUBi66xm2f",
    "provider": "razorpay"
  }
}
```

**Response:** `status: completed`, order created

---

## 12. Conformance Checklist

- [ ] Implements order creation with required fields
- [ ] Supports card payment method with full validation
- [ ] Supports UPI payment method (India-specific)
- [ ] Implements payment capture endpoint
- [ ] Uses integer amounts in currency subunits
- [ ] Implements Basic Authentication correctly
- [ ] Returns proper error structures with descriptive messages
- [ ] Supports idempotency via `Idempotency-Key` header
- [ ] Links orders to checkout sessions via `notes` field
- [ ] Validates all input fields per specification
- [ ] Implements HTTPS/TLS 1.2+
- [ ] Handles PCI DSS requirements appropriately

---

## 13. Change Log

- **2025-09-29**: Initial draft. Defined Razorpay integration endpoints, request/response formats, and integration flow with Agentic Checkout Protocol.

