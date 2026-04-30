# API Reference

> AAP REST API v1

Base URL: `https://api.rako.sh`

## Authentication

All API requests require an API key via `Authorization` header or query parameter:

```
Authorization: Bearer aap_xxx
```
or
```
?api_key=aap_xxx
```

Agent builders receive a key on registration. Merchants receive a separate key for conversion reporting.

## Endpoints

### Health

#### `GET /health`

No authentication required.

```json
{ "status": "ok", "version": "0.0.1", "protocol": "AAP" }
```

---

### Offers

#### `GET /v1/offers`

Search and filter available offers. Automatically creates a session.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `vertical` | string | Filter by vertical: `sim`, `broadband`, `energy`, `flights`, `hotels`, `insurance` |
| `provider` | string | Filter by provider name (partial match) |
| `min_price` | number | Minimum price |
| `max_price` | number | Maximum price |
| `data_gb_min` | number | Minimum data in GB (SIM vertical) |
| `contract_months` | number | Contract length (0 = rolling) |

**Response:**

```json
{
  "sessionId": "01KN6KV0TWMH8VS6TDS3S7V2EJ",
  "offers": [
    {
      "id": "01JQXK1001EXAMPLEOFFER001",
      "merchantId": "01JQXK0001EXAMPLEMERCH001",
      "vertical": "sim",
      "provider": "Example Mobile",
      "name": "1GB Rolling Monthly",
      "price": 6.00,
      "currency": "GBP",
      "criteria": {
        "data_gb": 1,
        "minutes": "unlimited",
        "texts": "unlimited",
        "contract_months": 0,
        "network": "three",
        "requires_credit_check": false,
        "5g": false,
        "eu_roaming_included": true
      },
      "aapCode": "aap://rako.sh/v1/...",
      "status": "active"
    }
  ],
  "count": 1,
  "expiresAt": "2026-04-03T08:00:00.000Z"
}
```

---

### Recommendations

#### `POST /v1/recommend`

Record a recommendation. This is the attribution event.

**Request Body:**

```json
{
  "sessionId": "01KN6KV0TWMH8VS6TDS3S7V2EJ",
  "offerId": "01JQXK1001EXAMPLEOFFER001",
  "context": "User asked for the cheapest SIM deal with no contract"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `sessionId` | Yes | Session ID from offer search |
| `offerId` | Yes | ID of the offer being recommended |
| `context` | No | Why this was recommended (user query, comparison rationale) |

**Response:**

```json
{
  "recommendationId": "01KN6KWQDZ6Y5HPW8KFSDPKNSQ",
  "fallbackUrl": "https://aap.link/r/01KN6KWQDZ6Y5HPW8KFSDPKNSQ",
  "attributionRecorded": true,
  "offer": {
    "id": "01JQXK1001EXAMPLEOFFER001",
    "provider": "Example Mobile",
    "name": "1GB Rolling Monthly",
    "price": 6.00,
    "currency": "GBP"
  }
}
```

---

### Checkout

#### `POST /v1/checkout`

Initiate a server-side hosted checkout handoff for a recommended offer. The checkout response is not conversion proof; conversion requires later platform-side outcome evidence.

**Request Body:**

```json
{
  "sessionId": "01KN6KV0TWMH8VS6TDS3S7V2EJ",
  "recommendationId": "01KN6KWQDZ6Y5HPW8KFSDPKNSQ",
  "userDetails": {}
}
```

**Response:**

```json
{
  "transactionId": "01KN6KWQEENKX01N1TZE3R1TPX",
  "status": "checkout",
  "checkoutUrl": "https://checkout.example/hosted/abc123",
  "session": { "id": "...", "status": "checkout" }
}
```

---

### Sessions

#### `GET /v1/sessions/:id`

Get session details including recommendations and conversions.

**Response:**

```json
{
  "session": {
    "id": "01KN6KV0TWMH8VS6TDS3S7V2EJ",
    "agentId": "...",
    "status": "converted",
    "createdAt": "2026-04-02T08:00:00.000Z",
    "expiresAt": "2026-04-03T08:00:00.000Z"
  },
  "recommendations": [...],
  "conversions": [...]
}
```

---

### Conversions (Merchant-facing)

#### `POST /v1/conversions`

Report a completed sale or payable milestone. Called by merchants when merchant-side outcome evidence is available.

**Request Body:**

```json
{
  "sessionId": "01KN6KV0TWMH8VS6TDS3S7V2EJ",
  "orderReference": "ORD-123456",
  "transactionValue": 6.00
}
```

**Response:**

```json
{
  "conversionId": "01KN6KWQEP989CQWK1FPR94FE1",
  "sessionId": "...",
  "orderReference": "ORD-123456",
  "transactionValue": 6.00,
  "commission": {
    "amount": 8.00,
    "networkFee": 1.60,
    "builderPayout": 6.40,
    "type": "cpa"
  },
  "status": "pending",
  "validationPeriodDays": 30
}
```

---

### AAP Codes

#### `POST /v1/codes/issue`

Issue a signed AAP Code for an offer.

```json
{ "offerId": "01JQXK1001EXAMPLEOFFER001" }
```

#### `POST /v1/codes/verify`

Verify an AAP Code. No authentication required.

```json
{ "code": "aap://rako.sh/v1/..." }
```

#### `GET /v1/codes/public-key`

Get Rako's public key for independent verification.

---

### Redirect (Drop-off Recovery)

#### `GET /r/:recommendationId`

No authentication required. User-facing redirect.

Sets tracking cookie, appends UTM parameters, redirects (302) to merchant.

#### `POST /r/:recommendationId/convert`

Report a conversion from a tracked link.

```json
{
  "orderReference": "ORD-TRACK-001",
  "transactionValue": 6.00
}
```

#### `GET /r/:recommendationId/stats`

Get click and conversion stats for a recommendation's fallback link.

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": "Description of what went wrong"
}
```

| HTTP Code | Meaning |
|-----------|---------|
| 400 | Bad request — missing or invalid fields |
| 401 | Unauthorized — missing or invalid API key |
| 404 | Not found — resource doesn't exist |
| 410 | Gone — session expired or offer no longer available |
| 500 | Internal server error |

## Rate Limits

| Tier | Requests/minute |
|------|----------------|
| Standard | 60 |
| Growth | 300 |
| Enterprise | Custom |

Rate limit headers included in every response:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1743580800
```

---

*AAP Specification v1.0.0*
