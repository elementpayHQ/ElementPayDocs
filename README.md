
# 📤 Element Pay API Documentation

**Base URL:**  
`https://staging.elementpay.net/api/v1`

---

## 🔐 Authentication

All endpoints require an **API key**. Provide it via the `x-api-key` header:

```
x-api-key: YOUR_API_KEY_HERE
```

---

## 🔁 Order Types and Supported Tokens

Element Pay supports two types of transactions:

| `order_type` | Description  |
|--------------|--------------|
| `0`          | Onramp       |
| `1`          | Offramp      |

### ✅ Supported Tokens

| Network | Token |
|---------|-------|
| Scroll  | USDC  |
| Base    | USDC  |

For Offramp orders, **users must first approve** the contract to spend their tokens before calling the API.

---

## 💸 1. Offramp Flow – Token Approval Required

Before creating an Offramp order (`order_type: 1`), the user **must call `approve()`** on the token contract to allow Element Pay to withdraw the specified amount.

### 🪙 Step 1: Approve the Token

```js
const contract = new ethers.Contract(tokenAddress, erc20ABI, signer);
await contract.approve("0xELEMENT_CONTRACT", "6900000"); // e.g. 6.9 USDC (6 decimals)
```

If the `approve` step is skipped, the smart contract transaction will fail.

---

### 📥 Step 2: Create Offramp Order

```json
POST /orders/create
Headers:
  x-api-key: YOUR_API_KEY_HERE
Content-Type: application/json

{
  "user_address": "0xabc123...",
  "token": "0x06efdbff2a14a7c8e15944d1f4a48f9f95f663a4",
  "order_type": 1,
  "fiat_payload": {
    "amount_fiat": 1000,
    "phone_number": "254712345678",
    "cashout_type": "PHONE"
  }
}
```

---

## 📩 2. Create Onramp Order

**POST** `/orders/create`

Creates an onramp order from plaintext fiat payload. The server applies markup, encrypts the message, and submits the order on-chain.

### ✅ Request Body

#### Root Fields

| Field          | Type   | Required | Description                                       |
| -------------- | ------ | -------- | ------------------------------------------------- |
| `user_address` | string | Yes      | User's wallet address (e.g. `0x123...`)           |
| `token`        | string | Yes      | Address of token to receive                       |
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

### 📅 Example Request for Onramp

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

---

## 📱 3. Check Order Status

**GET** `/orders/tx/{tx_hash}`

Use the transaction hash returned in `/orders/create` to fetch the latest order status.

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

---

## 🚨 Webhooks 

Webhook URLs must be submitted to the dev team during onboarding. In future, webhook URLs will be linked to your API key.

Webhook payload:

```json
{
  "order_id": "1234...",
  "status": "SETTLED",
  "amount_fiat": 1000,
  "currency": "KES",
  "token": "USDC",
  "amount_crypto": 6.93,
  "file_id": null,
  "transaction_hash": "0x...",
  "reason": null
}
```

---

## 💱 4. Fetch Token to KES Rates

**GET** `/rates`

Use this endpoint to get the latest conversion rate from supported tokens (USDC, ETH) to KES **before** approving your Offramp transaction. This helps you avoid under-approving the amount and getting on-chain errors.

### ✅ Query Parameters

| Param     | Required | Description                          |
|-----------|----------|--------------------------------------|
| currency  | No       | Either `usdc` (default) or `eth`     |

### 📅 Example Request

```
GET /rates?currency=usdc
```

### ✅ Example Response

```json
{
  "currency": "usdc",
  "base_rate": 144.97,
  "marked_up_rate": 148.59,
  "markup_percentage": 2.5
}
```

### 🧠 Why This Matters

When using the **offramp flow**, you must first call `approve()` for the token.  
Use this `/rates` route to calculate the amount of tokens (e.g. USDC) to approve:

```
approve_amount = amount_fiat / marked_up_rate
```

This ensures the user has approved **enough tokens** before calling `/orders/create` with `order_type: 1`.

---

## 📦 5. Check Order Status by Transaction Hash

**GET** `/orders/tx/{tx_hash}`

Retrieve the full metadata of an order using its **creation transaction hash**.  
This is the same hash returned after calling `/orders/create`.

### 🔐 Authentication

This endpoint requires your API key:

```
x-api-key: YOUR_API_KEY_HERE
```

### ✅ Path Parameter

| Param     | Description                               |
|-----------|-------------------------------------------|
| `tx_hash` | The transaction hash from `createOrder()` |

### 📅 Example Request

```
GET /orders/tx/0xabc123...
```

### ✅ Example Response

```json
{
  "status": "success",
  "message": "Order fetched successfully",
  "data": {
    "order_id": "12345abcde...",
    "status": "PENDING",
    "order_type": "onramp",
    "amount_fiat": 1000,
    "currency": "KES",
    "token": "USDC",
    "wallet_address": "0xabc...",
    "phone_number": "254712345678",
    "receipt_number": "ws_CO_...",
    "transaction_hashes": {
      "creation": "0xabc123...",
      "settlement": null,
      "refund": null
    },
    "created_at": "2025-06-16 17:26:03"
  }
}
```

### 📘 Possible `status` values

| Status              | Meaning                                                                 |
|---------------------|-------------------------------------------------------------------------|
| `PENDING`           | Awaiting settlement (e.g. user has not completed payment or approval)   |
| `SETTLED`           | Successfully processed and funds released                               |
| `FAILED`            | Order failed (e.g. user cancelled STK push or allowance was insufficient)|
| `SETTLED_UNVERIFIED`| Blockchain confirms payment but off-chain verification failed            |

Use this endpoint to build order tracking, payment feedback, or trigger webhook retries if needed.

## ⚠️ Important Notes

- Phone number format: `2547XXXXXXXX` (no plus sign)
- Offramp requires pre-approved token amount
- Order status can be `PENDING`, `SETTLED`, `FAILED`, etc.
- Supported tokens: USDC on Scroll and Base

---


For questions, email: **elementpay.info@gmail.com**
