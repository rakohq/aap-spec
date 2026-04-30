# Payment and Checkout Architecture

> How AAP carries recommendation context into checkout and reconciles conversion evidence later.

## Core principle

AAP does not require agents or browsers to process payments, define attribution state, or decide whether a conversion occurred. Checkout creation is a handoff boundary: a server-side request can create a hosted checkout URL, and later platform-side outcome evidence determines whether a conversion should be recorded.

Implementations may use a payment orchestration layer, a merchant's payment processor, or merchant-reported outcome feeds. These are implementation choices, not protocol-level promises.

---

## Checkout handoff

### User checkout

The common path is a user-facing hosted checkout handoff:

```text
Agent or website → POST /v1/checkout { sessionId, recommendationId }
  ← { checkoutUrl, transactionId, status }

Buyer → opens checkoutUrl
  → completes the merchant checkout journey

Payment or merchant system → outcome evidence to AAP
  → AAP records a conversion if evidence validates
```

**Flow:**

1. An agent records a recommendation and receives a `recommendationId`.
2. A server-side checkout request sends the `sessionId` and `recommendationId` to AAP.
3. AAP validates the recommendation context and creates a hosted checkout link when the integration supports it.
4. The checkout URL is returned as a handoff for the buyer.
5. The buyer completes the checkout journey outside the agent's control.
6. A payment webhook, merchant report, or configured milestone later provides platform-side outcome evidence.
7. AAP records the conversion only if the evidence validates against the recommendation and offer context.

The checkout URL does not by itself prove payment, order acceptance, commission eligibility, or settlement eligibility.

### Autonomous or stored-payment flows

Some integrations may support an agent-initiated payment attempt with a legitimate stored payment method. In those flows, the same boundary still applies: the request may start a payment attempt, but conversion recording depends on later outcome evidence from the payment or merchant system.

AAP does not require agents to vault or manage user payment credentials.

---

## Evidence model

AAP separates recommendation evidence, checkout handoff, conversion evidence, and settlement eligibility.

| Stage | What it means | What it does not prove |
|-------|---------------|------------------------|
| Recommendation | An agent recorded offer context for a session. | That the buyer purchased. |
| Checkout handoff | A hosted checkout or purchase path was created. | That payment succeeded or commission is owed. |
| Conversion evidence | A payment, merchant, or milestone event was reconciled. | That settlement is final. |
| Settlement eligibility | Validation, refund, dispute, and clawback rules have been applied. | That future clawbacks are impossible. |

### Verification levels

| Level | Source | Trust model | Settlement handling |
|-------|--------|-------------|---------------------|
| Payment-observed | Payment processor or payment orchestration webhook | Platform-side payment evidence reconciled with AAP context | Subject to validation, refund, dispute, and clawback windows |
| Merchant-reported | Merchant reports an order, application, policy, activation, or other payable milestone | Merchant system evidence reconciled with AAP context | Subject to configured validation rules |
| Lower-confidence fallback | Redirect, tracked link, extension, or matching signal | Requires additional validation | May settle differently by merchant configuration |

Do not treat browser-supplied parameters, completion-page display, direct website traffic, or `website_direct` classification as agent attribution evidence.

---

## Fees and settlement

AAP commission is separate from the merchant's payment processing fees. Payment processing fees remain between the merchant and its payment processor. AAP settlement logic uses conversion evidence and the merchant's configured commercial terms to calculate commission, network fee, and builder payout.

The protocol does not require commission to be deducted from payment funds. Settlement can happen through invoice, payout, or other merchant-configured settlement processes.

---

## Merchant checkout setup

Merchants may connect checkout and outcome reporting in different ways:

1. Use a hosted checkout or payment orchestration integration.
2. Send payment webhooks or normalized payment events that include AAP context.
3. Report merchant-side order, application, activation, policy, or first-bill milestones.
4. Reconcile refunds, disputes, cancellations, clawbacks, and validation windows.

Any implementation should use least-privilege access and should not expose API keys, webhook secrets, processor credentials, or environment-specific URLs in public docs or client-side code.

---

## AAP Code in checkout metadata

When checkout is created by an AAP-controlled backend path, AAP context can be attached to the payment or checkout record:

```json
{
  "metadata": {
    "aap_code": "aap://rako.sh/v1/{base64url_payload}.{base64url_signature}",
    "aap_recommendation_id": "01KN6KWQDZ6Y5HPW8KFSDPKNSQ",
    "aap_session_id": "01KN6KV0TWMH8VS6TDS3S7V2EJ"
  }
}
```

This metadata should be set by server-side infrastructure, not by the browser or agent client. On outcome receipt, AAP can validate:

1. The `aap_code` signature is valid.
2. The payload matches the recommendation and offer.
3. The amount and currency match the offer within configured tolerance.
4. The outcome status is eligible for the configured conversion flow.

If checks pass, AAP can record a conversion in the ledger. Validation, refund, dispute, and clawback rules still apply before settlement.

---

## Implementation notes

- Hosted checkout is a handoff, not conversion proof.
- Conversion requires later platform-side outcome evidence.
- Agents and browsers should not construct attribution, source, conversion, or settlement state.
- Direct website fallback is not agent attribution evidence.
- Payment orchestration and processor integrations are implementation details, not protocol promises.

---

*AAP Protocol Spec v1 — Payment and Checkout Architecture*
