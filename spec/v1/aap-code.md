# AAP Code — Signed Attribution Token

> The core primitive of the Agent Attribution Protocol.

## What It Is

An AAP Code is a cryptographically signed token issued by Rako for every offer. It encodes the offer, merchant, price, currency, commission terms, and expiry in a signed string.

An agent cannot fake a valid Rako-issued code. A merchant cannot self-issue one. A competitor cannot generate one that Rako will honour.

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

When a purchase is initiated through AAP, the AAP Code is embedded in payment metadata by the AAP platform through the payment orchestration layer. Agents do **not** manually embed AAP Codes in order notes or descriptions.

### How It Works

1. Agent calls `POST /v1/purchase` with a `recommendationId`.
2. AAP creates a payment link/intent on the merchant's payment processor through the payment orchestration layer.
3. AAP sets the following metadata on the payment object:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01JQXK1001SMARTY1GB000001",
    "aap_session_id": "sess_abc123"
  }
}
```

4. The merchant's payment processor stores this metadata alongside the payment record.
5. On payment completion, the payment webhook delivers the metadata back to AAP.
6. AAP must verify the payment event source, the `aap_code` signature, and the match between the payment metadata and the recorded recommendation before treating the conversion as verified.

### Why Platform-Side Embedding

| Approach | Problem |
|----------|----------|
| Agent embeds AAP Code in order notes | Agent can forge or omit codes |
| Merchant embeds AAP Code at checkout | Merchant can strip codes to avoid commission |
| **AAP embeds through payment orchestration** | **Stronger attribution path — the agent does not control payment metadata** |

Because AAP creates the payment object through the payment orchestration layer, the AAP Code is set at creation time. The protocol relies on the payment processor returning that metadata in an authenticated payment event and on AAP verifying the event, signature, payload, amount, and status before recording a verified conversion.

### Verification on Webhook

For a payment event to close the verification loop, AAP must:

1. Authenticate the webhook as coming from the configured payment event source.
2. Extract `metadata.aap_code` from the payment event.
3. Verify the Ed25519 signature against the public key at `rako.sh/.well-known/jwks.json`.
4. Decode the payload and confirm the offer, merchant, recommendation, and price match.
5. Check the payment amount matches the AAP Code price (within tolerance).
6. Confirm the payment status is a successful payable state.
7. Record the conversion as **verified** only if all checks pass.

Implementation status should be stated separately from this protocol requirement. A hosted implementation that signs AAP Codes but does not yet verify those codes on the payment webhook path has not closed the verified-payment loop.

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

## What the AAP Code Proves

When a valid AAP Code is present:

1. **The offer was issued by Rako.** It was published in the AAP registry when the code was created.
2. **The price is a signed issuance snapshot.** It was accurate according to Rako's registry at the time of issuance.
3. **The merchant identifier was recorded by Rako.** The code identifies the merchant account associated with the offer at issuance time.
4. **The commission terms are signed.** They were included in the signed payload and cannot be changed without invalidating the code.
5. **The code is authentic.** Only Rako's private key could have produced the signature.

## What the AAP Code Does NOT Guarantee

1. **Real-time availability.** The offer may have sold out or expired since issuance. Check the `exp` field.
2. **Price accuracy at checkout.** For live-pricing verticals (flights, hotels), the price may change. The AAP Code captures the price at discovery, not at checkout.
3. **Conversion.** Having a code doesn't mean the user will buy.

---

*AAP Specification v1.0.0*
