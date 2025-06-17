# ElementPayDocs

# 📤 Element Pay Onramp API Documentation

**Base URL:**
`https://staging.elementpay.net/api/v1`

---

## 🔐 Authentication

All endpoints require an **API key**. Provide it via the `x-api-key` header:

```
x-api-key: YOUR_API_KEY_HERE
```

---

## 📩 1. Create Onramp Order

**POST** `/orders/create`

Creates an onramp order from plaintext fiat payload. The server applies markup, encrypts the message, and submits the order on-chain.

### ✅ Request Body

#### Root Fields

| Field          | Type   | Required | Description                                       |
| -------------- | ------ | -------- | ------------------------------------------------- |
| `user_address` | string | Yes      | User's wallet address (e.g. `0x123...`)           |
| `token`        | string | Yes      | Address of token to receive (e.g. USDC on Scroll) |
| `order_type`   | int    | Yes      | Must be `0` for Onramp                            |
| `fiat_payload` | object | Yes      | Fiat payment details (see below)                  |

#### `fiat_payload` Schema

| Field            | Type   | Required | Description                                     |
| ---------------- | ------ | -------- | ----------------------------------------------- |
| `amount_fiat`    | float  | Yes      | Amount in KES                                   |
| `cashout_type`   | string | Yes      | Must be `"PHONE"` (currently supported type)    |
| `phone_number`   | string | Yes      | M-PESA number in format `2547XXXXXXXX` (no `+`) |
| `currency`       | string | No       | Defaults to `"KES"`                             |
| `till_number`    | string | No       | Only required for `cashout_type = TILL`         |
| `paybill_number` | string | No       | Required for `cashout_type = PAYBILL`           |
| `account_number` | string | No       | Required for `cashout_type = PAYBILL`           |
| `reference`      | string | No       | Optional reference attached to the payment      |

### 📅 Example Request

```json
POST /orders/create
Headers:
  x-api-key: YOUR_API_KEY_HERE
Content-Type: application/json

{
  "user_address": "0xabc123...",
  "token": "0x06efdbff2a14a7c8e15944d1f4a48f9f95f663a4",
  "order_type": 0,
  "fiat_payload": {
    "amount_fiat": 1000,
    "phone_number": "254712345678",
    "cashout_type": "PHONE"
  }
}
```

### ✅ Example Response

```json
{
  "tx_hash": "0xabc123...",
  "status": "submitted",
  "rate_used": 144.32,
  "amount_sent": 6.93,
  "fiat_paid": 1000
}
```

### ❌ Error Responses

| Code | Reason                             |
| ---- | ---------------------------------- |
| 400  | `Amount must be greater than 0`    |
| 400  | Validation error on `cashout_type` |
| 403  | `Invalid API key`                  |
| 500  | `Internal error occurred`          |

---

## 📱 2. Check Order Status

**GET** `/orders/tx/{tx_hash}`

Poll this endpoint using the transaction hash returned from `/create` to get the latest status.

### ✅ Path Parameter

| Param     | Description                     |
| --------- | ------------------------------- |
| `tx_hash` | Order creation transaction hash |

### 📅 Example Request

```
GET /orders/tx/0xabc123...
Headers:
  x-api-key: YOUR_API_KEY_HERE
```

### ✅ Example Response

```json
{
  "status": "success",
  "message": "Order fetched successfully",
  "data": {
    "order_id": "12345abcde...",
    "status": "FAILED",
    "order_type": "onramp",
    "amount_fiat": 1000,
    "currency": "KES",
    "token": "USDC",
    "wallet_address": "0xabc...",
    "phone_number": "254712345678",
    "receipt_number": "ws_CO_...",
    "transaction_hashes": {
      "creation": "0x...",
      "settlement": "0x...",
      "refund": null
    },
    "failure_reason": "The initiator information is invalid.",
    "created_at": "2025-06-16 17:26:03"
  }
}
```

### 🚫 Errors

| Code | Reason                             |
| ---- | ---------------------------------- |
| 403  | Invalid API Key                    |
| 404  | Order not found for given tx\_hash |

---

## 🚨 Webhooks (Upcoming Requirement)

Currently, webhook URLs will have to be provided to our dev team for testing.
Our team will also provide API Key for usage.
In future versions, API consumers **must provide a webhook URL** when requesting production keys. This will be linked to your API key for automatic notifications.

Once configured, you will receive POST callbacks with the following structure:

```json
{
  "order_id": "1234...",
  "status": "SETTLED", // or FAILED
  "amount_fiat": 1000,
  "currency": "KES",
  "token": "USDC",
  "amount_crypto": 6.93,
  "file_id": null,
  "transaction_hash": "0x...",
  "reason": null
}
```

Webhook fields:

* `order_id`: Unique ID assigned to the order
* `status`: Order status (`PENDING`, `SETTLED`, `FAILED`)
* `reason`: Only included for failures
* `transaction_hash`: Settlement tx hash (if available)

---

## ⚠️ Important Notes

* ✅ Phone number must be in `2547XXXXXXXX` format (**no plus sign**)
* ✅ `cashout_type` must always be `"PHONE"` for this public API
* ❌ **Do not** use for `OFFRAMP`, `PAYBILL`, or `TILL` – those are abstracted
* ✅ Order type is an `enum` (`0 = OnRamp`, `1 = OffRamp`)
* ✅ Use `/orders/tx/{tx_hash}` for polling (statuses: `PENDING`, `SETTLED`, `FAILED`, etc.)

---

For questions, email: **[elementpay.info@gmail.com](mailto:elementpay.info@gmail.com)**
