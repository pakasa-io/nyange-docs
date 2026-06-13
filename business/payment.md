# Payment

**Intent**: Define the payment lifecycle states, rules, and late-payment reference handling for all order payment rails on the Nyange platform.

**Sources**: §6.2 Payment Lifecycle, §7.10 Late Payment References

**Related**: [order.md](order.md) — order lifecycle and mobile money order expiry; [finance.md](finance.md) — F-04 post-payment outlet reassignment; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-03 — Payment references are unique per provider.**
A mobile-money transaction reference cannot be applied to more than one active or completed payment per provider. The same reference string from different providers is not a duplicate. The second attempt to use the same reference at the same provider is rejected regardless of which customer, outlet, or order is involved, except for the explicitly handled same-customer late-reference reuse workflow in §7.10 where the original cancelled order remains cancelled and the reference is re-verified against the recreated order.

---

## State

```
PENDING
  ├─► [Mobile Money]
  │     AWAITING_CUSTOMER_PAYMENT
  │       └─► AWAITING_STAFF_VERIFICATION
  │             └─► STAFF_VERIFIED
  │                   ├─► PAID (verified amount = order total)
  │                   ├─► PARTIALLY_PAID (verified amount < order total; fulfillment blocked until Outlet Manager approves COD top-up)
  │                   │     └─► PAID (approved COD delta is collected at doorstep during delivery confirmation)
  │                   ├─► OVERPAID (verified amount > order total; refund liability opened for excess; fulfillment proceeds after liability handoff)
  │                   │     └─► PAID (order payment gate satisfied; refund lifecycle remains open until payout)
  │             └─► FAILED (bad reference / permanently rejected)
  │
  └─► [COD]
        PENDING ──► COLLECTED (PIN confirmation commits agent-recorded cash at door)

  PAID            (terminal for mobile money)
  COLLECTED       (terminal for COD)
  PARTIALLY_PAID  (underpayment verified; fulfillment blocked until an Outlet Manager approves a COD top-up delta; after approval, fulfillment may proceed and the state resolves to PAID after doorstep collection)
  OVERPAID        (overpayment verified; excess becomes a refund liability before fulfillment proceeds; resolves to PAID for the order payment gate while the refund lifecycle remains open until payout)
  FAILED          (terminal — bad reference or permanently rejected verification)
  CANCELLED       (order cancelled before payment)
  REFUNDED        (terminal — after successful cash refund payout)
```

---

## Business Rules

**Reference uniqueness:**
- A payment reference is unique per provider. The same reference cannot be applied to two orders at the same provider, but the same reference string from different providers is not a duplicate.

**Payment instructions:**
- Customer-facing mobile-money payment instructions are shown only after outlet allocation and use the assigned outlet's configured merchant account by default.
- The assigned payment account remains part of the payment record.
- A global company account is allowed only as an explicit fallback configuration.

**Late references:**
- A reference submitted after order cancellation is marked `LATE_REFERENCE_AFTER_CANCELLATION`. The cancelled order is not reopened. Payment history links the late reference to the cancelled order and, if reused, to the recreated order.

**Verification records:**
- Payment verification by an authorized outlet payment-verification actor records: verifier identity, role, timestamp, provider, transaction reference, verified amount, and decision.

**Verification independence:**
- Payment verification is independent from downstream cash and financial reconciliation.
- Unresolved agent cash handover, outlet settlement, refund payout, or daily closing does not by itself block recording a valid payment-verification decision.

**Failed COD deliveries:**
- For failed COD deliveries, no customer cash is collected, but the order keeps a zero-collection payment fact tied to the failed-delivery reason.
- That fact is not a payable amount and does not create a refund liability.

**Payer phone:**
- Payer phone entry is not required at launch.
- Payer phone is not a required verification input matched against the customer/order phone.
- Payment verification relies on the provider, transaction reference, verified amount, and verification decision, while the verifier manually compares the provider-statement phone to the customer/order phone.
- A phone mismatch is not a clean verification fact and cannot be cleared by payment verification alone; the Outlet Manager must resolve it operationally outside the payment-verification workflow.

**Verification gates:**
- `STAFF_VERIFIED` is the first operational gate for mobile-money orders. Once an authorized outlet payment-verification actor verifies the provider reference, the payment outcome branches to `PAID`, `PARTIALLY_PAID`, or `OVERPAID` based on the verified amount.
- Outlet acceptance, picking, batching, dispatch, and delivery-agent assignment remain blocked until `STAFF_VERIFIED`.
- For `PARTIALLY_PAID`, they remain blocked until the COD top-up delta is Outlet Manager-approved.

**Underpayment:**
- A verified underpayment cannot silently proceed, be waived, or be written off at launch.
- It clears only through an Outlet Manager-approved COD top-up for the full shortfall or a Super Admin-approved exception adjustment that records an explicit adjustment path.
- Customer Support Agents may route the review but cannot clear payment directly.

**Overpayment:**
- A verified overpayment clears the order payment gate only after the excess is handed off as a cash refund liability.
- The excess does not become customer credit, a wallet balance, or a payment-issued refund.

**Discrepancy resolution:**
- Payment discrepancy resolution is a durable business workflow for top-up approval, COD short/over collection exceptions, and refund-liability handoff.
- Customer Support Agents may route review through support, but they cannot mutate payment state.
- High-risk approvals and overrides require audit.

**Late reference reuse in payment lifecycle:**
- When a late reference is reused on a new order within 7 days (§7.10), it enters the payment lifecycle at `AWAITING_STAFF_VERIFICATION` — the reference already exists and requires re-verification against the new order, not a new customer payment.
- The lifecycle then proceeds normally (AWAITING_STAFF_VERIFICATION → STAFF_VERIFIED, etc.).
- The `LATE_REFERENCE_AFTER_CANCELLATION` flag on the original order is a historical marker only.

---

## Late Payment References (§7.10)

If a customer submits a mobile-money reference after their order has been cancelled:

- The reference is flagged; the cancelled order is not reopened. Payment history links the late reference to the cancelled order and, if reused, to the recreated order.
- The same customer may reuse the same reference on a new order within 7 days, including a recreated order with different items, subject to normal checkout validation.
- The recreated order runs normal outlet allocation from scratch; the cancelled order remains cancelled.

**Amount handling on reuse:**

| Comparison | Outcome |
|---|---|
| New total = paid amount | Applies normally |
| New total > paid amount | Shortfall requires an approved COD top-up before fulfillment proceeds |
| New total < paid amount | Cash refund liability created for the overage |

- After 7 days, mediation by an explicitly permissioned Outlet Cashier or Outlet Manager within scope, or by a Super Admin, is required.
- Cross-customer reuse of a reference requires an audited override by one of those actors with explicit cross-customer override authority; ordinary self-service reuse remains same-customer only.
- Late references and overpayments do not create customer wallet or credit balances at launch. They remain specific payment exceptions or refund liabilities.

---

## Permissions

Trimmed access matrix rows relevant to payment. Full matrix: [identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-06 | P-10 |
|---|---|---|---|---|---|
| Mobile money verification | – | – | Scoped (with explicit permission) | Scoped (with explicit permission) | Full |
| Payment account administration | – | – | – | – | Full |
| Payment reference submission | Own | – | – | – | Full |

---

## Authorization Edge Cases

**E-01**: A payment-verification actor cannot verify a mobile-money payment for an order assigned to a different outlet, even if both outlets are in the same city.
