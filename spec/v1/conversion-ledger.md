# Conversion Ledger

> Required attribution metadata, conversion lifecycle, and settlement fields.

The conversion ledger is the protocol record that connects a recommendation to a purchase outcome and, eventually, a settlement outcome. It is implementation-neutral: a conversion may be reported by a merchant API, verified by a payment-orchestration webhook, or reconciled from another approved payment source.

## Checkout Attribution Metadata

When checkout is initiated from a recommendation, the checkout record **must** carry enough metadata for Rako to match the resulting transaction back to the signed attribution event.

### Required Fields

| Field | Required | Description |
|-------|----------|-------------|
| `aap_code` | Yes | The signed AAP Code for the recommended offer. |
| `aap_recommendation_id` | Yes | Recommendation ID returned by `POST /v1/recommend`. |
| `aap_session_id` | Yes | Session ID that owns the recommendation. |
| `aap_offer_id` | Yes | Offer ID encoded in the AAP Code and recommendation. |
| `aap_merchant_id` | Yes | Merchant ID encoded in the AAP Code and recommendation. |
| `aap_checkout_id` | Yes | Rako checkout/transaction identifier for this checkout attempt. |

### Recommended Fields

| Field | Description |
|-------|-------------|
| `aap_protocol_version` | Protocol version, e.g. `v1`. |
| `aap_attribution_path` | `aap_flow`, `tracked_link`, `extension`, or `influence`. |
| `aap_created_at` | ISO-8601 timestamp when checkout was initiated. |
| `merchant_order_reference` | Merchant order/application reference when known at checkout time. |
| `payment_reference` | Processor, payment-orchestration, or payment-intent reference when available. |

The metadata may be stored on a payment object, checkout session, order record, application record, or another durable merchant-side transaction object. It does not require a specific processor field name, but integrations **must** preserve these values until conversion reconciliation is complete.

## Matching Rules

A conversion is matched to a recommendation when Rako can establish all of the following:

1. The `aap_code` signature is valid and unexpired at recommendation or checkout time.
2. The `aap_recommendation_id` belongs to the `aap_session_id` and was created by the authenticated agent.
3. The offer and merchant in the AAP Code match the checkout or conversion record.
4. The conversion occurred within the attribution window for its path.
5. The merchant order reference or payment reference has not already been attributed.

If multiple attribution paths could match one conversion, the priority order in [attribution.md](./attribution.md) applies. One conversion can produce at most one commission record.

## Conversion Ledger Record

Each conversion ledger entry records the commercial outcome and the evidence used to support attribution.

| Field | Required | Description |
|-------|----------|-------------|
| `conversion_id` | Yes | Unique Rako conversion ID. |
| `status` | Yes | Current conversion lifecycle status. |
| `attribution_path` | Yes | `aap_flow`, `tracked_link`, `extension`, or `influence`. |
| `verification_level` | Yes | `orchestrated`, `merchant_reported`, `payment_observed`, or `reconciled`. |
| `session_id` | Yes | Matched AAP session ID. |
| `recommendation_id` | Yes | Matched recommendation ID. |
| `agent_id` | Yes | Agent that made the recommendation. |
| `builder_id` | Yes | Builder that owns the agent. |
| `merchant_id` | Yes | Merchant responsible for commission. |
| `offer_id` | Yes | Offer that was recommended. |
| `aap_code_hash` | Yes | Hash of the AAP Code used for matching; raw code storage is optional. |
| `merchant_order_reference` | Yes | Merchant's durable order/application reference. |
| `transaction_value` | Yes | Gross transaction value used for commission calculation. |
| `currency` | Yes | ISO-4217 currency code. |
| `converted_at` | Yes | When the merchant/payment source says conversion occurred. |
| `recorded_at` | Yes | When Rako recorded the ledger entry. |
| `validation_ends_at` | Yes | Earliest time the conversion can become commission-confirmed. |

### Evidence Fields

| Field | Required | Description |
|-------|----------|-------------|
| `evidence_source` | Yes | Source of conversion evidence: merchant API, webhook, payment orchestrator, extension, or reconciliation import. |
| `evidence_reference` | Yes | Source-system event, webhook, payment, order, or file reference. |
| `evidence_received_at` | Yes | When Rako received the evidence. |
| `evidence_hash` | Recommended | Hash of normalized evidence payload for audit and dispute handling. |

## Lifecycle

```
CHECKOUT_INITIATED → CONVERSION_REPORTED → MATCHED → PENDING_VALIDATION → VALIDATED → SETTLEMENT_PENDING → SETTLED
                                      ↓              ↓                ↓
                                  UNMATCHED       REJECTED        DISPUTED → RESOLVED
                                                     ↓                ↓
                                                  CLAWED_BACK      CLAWED_BACK
```

| Status | Meaning |
|--------|---------|
| `checkout_initiated` | AAP created or recorded a checkout attempt with attribution metadata. |
| `conversion_reported` | A merchant, payment source, or reconciliation process reported a conversion candidate. |
| `matched` | Rako matched the candidate to one recommendation. |
| `unmatched` | The candidate could not be matched; it may be retried or manually reviewed. |
| `pending_validation` | Attribution is accepted but cancellation/chargeback/fraud windows remain open. |
| `validated` | Validation period passed and commission can be settled. |
| `rejected` | Conversion is invalid, cancelled before validation, fraudulent, or outside protocol rules. |
| `settlement_pending` | Commission is confirmed and included in the next settlement run. |
| `settled` | Builder payout and Rako network fee have been settled or booked as settled. |
| `disputed` | Merchant or builder has challenged attribution, validation, or clawback. |
| `resolved` | Dispute has been resolved with an audit decision. |
| `clawed_back` | Previously pending or settled commission was reversed under the clawback rules. |

## Settlement Fields

A validated conversion creates or updates a commission settlement record with the following fields.

| Field | Required | Description |
|-------|----------|-------------|
| `commission_id` | Yes | Unique commission/settlement line ID. |
| `conversion_id` | Yes | Linked conversion ledger entry. |
| `commission_type` | Yes | `cpa`, `percentage`, or `hybrid`. |
| `merchant_rate` | Yes | Merchant-defined commission before attribution-path adjustment. |
| `path_multiplier` | Yes | Multiplier from the conversion path, e.g. `1.00`, `0.75`, or `0.50`. |
| `gross_commission` | Yes | Commission owed by merchant before network fee. |
| `network_fee_rate` | Yes | Rako network fee rate. Standard v1 rate is `0.20`. |
| `network_fee_amount` | Yes | Amount retained by Rako. |
| `builder_payout_amount` | Yes | Amount payable to the builder. |
| `currency` | Yes | Settlement currency. |
| `validation_period_days` | Yes | Validation period applied to this conversion. |
| `settlement_status` | Yes | `pending`, `confirmed`, `invoiced`, `paid`, `reversed`, or `disputed`. |
| `settlement_batch_id` | No | Monthly or ad hoc settlement batch identifier. |
| `invoice_reference` | No | Merchant invoice reference, if commission is invoiced. |
| `paid_at` | No | When builder payout was completed. |

Settlement fields describe accounting outcomes only. They do not require AAP to process the customer payment or hold merchant funds.

---

*AAP Specification v1.0.0*
