# AAP Code — Signed Attribution Token

> The core primitive of the Agent Attribution Protocol.

## What It Is

An AAP Code is a cryptographically signed token issued by Rako for every offer. It encodes everything needed for attribution in a single, tamper-proof string.

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

## What the AAP Code Guarantees

When a valid AAP Code is present:

1. **The offer is real.** It was published by a verified merchant on the AAP registry.
2. **The price is current.** It was accurate at the time of issuance.
3. **The merchant is vetted.** They passed Rako's merchant verification.
4. **The commission terms are immutable.** They were locked at issuance and cannot be altered retroactively.
5. **The code is authentic.** Only Rako's private key could have produced the signature.

## What the AAP Code Does NOT Guarantee

1. **Real-time availability.** The offer may have sold out or expired since issuance. Check the `exp` field.
2. **Price accuracy at checkout.** For live-pricing verticals (flights, hotels), the price may change. The AAP Code captures the price at discovery, not at checkout.
3. **Conversion.** Having a code doesn't mean the user will buy.

---

*AAP Specification v1.0.0*
