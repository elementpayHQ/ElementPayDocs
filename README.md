
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

| Network   | Token |
|-----------|-------|
| Lisk      | USDT  |
| Arbitrum  | WXM   |
| Scroll    | USDC  |
| Base      | USDC  |
| Ethereum  | ETH   |

For Offramp orders, **users must first approve** the contract to spend their tokens before calling the API.

---

## 💸 1. Offramp Flow – Payout to MPESA (Token Approval Required)

An **Offramp order (`order_type: 1`)** allows a user to **cash out crypto directly to a mobile money account (MPESA)**. The tokens are withdrawn from the user's connected wallet and settled to the **MPESA phone number** provided in the request.

> 🔐 **Important:** Before creating an Offramp order, the user **must approve** the Element Pay smart contract to spend their tokens. This is a required step for all ERC-20 tokens.

---

### 🪙 Step 1: Approve the Token

```js
const contract = new ethers.Contract(tokenAddress, erc20ABI, signer);
await contract.approve("0xELEMENT_CONTRACT", "6900000"); // e.g. 6.9 USDC (6 decimals)
```

If the `approve` step is skipped, the smart contract transaction will fail.

---

### 📥 Step 2: Create Offramp Order

Send a POST request to initiate the Offramp payout. This will transfer tokens from the connected wallet to the recipient's MPESA phone number.

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
> 🧾 This request will:
> 
> - Transfer the approved token amount from the user's wallet  
> - Convert the tokens to fiat (KES)  
> - Payout to the specified MPESA phone number  


---

## 📩 2. Create Onramp Order

An **Onramp order (`order_type: 0`)** allows anyone—including international companies or developers—to receive crypto payments in a wallet address after a user pays in **fiat (KES)** via MPESA.

You can:
- Use this to **accept payments from users in Kenya** through MPESA.
- Provide a **webhook URL** to receive real-time updates when the payment is completed or fails.
- Use the webhook response to **update your systems**, **trigger workflows**, or **record the payment** in your database.

ElementPay handles:
- Converting the fiat payment (e.g., from a phone number) into crypto.
- Sending the crypto to the wallet address provided.
- Sending a webhook notification to the given `webhook_url`.

> 💡 This is ideal for businesses, platforms, or wallets looking to **programmatically accept MPESA payments** and receive crypto on-chain in return.

---

### **POST** `/orders/create`

Creates an onramp order from plaintext fiat payload. The server applies markup, encrypts the message, and submits the order on-chain.

---

### ✅ Request Body

#### Root Fields

| Field          | Type   | Required | Description                                                  |
|----------------|--------|----------|--------------------------------------------------------------|
| `user_address` | string | Yes      | User's wallet address (e.g. `0x123...`)                      |
| `token`        | string | Yes      | Address of token to receive                                  |
| `order_type`   | int    | Yes      | Must be `0` for Onramp                                       |
| `webhook_url`  | string | Yes      | URL to receive order status updates (settled/failed/etc)     |
| `fiat_payload` | object | Yes      | Fiat payment details (see below)                             |

#### `fiat_payload` Schema

| Field            | Type   | Required | Description                                     |
|------------------|--------|----------|-------------------------------------------------|
| `amount_fiat`    | float  | Yes      | Amount in KES                                   |
| `cashout_type`   | string | Yes      | Must be `"PHONE"` (currently supported type)    |
| `phone_number`   | string | Yes      | M-PESA number in format `2547XXXXXXXX` (no `+`) |
| `currency`       | string | No       | Defaults to `"KES"`                             |
| `till_number`    | string | No       | Only required for `cashout_type = TILL`         |
| `paybill_number` | string | No       | Required for `cashout_type = PAYBILL`           |
| `account_number` | string | No       | Required for `cashout_type = PAYBILL`           |
| `reference`      | string | No       | Optional reference attached to the payment      |

### 🆕 Example Request with Webhook

```json
POST /orders/create
Headers:
  x-api-key: YOUR_API_KEY_HERE
Content-Type: application/json

{
  "user_address": "0x40C2f2e0326bD1f647fbeB8732529e08B4DB309f",
  "token": "0x05D032ac25d322df992303dCa074EE7392C117b9",
  "order_type": 0,
  "webhook_url": "https://crawdad-modern-squid.ngrok-free.app/webhooks/test",
  "fiat_payload": {
    "amount_fiat": 10,
    "cashout_type": "PHONE",
    "phone_number": "254712531490"
  }
}
```

> 🧾 This request will:
>
> - Accept an MPESA payment from the provided phone number by sending an STK push to their number where they only have to input their security PIN  
> - Convert the KES amount into crypto  
> - Send the tokens to the provided wallet address  
> - Notify the webhook URL of the final status (settled/failed)  


> **ℹ️ Note:** This `webhook_url` will receive a POST request when the order is `SETTLED`, `FAILED`, or otherwise updated.
> Sample webhook response

```json
{'order_id': 'dff5f84c7539d885248921cd43f28b84cb6f5504a7f92f608a2052c15738b6bb', 'status': 'settled', 'amount_fiat': 10.0, 'currency': 'KES', 'token': 'LISK_USDT', 'amount_crypto': 0.075529, 'file_id': 'EPay-dff5f84c75', 'transaction_hash': 'dda8c65e8825c3a870fdce03b32db2199d2d7f351a2179eb6234bec3a2621f79', 'reason': None}
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

> 📡 Integrator Responsibilities:
> 
> - Your system must be able to receive POST requests to the `webhook_url` you provide.
> - On receiving the webhook, **verify the `status` field** (`SETTLED`, `FAILED`, etc).
> - Use the `amount_fiat` and `status` fields to **trigger appropriate business logic** (e.g. unlock a solar charger for X minutes or save the details of the user who paid).
> - Optionally store `order_id` or `transaction_hash` for audit and traceability.


---

## 💱 4. Fetch Token to KES Rates

**GET** `/rates`

Use this endpoint to get the latest conversion rate from supported tokens (USDC, ETH) to KES **before** approving your Offramp transaction. This helps you avoid under-approving the amount and getting on-chain errors.

### ✅ Query Parameters

| Param     | Required | Description                                                      |
|-----------|----------|------------------------------------------------------------------|
| currency  | No       | One of `usdc`, `eth`, `wxm`, `usdt_lisk` (default: `usdc`)       |

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

For questions, email: **<elementpay.info@gmail.com>**
