# AAP Code — Signed Attribution Token

> The core primitive of the Agent Attribution Protocol.

## What It Is

An AAP Code is a cryptographically signed token issued by Rako for every offer. It encodes offer and attribution context in a signed string.

An agent cannot fake one. A merchant cannot self-issue one. A competitor cannot generate one that Rako will honour.

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
| `pro` | string | Provider name (e.g. "Example Mobile"). |
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

When checkout is initiated through an AAP-controlled server-side path, the AAP Code can be embedded in checkout or payment metadata by platform infrastructure. Agents do **not** manually embed AAP Codes in order notes or descriptions.

### How It Works

1. Agent or website calls `POST /v1/checkout` with a `sessionId` and `recommendationId`.
2. AAP validates context and creates a hosted checkout handoff when the integration supports it.
3. AAP sets the following metadata on the checkout or payment object:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01JQXK1001EXAMPLE1GB000001",
    "aap_session_id": "sess_abc123"
  }
}
```

4. The checkout or payment system stores this metadata alongside the record.
5. Later platform-side outcome evidence delivers the metadata back to AAP.
6. AAP verifies the `aap_code` signature and matches it to the recommendation before recording a conversion.

### Why Platform-Side Embedding

| Approach | Problem |
|----------|----------|
| Agent embeds AAP Code in order notes | Agent can forge or omit codes |
| Merchant embeds AAP Code at checkout | Merchant can strip codes to avoid commission |
| Server-side AAP-controlled embedding | Browser or agent clients do not control metadata |

Because AAP creates the checkout handoff through server-side infrastructure, the AAP Code is set before the buyer enters checkout. Agents and browser clients should not be treated as the authority for attribution, source, conversion, or settlement state.

### Verification on outcome evidence

When AAP receives platform-side outcome evidence, it:

1. Extracts `metadata.aap_code` from the outcome event.
2. Verifies the Ed25519 signature against the public key at `rako.sh/.well-known/jwks.json`.
3. Decodes the payload and confirms the offer, merchant, and price match.
4. Checks the amount and currency match the AAP Code price within configured tolerance when relevant.
5. Records the conversion if all checks pass and the configured conversion flow allows it.

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

{ "offerId": "01JQXK1001EXAMPLE1GB000001" }
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
  "offer": { "id": "...", "provider": "Example Mobile", "product": "1GB Rolling Monthly", "price": 6, "currency": "GBP" },
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

## What a valid AAP Code proves

When a valid AAP Code is present:

1. **The code is authentic.** Only Rako's private key could have produced the signature.
2. **The signed payload was issued by Rako.** Offer, merchant, price, currency, and commission fields match the issued payload.
3. **The payload has integrity.** Changing the payload invalidates the signature.
4. **The code has an issuance and expiry time.** Consumers can check whether the code is still inside its validity window.

## What a valid AAP Code does not prove

1. **Real-time availability.** The offer may have sold out or expired since issuance. Check the `exp` field.
2. **Price accuracy at checkout.** For live-pricing verticals, the price may change. The AAP Code captures the price at discovery, not necessarily at checkout.
3. **Conversion.** Having a code does not mean the user will buy.
4. **Settlement eligibility.** Conversion and settlement depend on later outcome evidence and validation rules.

---

*AAP Specification v1.0.0*
