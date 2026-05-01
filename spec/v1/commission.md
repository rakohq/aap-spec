# Commission Model

> Merchant-set with AAP-defined guardrails.

## Commission Types

Merchants choose their commission structure when they join AAP:

| Type | Description | Example |
|------|-------------|---------|
| **CPA** (Cost Per Acquisition) | Fixed amount per completed transaction | £8 per SIM activation |
| **Percentage** | Percentage of transaction value | 5% of flight booking |
| **Hybrid** | Base CPA + percentage bonus | £5 base + 3% of premium |

## Commission Floors

To prevent a race to the bottom, AAP enforces minimum commission floors per vertical. Merchants cannot set commission below these floors.

| Vertical | Minimum CPA | Minimum Percentage | Rationale |
|----------|-------------|-------------------|-----------|
| **SIM deals** | £3.00 | — | Industry range: £5-40 per activation. Floor protects agent economics on low-value plans. |
| **Broadband** | £8.00 | — | Industry range: £20-80 per signup. Floor set conservatively to encourage merchant adoption. |
| **Energy** | £10.00 | — | Industry range: £25-65 per switch. Regulated vertical, higher floor justified. |
| **Insurance** | £15.00 | 2% | Industry range: £50-100 per policy. Floor accounts for high cancellation rates. |
| **Flights** | — | 1% | Variable transaction values. Percentage floor more appropriate than fixed CPA. |
| **Hotels** | — | 2% | Variable transaction values. Industry standard: 5-15%. Floor set low to attract merchants. |

Floors are reviewed quarterly and adjusted based on:
- Industry benchmark commission rates (Awin, CJ, Impact data)
- Agent builder feedback (are rates high enough to justify integration?)
- Merchant acquisition needs (are floors blocking merchant signups?)

## Network Fee

AAP takes a 20% network fee on every commission paid out.

```
Gross commission (merchant pays)
  × 0.80 = Builder payout
  × 0.20 = AAP network fee

Example:
  SMARTY CPA: £8.00
  Builder receives: £6.40
  AAP receives: £1.60
```

The network fee is non-negotiable for standard tiers. Enterprise merchants (post-2028) may negotiate volume-based discounts (15-18%).

## Commission by Conversion Path

Different conversion paths earn different commission rates:

| Path | Rate | Example (£8 CPA) |
|------|------|-------------------|
| **AAP flow** (direct) | 100% of merchant rate | £8.00 → £6.40 builder, £1.60 AAP |
| **Tracked link** | 75% of merchant rate | £6.00 → £4.80 builder, £1.20 AAP |
| **Browser extension** | 75% of merchant rate | £6.00 → £4.80 builder, £1.20 AAP |
| **Influence attribution** | 50% of merchant rate | £4.00 → £3.20 builder, £0.80 AAP |

The reduced rates for non-direct paths reflect:
- Lower certainty of attribution
- Higher fraud risk
- User chose not to complete through the agent (lower intent signal)

## Validation Period

Commission is not confirmed immediately. A validation period allows for cancellations, returns, and fraud detection.

| Vertical | Validation Period | Rationale |
|----------|------------------|-----------|
| SIM deals | 30 days | Standard cooling-off period |
| Broadband | 31 days | Installation + first bill cycle |
| Energy | 42 days | Switch completion + first bill |
| Insurance | 14 days | Cooling-off period |
| Flights | 7 days | Post-departure |
| Hotels | Post-checkout | After guest stays |

During the validation period:
- Commission status is "pending"
- Customer cancellation → commission clawed back
- Chargeback → commission reversed
- Merchant flags fraud → commission rejected

After validation:
- Commission confirmed → settled to builder
- Late cancellation → merchant absorbs the loss

## Clawback Rules

| Event | During Validation | After Validation |
|-------|------------------|-----------------|
| Customer cancels | Commission clawed back | Merchant absorbs |
| Chargeback | Commission reversed | Commission reversed |
| Fraud detected | Commission rejected | Commission clawed back + builder flagged |
| Merchant disputes | Under review | Under review |

Builders can challenge clawbacks through the dispute process with evidence from the signed event chain.

## Settlement

Settlement is recorded as ledger fields linked to a validated conversion. The protocol fields are defined in [conversion-ledger.md](./conversion-ledger.md); implementations may invoice merchants, batch payouts, or use external payout rails without changing the attribution record.

### Schedule
Commission is settled monthly, after validation periods close for the relevant transactions.

### Payment Rails
Settlement uses standard payout infrastructure:
- Bank transfer where supported
- Processor-managed payout accounts where supported
- Future agent-native payout rails as they mature

### Minimum Payout
£25 minimum per settlement. Below threshold, balance carries forward to next period.

## Commission Calculation Examples

### CPA (SIM deal)
```
Merchant: SMARTY
Commission type: CPA
Commission value: £8.00
Conversion path: AAP flow (100%)

Commission: £8.00
Network fee (20%): £1.60
Builder payout: £6.40
```

### Percentage (Flight)
```
Merchant: British Airways
Commission type: Percentage
Commission value: 5%
Transaction value: £189.00
Conversion path: AAP flow (100%)

Commission: £189 × 0.05 = £9.45
Network fee (20%): £1.89
Builder payout: £7.56
```

### Hybrid (Insurance)
```
Merchant: Aviva
Commission type: Hybrid
Commission min: £15.00
Commission value: 3%
Annual premium: £487.32
Conversion path: Tracked link (75%)

Base commission: £15.00 + (£487.32 × 0.03) = £29.62
Tracked link rate: £29.62 × 0.75 = £22.22
Network fee (20%): £4.44
Builder payout: £17.78
```

---

*AAP Specification v1.0.0*
