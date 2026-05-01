# AAP Code — Signed Attribution Token

> The core primitive of the Agent Attribution Protocol.

## What It Is

An AAP Code is a cryptographically signed token issued by Rako for every offer. It encodes the attribution fields needed to verify that a recommendation refers to a registry offer issued by Rako.

An agent cannot forge a valid signature. A merchant cannot self-issue a valid Rako code. A competitor cannot generate one that Rako will honour.

## Format

```
aap://rako.sh/v1/{base64url_payload}.{base64url_signature}
```

**Example:**
```
aap://rako.sh/v1/eyJ2IjoiMSIsImlzcyI6InJha28uc2giLCJvZmYiOiIwMUpRWEsxMDAxU01BUlRZMUdCMDAwMDAxIiwibWVyIjoiMDFKUVhLMDAwMVNNQVJUWTAwMDAwMDAwMSIsInBybyI6IlNNQVJUWSIsInByZCI6IjFHQiBSb2xsaW5nIE1vbnRobHkiLCJwcmMiOjYsImN1ciI6IkdCUCIsImNvbSI6eyJ0eXBlIjoiY3BhIiwidmFsdWUiOjh9LCJpYXQiOjE3NDM1ODA3MjUsImV4cCI6MTc0NjE3MjcyNX0.SIGNATURE
```

## Payload Schema

| Field | Type | Description |
|-------|------|-------------|
| `v` | string | Protocol version. Always `"1"` for v1. |
| `iss` | string | Issuer. Always `"rako.sh"`. |
| `off` | string | Offer ID in the AAP registry. |
| `mer` | string | Merchant ID in the AAP registry. |
| `pro` | string | Provider name (e.g. "SMARTY"). |
| `prd` | string | Product name (e.g. "1GB Rolling Monthly"). |
| `prc` | number | Price (e.g. 6.00). |
| `cur` | string | Currency code (e.g. "GBP"). |
| `com` | object | Commission terms. |
| `com.type` | string | `"cpa"`, `"percentage"`, or `"hybrid"`. |
| `com.value` | number | Commission amount (CPA) or rate (percentage). |
| `com.min` | number? | Minimum commission (hybrid only). |
| `iat` | number | Issued at (Unix timestamp). |
| `exp` | number | Expiry (Unix timestamp). Default: 30 days from issuance. |

---

## Payment Metadata Embedding

When a checkout handoff is initiated through AAP, the AAP Code is associated with platform-side attribution context by AAP or its payment orchestration layer where the checkout path supports it. Agents do **not** manually embed AAP Codes in order notes or descriptions.

### How It Works

1. Agent calls `POST /v1/checkout` with a `recommendationId`.
2. AAP creates or returns a hosted checkout link for the merchant's configured checkout path.
3. AAP associates the following attribution context with the checkout or payment object where supported:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01JQXK1001SMARTY1GB000001",
    "aap_session_id": "sess_abc123"
  }
}
```

4. The merchant or payment provider stores this context where the checkout path supports metadata.
5. On payment completion, an authenticated outcome event or merchant report delivers the relevant attribution context back to AAP.
6. AAP verifies the `aap_code` signature and matches it to the recommendation.

### Why Platform-Side Embedding

| Approach | Problem |
|----------|----------|
| Agent embeds AAP Code in order notes | Agent can omit codes or include unverifiable text |
| Merchant embeds AAP Code at checkout | Merchant-side handling can be inconsistent across checkout systems |
| **AAP associates the code in platform-side attribution context** | **AAP can reconcile the checkout outcome against the signed code and recommendation record** |

Because AAP associates the AAP Code with the checkout handoff, later outcome evidence can be checked against the signed payload, recommendation record, and allowed amount/status rules.

### Verification on Webhook

When AAP receives a payment webhook, it:

1. Extracts `metadata.aap_code` from the payment event.
2. Verifies the Ed25519 signature against the public key at `rako.sh/.well-known/jwks.json`.
3. Decodes the payload and confirms the offer, merchant, and price match.
4. Checks the payment amount matches the AAP Code price (within tolerance).
5. Records the conversion state supported by the evidence if all applicable checks pass.

## Signing

AAP Codes are signed using **Ed25519** (EdDSA over Curve25519).

### Why Ed25519
- Fast signing and verification (important for high-throughput offer serving)
- Small signatures (64 bytes)
- Deterministic — same input always produces same signature
- No padding oracle attacks (unlike RSA)
- Widely supported in every language

### Signing Process

1. Construct the payload as a JSON object
2. Serialise to UTF-8 bytes
3. Encode as base64url (no padding)
4. Sign the UTF-8 payload bytes with Rako's Ed25519 private key
5. Encode signature as base64url
6. Concatenate: `aap://rako.sh/v1/{payload_b64url}.{signature_b64url}`

### Verification Process

1. Parse the code: extract payload and signature (split on last `.`)
2. Decode payload from base64url to UTF-8 bytes
3. Decode signature from base64url to bytes
4. Verify the signature against the payload bytes using Rako's Ed25519 public key
5. If valid, decode payload JSON
6. Check `exp` > current time
7. Check `iss` == `"rako.sh"`

### Public Key Distribution

Rako's public key is available at:

```
GET https://api.rako.sh/v1/codes/public-key
```

Returns:
```json
{
  "algorithm": "Ed25519",
  "publicKey": "-----BEGIN PUBLIC KEY-----\nMCowBQYDK2VwAyEA...\n-----END PUBLIC KEY-----",
  "format": "SPKI/PEM"
}
```

Agent builders and merchants can cache this key and verify AAP Codes independently without calling the API.

## API Endpoints

### Issue a code

```
POST /v1/codes/issue
Authorization: Bearer <api_key>

{ "offerId": "01JQXK1001SMARTY1GB000001" }
```

Response:
```json
{
  "aapCode": "aap://rako.sh/v1/...",
  "payload": { ... },
  "expiresAt": "2026-05-02T08:18:45.000Z"
}
```

### Verify a code

```
POST /v1/codes/verify

{ "code": "aap://rako.sh/v1/..." }
```

Response (valid):
```json
{
  "valid": true,
  "issuer": "rako.sh",
  "offer": { "id": "...", "provider": "SMARTY", "product": "1GB Rolling Monthly", "price": 6, "currency": "GBP" },
  "commission": { "type": "cpa", "value": 8 },
  "issuedAt": "2026-04-02T08:18:45.000Z",
  "expiresAt": "2026-05-02T08:18:45.000Z"
}
```

Response (invalid):
```json
{
  "valid": false,
  "error": "Invalid or expired AAP Code."
}
```

## What the AAP Code Asserts

When a valid AAP Code is present:

1. **Rako issued the code.** The signature matches Rako's public key.
2. **The payload was not changed after signing.** Any payload modification invalidates the signature.
3. **The code refers to a registry offer.** The payload identifies the offer, merchant, price, currency, and commission terms as signed at issuance.
4. **The code has an issuance and expiry time.** Consumers can check whether the code is still inside its validity window.
5. **The code can be independently verified.** Builders and merchants can verify the signature without calling the API.

## What the AAP Code Does NOT Assert

1. **Real-time availability.** The offer may have sold out or expired since issuance. Check the `exp` field.
2. **Price accuracy at checkout.** For live-pricing verticals such as flights or hotels, the price may change. The AAP Code captures the price at discovery, not necessarily the checkout price.
3. **Conversion.** Having a code does not mean the user will buy or that a conversion has occurred.
4. **Commercial eligibility.** The code does not by itself prove that a user, merchant, or transaction qualifies for payout.

---

*AAP Specification v1.0.0*
