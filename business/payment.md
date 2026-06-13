# Payment

**Intent**: Define payment lifecycle states, payment-reference handling, mobile
money verification, COD collection, and late-reference reuse.

**Reader task**: Use this document to decide whether a payment can satisfy an
order payment gate and how exceptions become refund liabilities or COD top-ups.

**Sources**: §6.2 Payment Lifecycle, §7.10 Late Payment References

**Related**:
[order.md](order.md) for order lifecycle and mobile-money expiry;
[refund.md](refund.md) for overpayment cash-refund liabilities;
[finance.md](finance.md) for post-payment outlet reassignment settlement;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-03 — Payment references are unique per provider.**

- A mobile-money transaction reference cannot be applied to more than one active
  or completed payment per provider.
- The same reference string from different providers is not a duplicate.
- A second attempt to use the same reference at the same provider is rejected
  regardless of customer, outlet, or order.
- The only exception is the same-customer late-reference reuse workflow in
  §7.10. In that workflow, the original cancelled order remains cancelled and
  the reference is re-verified against the recreated order.

## Boundary

- Payment owns reference submission, provider-reference verification, verified
  amount outcomes, and COD collection facts needed to satisfy payment gates.
- Payment does not own cash refund payout, internal settlement, daily closing,
  or financial receipt closure.
- Payment verification is independent from downstream cash and financial
  reconciliation.

## State

```
PENDING
  ├─► [Mobile Money]
  │     AWAITING_CUSTOMER_PAYMENT
  │       └─► AWAITING_STAFF_VERIFICATION
  │             └─► STAFF_VERIFIED
  │                   ├─► PAID
  │                   ├─► PARTIALLY_PAID
  │                   │     └─► PAID
  │                   └─► OVERPAID
  │                         └─► PAID
  │             └─► FAILED
  │
  └─► [COD]
        PENDING
          └─► COLLECTED

  CANCELLED
  REFUNDED
```

| State | Meaning |
| --- | --- |
| `PAID` | Terminal for mobile-money order payment gate. Verified amount equals order total, or an approved exception has resolved the branch. |
| `COLLECTED` | Terminal for COD. PIN confirmation commits agent-recorded cash at door. |
| `PARTIALLY_PAID` | Verified amount is below order total. Fulfillment remains blocked until an Outlet Manager approves a COD top-up delta. |
| `OVERPAID` | Verified amount is above order total. Excess becomes a cash refund liability before fulfillment proceeds. |
| `FAILED` | Terminal bad reference or permanently rejected verification. |
| `CANCELLED` | Order cancelled before payment. |
| `REFUNDED` | Terminal after successful cash refund payout. |

## Business Rules

### Payment Instructions

- Customer-facing mobile-money instructions are shown only after outlet
  allocation.
- Instructions use the assigned outlet's configured merchant account by default.
- The assigned payment account remains part of the payment record.
- A global company account is allowed only as an explicit fallback
  configuration.

### Verification Records

An authorized outlet payment-verification actor records:

- verifier identity;
- verifier role;
- timestamp;
- provider;
- transaction reference;
- verified amount;
- decision.

### Verification Gates

- `STAFF_VERIFIED` is the first operational gate for mobile-money orders.
- Once an authorized outlet actor verifies the provider reference, the payment
  outcome branches by verified amount.

```
STAFF_VERIFIED →
  if verified_amount == order_total  → PAID
  if verified_amount <  order_total  → PARTIALLY_PAID
  if verified_amount >  order_total  → OVERPAID
```

- Outlet acceptance, picking, batching, dispatch, and delivery-agent assignment
  remain blocked until `STAFF_VERIFIED`.
- For `PARTIALLY_PAID`, those operations remain blocked until the COD top-up
  delta is Outlet Manager-approved.
- Unresolved agent cash handover, outlet settlement, refund payout, or daily
  closing does not by itself block recording a valid verification decision.

### Underpayment

- A verified underpayment cannot silently proceed, be waived, or be written off
  at launch.
- It clears only through an Outlet Manager-approved COD top-up for the full
  shortfall, or a Super Admin-approved exception adjustment that records an
  explicit adjustment path.
- Customer Support Agents may route the review but cannot clear payment
  directly.

### Overpayment

- A verified overpayment clears the order payment gate only after the excess is
  handed off as a cash refund liability.
- The excess does not become customer credit, wallet balance, or
  provider-issued refund.

### COD Deliveries

- For failed COD deliveries, no customer cash is collected.
- The order keeps a zero-collection payment fact tied to the failed-delivery
  reason.
- The zero-collection fact is not a payable amount and does not create a refund
  liability.

### Payer Phone

- Payer phone entry is not required at launch.
- Payer phone is not a required verification input matched against the
  customer/order phone.
- Verification relies on provider, transaction reference, verified amount, and
  verification decision.
- The verifier manually compares the provider-statement phone to the
  customer/order phone.
- A phone mismatch is not a clean verification fact and cannot be cleared by
  payment verification alone.
- The Outlet Manager resolves phone mismatches operationally outside the
  payment-verification workflow.

### Discrepancy Resolution

- Payment discrepancy resolution is a durable business workflow for top-up
  approval, COD short/over collection exceptions, and refund-liability handoff.
- Customer Support Agents may route review through support, but cannot mutate
  payment state.
- High-risk approvals and overrides require audit.

## Late Payment References

If a customer submits a mobile-money reference after their order has been
cancelled:

- The reference is flagged as `LATE_REFERENCE_AFTER_CANCELLATION`.
- The cancelled order is not reopened.
- Payment history links the late reference to the cancelled order and, if
  reused, to the recreated order.
- The same customer may reuse the same reference on a new order within 7 days,
  including a recreated order with different items, subject to normal checkout
  validation.
- The recreated order runs normal outlet allocation from scratch.
- The cancelled order remains cancelled.

### Reuse Lifecycle

- When a late reference is reused within 7 days, it enters the payment lifecycle
  at `AWAITING_STAFF_VERIFICATION`.
- The reference already exists and requires re-verification against the new
  order.
- The lifecycle then proceeds normally.
- The `LATE_REFERENCE_AFTER_CANCELLATION` flag on the original order is a
  historical marker only.

### Amount Handling on Reuse

| Comparison | Outcome |
| --- | --- |
| New total equals paid amount | Applies normally |
| New total is greater than paid amount | Shortfall requires approved COD top-up before fulfillment |
| New total is less than paid amount | Cash refund liability is created for the overage |

### Reuse Limits

- After 7 days, mediation is required by an explicitly permissioned Outlet
  Cashier or Outlet Manager within scope, or by a Super Admin.
- Cross-customer reuse of a reference requires an audited override by an actor
  with explicit cross-customer override authority.
- Ordinary self-service reuse remains same-customer only.
- Late references and overpayments do not create customer wallet or credit
  balances at launch.

## Permissions

Trimmed access matrix rows relevant to payment. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-06 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Mobile money verification | - | - | Scoped with explicit permission | Scoped with explicit permission | Full |
| Payment account administration | - | - | - | - | Full |
| Payment reference submission | Own | - | - | - | Full |

## Authorization Edge Cases

**E-01**: A payment-verification actor cannot verify a mobile-money payment for
an order assigned to a different outlet, even if both outlets are in the same
city.
