# AAP v1 Specification

> Agent Attribution Protocol — Version 1.0

## 1. Overview

The Agent Attribution Protocol (AAP) is an open specification for attributing AI agent-driven commerce. It defines how:

1. **Merchants** publish offers into a structured registry
2. **Agents** discover, compare, and recommend those offers
3. **Attribution** is recorded cryptographically at the moment of recommendation
4. **Conversions** are matched to recommendations
5. **Commission** is calculated and settled

AAP is an open spec with a centralised trust model. The specification is public — anyone can read it, understand it, build for it. The issuance, verification, and settlement are operated by Rako.

## 2. Participants

### Merchants
Businesses that sell products or services. They publish offers into the AAP registry and pay commission when agent-driven recommendations lead to sales.

### Agent Builders
Developers who build AI agents. They integrate the AAP SDK so their agents can discover offers, make recommendations, and preserve attribution for eligible conversions.

### Agents
AI systems (Claude, ChatGPT, custom agents) that recommend products to users on behalf of their builders.

### Rako (Protocol Authority)
The entity that operates the AAP infrastructure:
- Issues and signs AAP Codes
- Operates the offer registry
- Registers merchants and agents
- Matches recommendations to conversions
- Calculates and settles commission
- Provides fraud detection

## 3. Core Concepts

### AAP Code
A cryptographically signed token issued by Rako for every offer. It is the core attribution primitive. See [aap-code.md](./aap-code.md).

### Session
A bounded interaction between an agent and the AAP API. Created when an agent searches for offers. Expires after 24 hours. One session tracks one user intent through to completion. Sessions are non-transferable.

### Recommendation
An attribution event. When an agent calls `recommend()`, it records who recommended what, when, why, and where. This is the equivalent of a "click" in traditional affiliate marketing.

### Conversion
A completed transaction or approved commercial outcome. When a user purchases, applies, activates, or otherwise completes the merchant-defined conversion event for a recommended product, the merchant or payment evidence source reports the conversion. AAP matches it to the original recommendation and records it in the conversion ledger.

### Commission
The payment from merchant to agent builder for driving a sale. Calculated based on merchant-defined terms, minus the 20% network fee.

## 4. The Attribution Flow

```
1. DISCOVER    Agent calls AAP → searches for offers
               AAP returns offers with signed AAP Codes
               Session token minted

2. RECOMMEND   Agent recommends an offer to the user
               Agent calls recommend() → attribution event recorded
               Fallback URL generated for drop-off recovery

3. TRANSACT    User decides to purchase
               Agent calls checkout() → transaction initiated
               AAP routes to merchant checkout

4. CONVERT     Merchant or payment evidence source confirms the outcome
               Conversion evidence includes checkout attribution metadata
               AAP matches conversion to recommendation and records ledger entry

5. SETTLE      Validation period passes (no cancellation)
               Commission calculated: merchant terms minus 20% network fee
               Builder receives payout
```

## 5. Session Lifecycle

```
CREATED → ACTIVE → CHECKOUT → CONVERTED → VALIDATED → SETTLED
                                  ↓             ↓
                               CANCELLED     DISPUTED → RESOLVED
                                  ↓
                              CLAWED_BACK
```

| State | Trigger |
|-------|---------|
| CREATED | Agent searches for offers |
| ACTIVE | Agent records a recommendation |
| CHECKOUT | Agent initiates checkout |
| CONVERTED | Merchant confirms sale |
| VALIDATED | Validation period passes without cancellation |
| SETTLED | Commission paid to builder |
| CANCELLED | Customer cancels within validation period |
| CLAWED_BACK | Commission reversed due to cancellation |
| DISPUTED | Merchant or builder disputes attribution |
| RESOLVED | Dispute resolved |

### Session Rules
- **One session, one transaction.** A session tracks a single user intent.
- **24-hour expiry.** No checkout within 24 hours = session expires.
- **Non-transferable.** The agent that started the session gets credit.
- **Binary attribution.** Through AAP or nothing. No partial credit.

## 6. Integration Levels

| Level | Package | For |
|-------|---------|-----|
| MCP Server | `@rakohq/mcp` | Claude, ChatGPT, any MCP-compatible agent |
| JavaScript SDK | `@rakohq/sdk` | Web agents, Node.js agents |
| Python SDK | `agent-attribution-protocol` | Python agents, LangChain, CrewAI |
| REST API | `api.rako.sh` | Any language, any platform |

## 7. Sub-specifications

- **[AAP Code](./aap-code.md)** — Signed attribution token format and verification
- **[Attribution Model](./attribution.md)** — Binary attribution + fallback chain
- **[Commission Model](./commission.md)** — Types, floors, calculation, settlement
- **[Conversion Ledger](./conversion-ledger.md)** — Checkout metadata, conversion lifecycle, and settlement fields
- **[API Reference](./api.md)** — REST endpoints
- **[MCP Integration](./mcp.md)** — MCP tool definitions
- **[Payment Architecture](./payments.md)** — Payment modes, verification requirements, and payment orchestration
- **[Security](./security.md)** — Signing, fraud prevention, trust model

## 8. Versioning

The protocol uses semantic versioning. v1 is the initial release. Breaking changes require a major version bump. Non-breaking additions (new optional fields, new endpoints) are minor version bumps.

Current version: **v1.0.0**

---

*AAP Specification v1.0.0 — Published 2 April 2026*
*Rako Ltd — [rako.sh](https://rako.sh)*
