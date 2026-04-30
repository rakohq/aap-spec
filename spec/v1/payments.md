# Payment Architecture

> Protocol design for payment-observed attribution without Rako processing customer payments.

## Implementation Status

This document describes the AAP v1 payment architecture and verification requirements. Rako's hosted implementation is still closing the full payment-verification loop: payment webhook authentication, AAP Code verification on webhook receipt, amount matching, payload matching, and status reconciliation must all be implemented and tested before public materials describe hosted conversions as fully payment-verified.

## Core Principle

AAP does not need to process customer funds to attribute a conversion. In the designed production model, AAP can orchestrate checkout on the merchant's own payment processor through a payment orchestration layer or other approved checkout path. The merchant pays the same processor fees they'd pay on their own website. Commission is invoiced separately to the merchant as a B2B receivable.

---

## Payment Modes

### Mode 1 — User Checkout (Default)

The agent recommends. The user completes checkout.

```
Agent → checkout request { recommendationId }
  ← { checkoutUrl, transactionId }

User → opens checkoutUrl (hosted checkout link)
  → enters card on merchant-processor-hosted checkout
  → payment settles to merchant via merchant's payment processor

Payment webhook → AAP
  → AAP records conversion after required verification checks pass
```

**Flow:**
1. Agent calls the checkout endpoint or tool with the `recommendationId` from a prior recommendation.
2. AAP creates or records a hosted checkout link on the merchant's connected processor or approved checkout path.
3. AAP embeds the AAP Code in structured attribution metadata.
4. The checkout URL is returned to the agent, which presents it to the user.
5. The user enters payment credentials directly into the merchant-processor-hosted checkout.
6. In the intended merchant-processor model, payment settles from buyer → merchant's payment processor → merchant. AAP does not receive or control buyer funds.
7. The payment event source sends a webhook to AAP with the payment outcome.
8. AAP records the conversion as verified only after the webhook signature, AAP Code signature, metadata payload, amount, and payment status checks pass.

### Mode 2 — Agent Autonomous

The agent has stored payment credentials and completes the purchase without user interaction.

```
Agent → checkout request { recommendationId, paymentMethod }
  ← { confirmation, transactionId, status }

Payment orchestration layer → processes via merchant's payment processor
  → webhook to AAP
  → AAP records conversion after required verification checks pass
```

**Flow:**
1. Agent calls the checkout endpoint or tool with `recommendationId` and a `paymentMethod` (tokenised card, stored credential).
2. AAP creates a payment intent on the merchant's connected processor and requests confirmation.
3. AAP Code is embedded in structured attribution metadata.
4. In the intended merchant-processor model, payment settles to the merchant via the merchant's payment processor. AAP does not receive or control buyer funds.
5. Webhook confirms outcome. Conversion is recorded as verified only after the same webhook, AAP Code, payload, amount, and status checks pass.

**Note:** Mode 2 requires the agent to have legitimate stored payment credentials from the user. AAP does not vault or manage user payment credentials.

---

## Verification Model

### Why Independent Payment Verification Matters

Every party in the transaction has an incentive to misreport:

| Party | Incentive |
|-------|-----------|
| Agent | Wants conversion credit — could fabricate conversions |
| Merchant | Wants to avoid commission — could deny conversions |
| Merchant's payment processor | No stake in AAP's commission — independent payment evidence source |

The protocol treats the merchant's payment processor as the independent evidence source. A payment event is eligible to become a verified conversion only when AAP can authenticate the event source and reconcile the amount, status, recommendation, offer, and signed AAP Code.

### Verification Levels

| Level | Source | Trust | Commission Rate | Settlement Speed |
|-------|--------|-------|-----------------|------------------|
| **Payment-verified** | Authenticated payment webhook plus AAP reconciliation checks | Independent payment evidence confirms the payable outcome | Path-dependent | Standard after validation period |
| **Merchant-reported** | Merchant conversion API or webhook | Merchant attestation plus audit rights | Path-dependent | Standard or extended validation period |
| **Payment-observed** | Payment processor, open banking, or reconciliation feed | Strong where source is independent | Path-dependent | Standard after validation period |
| **Reconciled** | Batch import or manual reconciliation | Evidence-dependent | Path-dependent | Delayed or manual review |

