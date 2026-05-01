# Payment Architecture

> How AAP records checkout handoffs and reconciles purchase outcome evidence without processing payments.

## Core Principle

AAP does not process payments, collect payment credentials, hold funds, or move money. AAP can create or return hosted checkout links through payment orchestration controlled by the platform and merchant configuration. The user completes checkout with the merchant or the merchant's payment provider. Commission is accounted for separately as a merchant-to-AAP receivable, not as a deduction from user payment funds.

---

## Payment Modes

### Mode 1 — User Checkout (Default)

The agent recommends an offer. The user chooses whether to open a hosted checkout link and complete checkout.

```text
Agent → POST /v1/checkout { recommendationId }
  ← { checkoutUrl, transactionId, status }

User → opens checkoutUrl
  → reviews merchant checkout
  → enters payment credentials with the merchant or payment provider
  → payment settles to the merchant through the merchant's payment setup

Payment outcome event or merchant report → AAP
  → AAP reconciles conversion evidence
```

**Flow:**
1. Agent calls `POST /v1/checkout` with the `recommendationId` from a prior recommendation.
2. AAP creates or returns a hosted checkout link for the selected offer.
3. AAP records the AAP Code and recommendation identifiers in platform-side attribution context where supported by the checkout path.
4. The checkout URL is returned to the agent, which presents it to the user.
5. The user enters payment credentials directly with the merchant or payment provider.
6. Payment settles from buyer to merchant. AAP does not touch funds.
7. A payment outcome event or merchant conversion report is reconciled against the recommendation and AAP Code.
8. AAP records the conversion state according to the evidence available and validation checks that pass.

### Mode 2 — Agent-Assisted Checkout (Future)

Some agent-commerce flows may eventually support user-authorised stored payment credentials. This mode is not required for the v1 developer flow.

```text
Agent → POST /v1/checkout { recommendationId, user-authorised payment reference }
  ← { transactionId, status }

Payment outcome event or merchant report → AAP
  → AAP reconciles conversion evidence
```

**Note:** Any agent-assisted payment flow requires explicit user authorisation and compliant credential handling by the relevant payment provider. AAP does not vault or manage user payment credentials.

---

## Verification Model

### Why Outcome Evidence Matters

Every party in the transaction has a different incentive:

| Party | Incentive |
|-------|-----------|
| Agent | Wants commission and may over-claim conversions |
| Merchant | Wants accurate commission liability and may dispute conversions |
| Payment outcome source | Provides transaction status, amount, and attribution metadata where available |

AAP reconciles conversion evidence from the available sources. Evidence may include authenticated payment outcome events, merchant conversion reports, recommendation records, AAP Code validation, amount matching, status checks, and validation-period results.

### Verification Levels

| Level | Source | Evidence Basis | Commission Rate | Settlement Speed |
|-------|--------|----------------|-----------------|------------------|
| **Payment-observed** | Authenticated payment outcome event | Payment status, amount, attribution context, and AAP Code checks match | Full commission where merchant terms allow | Standard after validation period |
| **Merchant-reported** | Merchant conversion report | Merchant report plus recommendation/session matching | Merchant terms, usually after validation checks | Standard or delayed by validation policy |
| **Fallback-reported** | Agent callback, tracked link, or other fallback evidence | Lower-confidence attribution evidence requiring additional validation | Reduced commission where supported | Delayed by extended validation policy |

**Payment-observed conversions** are recorded when authenticated payment outcome evidence can be matched to the recommendation and AAP Code checks pass.

**Merchant-reported conversions** are recorded when the merchant reports a completed sale and the report matches a known recommendation/session.

**Fallback-reported conversions** occur when the purchase happens outside the hosted checkout handoff, such as through a tracked link fallback. These rely on lower-confidence evidence and may carry a longer validation period or reduced commission.

---

## Fee Structure

AAP introduces **no additional card processing fee charged by AAP**.

| Fee | Who Pays | To Whom |
|-----|----------|---------|
| Payment processing fee | Merchant | Merchant's payment provider |
| AAP commission | Merchant | AAP, invoiced or otherwise accounted for as a B2B receivable |
| AAP network fee | Deducted from commission | AAP retains the network fee; the remainder is payable to the agent builder |

The merchant's payment processing fees are separate from AAP commission. Commission is not deducted from user payment funds by the protocol.

---

## Merchant Payment Configuration

Merchants configure the payment or checkout path that AAP may use for hosted checkout handoffs. The exact onboarding method depends on the merchant's payment setup and Rako's supported connectors.

Typical configuration includes:

1. Merchant selects an approved checkout/payment connection in the AAP dashboard.
2. Merchant authorises the minimum required permissions for checkout creation and outcome events.
3. AAP stores the connection configuration securely.
4. AAP can create or return hosted checkout links and reconcile outcome events for the merchant's account.

**Scope restrictions:**
- Create checkout links or payment intents where supported
- Read payment status and attribution metadata needed for reconciliation
- Receive payment outcome events
- **Excluded:** payouts, transfers, balance reads, bank account access, and unrelated merchant account administration

---

## AAP Code in Attribution Context

AAP-controlled checkout paths should associate the AAP Code with the platform-side attribution context:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01JQXK1001SMARTY1GB000001",
    "aap_session_id": "sess_abc123"
  }
}
```

This context is set by AAP or its payment orchestration layer when the checkout object is created. Agents do not set payment metadata themselves.

On outcome receipt, AAP checks:
1. The `aap_code` signature is valid using Ed25519 verification.
2. The `aap_code` payload matches the recommendation and offer.
3. The reported amount matches the offer price within allowed tolerance.
4. The payment or merchant-reported status indicates a completed transaction.
5. The event source and payload pass the applicable authentication and reconciliation checks.

If the applicable checks pass, AAP records the conversion state supported by that evidence.

---

## Payment Orchestration Role

AAP's payment orchestration layer provides a neutral abstraction over merchant checkout and payment-provider differences. It may provide:

- Hosted checkout link creation
- Payment outcome event normalization
- Connector configuration for supported merchant payment setups
- Attribution metadata attachment where supported
- Reconciliation inputs for conversion matching

The protocol does not require public implementers to depend on a named payment vendor. Public integrations should describe this layer generically as payment orchestration, hosted checkout, payment provider, or outcome-event reconciliation.

---

*AAP Protocol Spec v1 — Payment Architecture*
