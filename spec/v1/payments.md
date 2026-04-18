# Payment Architecture

> How AAP verifies purchases trustlessly without processing payments.

## Core Principle

AAP does not process payments. AAP orchestrates payments on the merchant's own payment processor via Hyperswitch. The merchant pays the same processor fees they'd pay on their own website. Commission is invoiced separately to the merchant as a B2B receivable.

---

## Payment Modes

### Mode 1 — User Checkout (Default)

The agent recommends. The user completes checkout.

```
Agent → POST /v1/purchase { recommendationId }
  ← { checkoutUrl, paymentId }

User → opens checkoutUrl (Hyperswitch payment link)
  → enters card on merchant's PSP-hosted checkout
  → payment settles to merchant via merchant's PSP

Hyperswitch → webhook to AAP
  → AAP records verified conversion
```

**Flow:**
1. Agent calls `POST /v1/purchase` with the `recommendationId` from a prior recommendation.
2. AAP creates a Hyperswitch payment link on the merchant's connected processor.
3. AAP embeds the AAP Code in the payment's `metadata.aap_code` field.
4. The checkout URL is returned to the agent, which presents it to the user.
5. The user enters payment credentials directly into the PSP-hosted checkout.
6. Payment settles from buyer → merchant's PSP → merchant. AAP never touches funds.
7. Hyperswitch fires a webhook to AAP confirming the payment outcome.
8. AAP records the conversion as **verified** (trustless — confirmed by neutral PSP).

### Mode 2 — Agent Autonomous

The agent has stored payment credentials and completes the purchase without user interaction.

```
Agent → POST /v1/purchase { recommendationId, paymentMethod }
  ← { confirmation, paymentId, status }

Hyperswitch → processes via merchant's PSP
  → webhook to AAP
  → AAP records verified conversion
```

**Flow:**
1. Agent calls `POST /v1/purchase` with `recommendationId` and a `paymentMethod` (tokenised card, stored credential).
2. AAP creates a Hyperswitch payment intent and processes it directly.
3. AAP Code is embedded in payment metadata.
4. Payment settles to merchant via merchant's PSP. AAP never touches funds.
5. Webhook confirms outcome. Conversion recorded as verified.

**Note:** Mode 2 requires the agent to have legitimate stored payment credentials from the user. AAP does not vault or manage user payment credentials.

---

## Verification Model

### Why Trustless Verification Matters

Every party in the transaction has an incentive to misreport:

| Party | Incentive |
|-------|-----------|
| Agent | Wants commission — could fabricate conversions |
| Merchant | Wants to avoid commission — could deny conversions |
| Merchant's PSP | No stake in AAP's commission — neutral third party |

AAP trusts the PSP. The PSP reports what happened: payment created, succeeded or failed, amount and metadata. This is the same trust model as a bank statement.

### Verification Levels

| Level | Source | Trust | Commission Rate | Settlement Speed |
|-------|--------|-------|-----------------|------------------|
| **Verified** | Hyperswitch PSP webhook | Trustless — neutral third party confirms payment | Full commission | Standard (after validation period) |
| **Unverified** | Agent callback or merchant self-report | Trust-dependent — requires validation period | Reduced commission (75%) | Delayed (extended validation period) |

**Verified conversions** are confirmed by the merchant's payment processor via Hyperswitch webhook. The processor has no relationship with AAP and no reason to fabricate or hide transactions.

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
4. AAP stores the access token as a Hyperswitch connector.
5. AAP can now create payment links and receive webhooks on the merchant's account.

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

This is set by AAP when creating the payment via Hyperswitch — not by the agent. The agent cannot modify or forge payment metadata. The PSP stores it alongside the payment record, creating a tamper-proof attribution link.

On webhook receipt, AAP verifies:
1. The `aap_code` signature is valid (Ed25519 verification).
2. The `aap_code` payload matches the recommendation and offer.
3. The payment amount matches the offer price (within tolerance).
4. The payment status is `succeeded`.

If all checks pass, the conversion is recorded as **verified**.

---

## Hyperswitch's Role

Hyperswitch is an open-source payment switch. It is **not** a payment processor.

- Does not hold money
- Does not move money
- Does not compete with Stripe, Adyen, or Worldpay

Hyperswitch provides:
- **One API** for AAP to create payments across 275+ processors
- **Payment links** — hosted checkout pages
- **Webhooks** — unified event format regardless of underlying processor
- **Connector framework** — merchant connects their PSP once, AAP uses it for all transactions

---

*AAP Protocol Spec v1 — Payment Architecture*