**Payment-verified conversions** require an authenticated event from the merchant's payment processor and successful reconciliation against the signed AAP Code, recommendation, offer, amount, and payment status.

**Merchant-reported conversions** occur when the purchase happens in a merchant-controlled flow or non-payment conversion event. These rely on merchant attestation, audit logs, and validation periods.

**Payment-observed conversions** are confirmed by payment data even if AAP did not create the checkout object.

**Reconciled conversions** are matched from later data sources and may require manual review before settlement.

---

## Fee Structure

AAP introduces **zero additional payment processing fees**.

| Fee | Who Pays | To Whom |
|-----|----------|---------|
| Payment processing fee (example rate set by processor) | Merchant | Merchant's payment processor |
| AAP commission (e.g., £8 CPA) | Merchant | AAP (invoiced monthly as B2B receivable) |
| AAP network fee (20% of commission) | Deducted from commission | AAP retains; remainder to Agent Builder |

The merchant pays the same payment-processing fees they would pay without AAP. Commission is a separate invoice, not a deduction from payment funds.

---

## Merchant Payment Connector Onboarding

Merchants connect their existing payment processor account to AAP through a scoped connector flow:

1. Merchant clicks **"Connect payment processor"** on the AAP dashboard.
2. The connector flow redirects to the merchant's selected processor or processor-authorisation page.
3. Merchant authorises scoped access with the minimum required permissions.
4. AAP stores the authorised connector configuration for the merchant account.
5. AAP can now create payment links and receive payment events for that merchant account.

**Scope restrictions (ENG-08):**
- Read payment intents and payment methods
- Create payment links / checkout sessions
- Receive webhooks
- **Excluded:** payouts, transfers, balance reads, bank account access

---

## Checkout Attribution Metadata

Every AAP-orchestrated checkout carries AAP attribution metadata on the durable checkout, payment, order, application, or equivalent transaction object. For payment-orchestrated flows, this is usually payment metadata:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01KN6KWQDZ6Y5HPW8KFSDPKNSQ",
    "aap_session_id": "sess_abc123",
    "aap_offer_id": "01JQXK1001SMARTY1GB000001",
    "aap_merchant_id": "01JQXK0001SMARTY000000001",
    "aap_checkout_id": "01KN6KWQEENKX01N1TZE3R1TPX"
  }
}
```

This is set by AAP or by a certified merchant integration when creating the checkout record — not by the agent. The checkout system stores it alongside the transaction record, creating an auditable attribution link. Required and recommended fields are defined in [conversion-ledger.md](./conversion-ledger.md).

For a conversion to be recorded as verified, AAP must verify all of the following on webhook receipt:
1. The payment webhook is authenticated as coming from the configured payment event source.
2. The `aap_code` signature is valid (Ed25519 verification).
3. The `aap_code` payload matches the recommendation and offer.
4. The payment amount matches the offer price or merchant-defined conversion value within tolerance.
5. The payment status is a successful payable state.

If any check is missing or fails, the conversion must remain unverified, pending, or rejected according to the implementation's risk policy.

**Hosted implementation note:** these checks are protocol requirements. Public hosted-Rako claims should not describe a payment conversion as fully payment-verified unless the deployed webhook handler implements and tests every check above.

---

## Payment Orchestration Role

AAP uses a payment orchestration layer between AAP and the merchant's payment processor. The orchestration layer is **not** itself the merchant's payment processor.

- Does not hold buyer funds
- Does not become the merchant of record
- Does not replace the merchant's payment processor

The orchestration layer provides:
- **One API** for AAP to create payments across supported processors
- **Payment links** — hosted checkout pages
- **Webhooks** — unified event format regardless of underlying processor
- **Connector framework** — merchant connects their payment processor once, AAP uses it for eligible transactions

---

*AAP Protocol Spec v1 — Payment Architecture*
