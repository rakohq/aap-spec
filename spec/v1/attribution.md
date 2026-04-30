# Attribution Model

> Binary between agents. Fallback chain within a single recommendation.

## Two Rules

AAP attribution operates under two distinct rules:

### Rule 1 — Between Agents: Binary, Last-Touch, No Splits

The agent whose session token is on the transaction gets 100% of the commission. No partial credit. No "assisted" attribution. No splits.

**Why:**
- Simplicity drives adoption. Builders don't need to understand multi-touch models.
- Splits create disputes. Multi-touch attribution has plagued traditional affiliate marketing for 20 years.
- Sessions are non-transferable and expire in 24 hours. The architecture prevents competing claims.
- Incentivises agents that complete the full loop (discover → recommend → checkout).

**Example:**
> User asks Agent A about broadband. Agent A recommends an Example Broadband offer via AAP. User doesn't buy. Next day, user asks Agent B about broadband. Agent B recommends the same offer via AAP. User buys through Agent B's valid AAP flow.
>
> **Result:** Agent B's builder is attributed. Agent A's expired session is not attributed.

### Rule 2 — Single-Agent Drop-Off: Fallback Chain

When an agent recommends a product and the user does not continue through the agent checkout flow, fallback paths can preserve lower-confidence attribution context if the merchant configuration supports it.

| Priority | Conversion Path | Commission Rate | Mechanism |
|----------|----------------|-----------------|-----------|
| **Best** | AAP flow (user reaches checkout through accepted agent context) | Full rate | Session-based attribution plus later conversion evidence |
| **Good** | Tracked link (user clicks `aap.link/r/...`) | 75% of full rate | Cookie/UTM via redirect |
| **Okay** | Browser extension (catches user on merchant site) | 75% of full rate | Extension activates tracking |
| **Last** | Influence attribution (data-matched) | 50% of full rate or flat fee | Temporal + product matching |

**Key distinction:** This is not multi-agent attribution. It is single-agent revenue recovery. The agent that made the original recommendation gets credit across all fallback paths.

## The Fallback URL

Every `recommend()` response includes a fallback URL:

```json
{
  "recommendationId": "rec_abc123",
  "fallbackUrl": "https://aap.link/r/rec_abc123",
  "attributionRecorded": true
}
```

When a user clicks this URL:

1. AAP looks up the original recommendation
2. Sets a 30-day tracking cookie
3. Appends UTM parameters (`utm_source=aap`, `utm_medium=agent_referral`)
4. Appends `aap_ref={recommendationId}` for merchant-side matching
5. Redirects (302) to the merchant's product page
6. Logs the click event

If later merchant or payment evidence validates a payable outcome within the configured window, the conversion can be attributed to the original recommending agent at the tracked-link rate.

## Attribution Window

| Path | Window |
|------|--------|
| AAP flow | Session duration (24 hours max) |
| Tracked link | 30 days from click |
| Browser extension | 30 days from activation |
| Influence matching | 24 hours from recommendation |

## What Gets Recorded

When `recommend()` is called, the following attribution event is created:

| Field | Description |
|-------|-------------|
| Recommendation ID | Unique identifier for this recommendation |
| Session ID | The session this recommendation belongs to |
| Agent ID | Which agent made the recommendation |
| Offer ID | Which offer was recommended |
| AAP Code | The signed token for the offer |
| Context | Why the agent recommended this (user query, comparison rationale) |
| Timestamp | When the recommendation was made |
| Fallback URL | The drop-off recovery URL |

## Disputes

### When Disputes Happen
- Merchant says a conversion wasn't genuine
- Builder says they drove a sale that wasn't attributed
- Builder challenges a clawback
- Multiple paths claim the same conversion

### Resolution
Disputes are resolved using the recorded event chain. Events such as search, recommendation, click, checkout handoff, and conversion are logged with timestamps and actor identity. AAP evaluates disputes against this evidence and the merchant's configured validation rules.

### Priority Order
If a conversion could be attributed via multiple paths (e.g., both AAP flow and tracked link), the highest-priority path wins:

1. AAP flow (session-based recommendation plus later conversion evidence) — highest priority
2. Tracked link — takes precedence over extension and influence
3. Browser extension — takes precedence over influence
4. Influence matching — lowest priority

Duplicate attribution should be prevented. One accepted conversion maps to one commission path.

---

*AAP Specification v1.0.0*
