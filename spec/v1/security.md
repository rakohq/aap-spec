# Security

> Cryptographic signing, fraud prevention, and trust model.

## Cryptographic Signing

### Ed25519 Signatures

All AAP Codes are signed using Ed25519 (EdDSA over Curve25519). This provides:

- **Authenticity:** Only Rako's private key can produce valid signatures
- **Integrity:** Any modification to the payload invalidates the signature
- **Non-repudiation:** Rako cannot deny having issued a validly signed code
- **Efficiency:** 64-byte signatures, fast signing/verification

### Key Management

- **Private key:** Stored on Rako's servers. Never transmitted. Used only for signing.
- **Public key:** Published at `/v1/codes/public-key`. Anyone can verify signatures independently.
- **Key rotation:** Keys are rotated annually. Old public keys remain available for verification of previously issued codes. The `iat` (issued at) timestamp determines which key to use.

### Event Chain

Every event in the attribution chain is logged with:
- Timestamp
- Agent/merchant identity
- Event type (search, recommend, click, checkout, conversion)
- Reference to previous event (hash chain)

Checkout source and conversion state should be derived server-side from accepted context and platform-side outcome evidence. Browser-edited parameters, completion-page display, direct website fallback, or `website_direct` classification are not agent attribution evidence.

This creates a tamper-evident audit trail. Modifying a recorded event should be detectable because subsequent events reference the earlier chain state.

## Fraud Prevention

### Agent-Specific Fraud Types

| Fraud Type | Description | Prevention |
|------------|-------------|------------|
| **Session stuffing** | Agent creates thousands of sessions hoping some convert | Rate limiting per agent/builder. Conversion rate floors. |
| **Identity spoofing** | Agent claims to be a different registered agent | Cryptographic identity via API key. Signed session tokens. |
| **Commission hijacking** | Agent intercepts another agent's session | Session tokens bound to originating agent. Non-transferable. |
| **Fake conversions** | Agent simulates a transaction | Platform-side outcome evidence required. Agent self-reporting is not sufficient. |
| **Self-dealing** | Builder creates fake users | Configured merchant, payment, or milestone validation controls. |
| **Churning** | Agent signs up users who immediately cancel | Validation periods + clawback. High churn → builder flagged. |
| **Click fraud** | Automated clicks on fallback links | IP deduplication, rate limiting, bot detection on redirect service. |

### Detection Signals

- Conversion rate anomalies (too high or too low)
- Velocity checks (too many sessions/recommendations in short period)
- Cancellation rate spikes
- Geographic anomalies (agent claims UK but traffic from elsewhere)
- User agent analysis on redirect clicks
- Temporal patterns (all conversions at same time of day)

### Enforcement

| Severity | Action |
|----------|--------|
| Suspicious | Flag for review. Continue paying. |
| Probable fraud | Pause payouts. Manual review. |
| Confirmed fraud | Suspend builder and apply configured clawback policy. |
| Repeat offender | Permanent ban. Report to fraud databases. |

## Trust Model

### Open Spec, Centralised Trust

AAP uses an open-spec, centrally operated trust model:

| Layer | Open or Centralised |
|-------|-------------------|
| The specification | **Open** — anyone can read and implement |
| AAP Code issuance | **Centralised** — only Rako issues valid codes |
| Verification | **Centralised** — Rako verifies conversions |
| Settlement | **Centralised** — Rako settles commission |
| Fraud detection | **Closed** — proprietary models and rules |

### What Each Party Can Verify

**Merchants:**
- All transactions attributed to them
- Commission calculations
- Session logs for any transaction

**Builders:**
- All transactions their agents drove
- Commission earned and settled
- Clawback justifications

**Neither can see:**
- Other parties' data
- Cross-network intelligence
- Fraud model internals

### Why Not Blockchain

The same question comes up every time someone builds a trust layer. The answer for AAP:

- **Speed:** Agent transactions happen in milliseconds. Block confirmation takes seconds to minutes.
- **Cost:** Per-transaction on-chain fees eat into commissions.
- **Privacy:** On-chain data is visible. Merchant and builder data must be siloed.
- **Equivalent integrity goal:** Hash-chained signed logs can provide tamper-evidence without distributed consensus overhead.

The trust model uses cryptographic signatures and auditable records rather than distributed consensus.

## API Security

### Authentication
- API keys transmitted via HTTPS only
- Keys are hashed (SHA-256) before storage — Rako cannot read raw keys
- Keys can be rotated by the builder/merchant at any time
- Compromised keys can be revoked immediately

### Transport
- TLS 1.3 required for all API communication
- HSTS enforced
- Certificate pinning recommended for SDK implementations

### Rate Limiting
- Per-key rate limits prevent abuse
- Graduated response: slow down → temporary block → review
- DDoS protection at the edge

---

*AAP Specification v1.0.0*
