# Payment Architecture

> Protocol design for payment-observed attribution without Rako processing customer payments.

## Implementation Status

This document describes the AAP v1 payment architecture and verification requirements. Rako's hosted implementation is still closing the full payment-verification loop: payment webhooks, AAP Code verification on webhook receipt, amount matching, and payload matching must all be implemented and tested before public materials should describe hosted conversions as fully verified, trustless, or tamper-proof.

## Core Principle

AAP does not process payments. In the designed production model, AAP orchestrates payments on the merchant's own payment processor through a payment orchestration layer. The merchant pays the same processor fees they'd pay on their own website. Commission is invoiced separately to the merchant as a B2B receivable.

---

## Payment Modes

### Mode 1 — User Checkout (Default)

The agent recommends. The user completes checkout.

```
Agent → POST /v1/purchase { recommendationId }
  ← { checkoutUrl, paymentId }

User → opens checkoutUrl (hosted payment link)
  → enters card on merchant's PSP-hosted checkout
  → payment settles to merchant via merchant's PSP

Payment webhook → AAP
  → AAP records conversion after required verification checks pass
```

**Flow:**
1. Agent calls `POST /v1/purchase` with the `recommendationId` from a prior recommendation.
2. AAP creates a hosted payment link on the merchant's connected processor.
3. AAP embeds the AAP Code in the payment's `metadata.aap_code` field.
4. The checkout URL is returned to the agent, which presents it to the user.
5. The user enters payment credentials directly into the PSP-hosted checkout.
6. Payment settles from buyer → merchant's PSP → merchant. AAP never touches funds.
7. The payment orchestration layer sends a webhook to AAP with the payment outcome.
8. AAP records the conversion as **verified** only after the webhook signature, AAP Code signature, metadata payload, amount, and payment status checks pass.

### Mode 2 — Agent Autonomous

The agent has stored payment credentials and completes the purchase without user interaction.

```
Agent → POST /v1/purchase { recommendationId, paymentMethod }
  ← { confirmation, paymentId, status }

Payment orchestration layer → processes via merchant's PSP
  → webhook to AAP
  → AAP records conversion after required verification checks pass
```

**Flow:**
1. Agent calls `POST /v1/purchase` with `recommendationId` and a `paymentMethod` (tokenised card, stored credential).
2. AAP creates a payment intent on the merchant's connected processor and requests confirmation.
3. AAP Code is embedded in payment metadata.
4. Payment settles to merchant via merchant's PSP. AAP never touches funds.
5. Webhook confirms outcome. Conversion is recorded as verified only after the same webhook, AAP Code, payload, amount, and status checks pass.

**Note:** Mode 2 requires the agent to have legitimate stored payment credentials from the user. AAP does not vault or manage user payment credentials.

---

## Verification Model

### Why Independent Payment Verification Matters

Every party in the transaction has an incentive to misreport:

| Party | Incentive |
|-------|-----------|
| Agent | Wants commission — could fabricate conversions |
| Merchant | Wants to avoid commission — could deny conversions |
| Merchant's PSP | No stake in AAP's commission — neutral third party |

The protocol treats the merchant's PSP as the independent evidence source. A payment event is eligible to become a verified conversion only when AAP can authenticate the event source and reconcile the amount, status, recommendation, offer, and signed AAP Code.

### Verification Levels

| Level | Source | Trust | Commission Rate | Settlement Speed |
|-------|--------|-------|-----------------|------------------|
| **Verified** | Authenticated PSP payment webhook plus AAP reconciliation checks | Independent payment evidence confirms the payable outcome | Full commission | Standard (after validation period) |
| **Unverified** | Agent callback or merchant self-report | Trust-dependent — requires validation period | Reduced commission (75%) | Delayed (extended validation period) |

**Verified conversions** require an authenticated event from the merchant's payment processor and successful reconciliation against the signed AAP Code, recommendation, offer, amount, and payment status.

**Unverified conversions** occur when the purchase happens outside AAP's orchestration (e.g., tracked link fallback). These rely on agent callbacks or merchant reporting and carry a longer validation period and reduced commission.

---

## Fee Structure

AAP introduces **zero additional payment processing fees**.

| Fee | Who Pays | To Whom |
|-----|----------|---------|
| PSP processing fee (e.g., 1.4% + 20p) | Merchant | Merchant's PSP (Stripe, Adyen, etc.) |
| AAP commission (e.g., £8 CPA) | Merchant | AAP (invoiced monthly as B2B receivable) |
| AAP network fee (20% of commission) | Deducted from commission | AAP retains; remainder to Agent Builder |

The merchant pays exactly the same PSP fees they'd pay without AAP. Commission is a separate invoice, not a deduction from payment funds.

---

## Merchant Onboarding (OAuth)

Merchants connect their existing PSP account to AAP via OAuth:

1. Merchant clicks **"Connect your Stripe"** on the AAP dashboard.
2. OAuth flow redirects to Stripe (or Adyen, Worldpay, etc.).
3. Merchant authorises scoped access (minimum required permissions).
4. AAP stores the authorised connector configuration for the merchant account.
5. AAP can now create payment links and receive payment events for that merchant account.

**Scope restrictions (ENG-08):**
- Read payment intents and payment methods
- Create payment links / checkout sessions
- Receive webhooks
- **Excluded:** payouts, transfers, balance reads, bank account access

---

## AAP Code in Payment Metadata

Every AAP-orchestrated payment embeds the AAP Code in the PSP's metadata:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01JQXK1001SMARTY1GB000001",
    "aap_session_id": "sess_abc123"
  }
}
```

This is set by AAP when creating the payment through the payment orchestration layer — not by the agent. The agent cannot modify the metadata AAP sets on that payment object.

For a conversion to be recorded as **verified**, AAP must verify all of the following on webhook receipt:
1. The payment webhook is authenticated as coming from the configured payment event source.
2. The `aap_code` signature is valid (Ed25519 verification).
3. The `aap_code` payload matches the recommendation and offer.
4. The payment amount matches the offer price (within tolerance).
5. The payment status is `succeeded`.

If any check is missing or fails, the conversion must remain unverified, pending, or rejected according to the implementation's risk policy.

**Hosted implementation note:** these checks are protocol requirements. Public hosted-Rako claims should not describe a payment conversion as fully verified, trustless, or tamper-proof unless the deployed webhook handler implements and tests every check above.

---

## Payment Orchestration Role

AAP uses a payment orchestration layer between AAP and the merchant's PSP. The orchestration layer is **not** itself the merchant's payment processor.

- Does not hold money
- Does not move money
- Does not replace the merchant's PSP

The orchestration layer provides:
- **One API** for AAP to create payments across supported processors
- **Payment links** — hosted checkout pages
- **Webhooks** — unified event format regardless of underlying processor
- **Connector framework** — merchant connects their PSP once, AAP uses it for eligible transactions

---

*AAP Protocol Spec v1 — Payment Architecture*
