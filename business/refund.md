# Refund

**Intent**: Define the customer refund liability lifecycle, customer-presented
collection code checks, and cash payout constraints.

**Reader task**: Use this document to decide when a refund liability exists, who
can pay it, and how customer-presented collection codes authorize payout.

**Source**: §6.6 Refund Lifecycle

**Related**:
[order.md](order.md) for cancellation and failed-delivery outcomes;
[pos.md](pos.md) for POS same-day voids that do not create refund liabilities;
[payment.md](payment.md) for cash collection and zero-collection facts;
[delivery.md](delivery.md) for failed-delivery fee waiver;
[finance.md](finance.md) for daily closing and liability reporting;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Invariants

**BI-12 — A refund liability persists until discharged.**

- A refund owed to a customer remains open until a cash payout event is recorded
  or a Super Admin-approved void/write-off path resolves it.
- The supported launch liability source is an authorized and posted
  Finance-owned cash over-collection correction.
- Post-collection price adjustment is not a launch refund-liability source.
- Daily closing and time passing do not discharge the liability.
- An open liability does not convert to revenue.
- Open refund liabilities appear in the owning outlet's daily closing and
  liability reports until paid, voided, or written off.

**BI-18 — A refund collection code is single-use.**

- The code is invalidated on first successful presentation.
- Failed attempts are audit-logged with actor, outlet, refund ID, timestamp, and
  safe failure reason.

## Boundary

- Refund owns the customer cash-refund liability and payout lifecycle.
- Refund does not reopen cancelled orders, rewrite payment records, or create
  electronic refund workflows.
- Refund payout is an outlet cash event reported through finance and daily
  closing.

## State

```
PENDING_CREATION
  └─► LIABILITY_OPEN
        └─► COLLECTIBLE

COLLECTIBLE
  └─► PAID

LIABILITY_OPEN|COLLECTIBLE
  ├─► VOIDED
  └─► WRITTEN_OFF
```

| State | Meaning |
| --- | --- |
| `PENDING_CREATION` | A liability condition exists and the refund record is being created. |
| `LIABILITY_OPEN` | Refund is owed and eligible for collection-code issuance; no currently usable collection code exists. |
| `COLLECTIBLE` | Code issued; customer can present it at the owning outlet. |
| `PAID` | Terminal successful cash payout. |
| `VOIDED` | Terminal Super Admin-approved void with reason, note, and audit. |
| `WRITTEN_OFF` | Terminal Super Admin-approved write-off with reason, note, and audit. |

## Business Rules

### Cash-Only Refunds

- All customer refunds are cash-only at launch.
- A cash refund does not create customer credit or an internal transfer.

### Cancellation, Unclaimable Closure, and Failed Delivery

- Cancellation before doorstep COD collection does not create a refund
  liability.
- `UNCLAIMABLE` closure before doorstep COD collection does not create a refund
  liability.
- Failed delivery does not create a refund liability.
- When an authorized and posted Finance-owned cash over-collection correction
  confirms customer cash collected above expected COD, the refund lifecycle
  begins from `LIABILITY_OPEN` for the excess amount.
- If Finance cannot authorize or post the correction, no Refund liability is
  created yet; the discrepancy remains Finance-owned until resolved.
- Post-collection refund liabilities do not reopen orders, change order state,
  rewrite the frozen order total, or rewrite the collected payment fact.

### POS Void Boundary

- POS same-day voids are not Refund-owned liabilities at launch.
- A permitted POS same-day void may return cash as a linked void reversal under
  [pos.md](pos.md), [payment.md](payment.md), and [finance.md](finance.md).
- POS customer returns, exchanges, post-sale price adjustments, and POS refund
  liabilities are not launch behavior.
- After daily closing, POS customer refund workflows require explicit
  launch-scope entry with a named source owner before they can create Refund
  liabilities.

### Terminal Exception Outcomes

- `VOIDED` and `WRITTEN_OFF` are terminal Super Admin-approved exception
  outcomes.
- Eligible source states are `LIABILITY_OPEN` and `COLLECTIBLE`.
- The transition requires actor identity, actor role, reason code, reason note,
  timestamp, and audit trail.
- A void or write-off discharges the refund liability for daily-closing and
  liability-reporting purposes.
- A void or write-off does not create an order mutation or payment mutation.

### Source Authorization and Collectibility

- At launch, a refund liability must originate from an authorized and posted
  Finance-owned cash over-collection correction source event.
- Finance owns source authorization, posting, and audit for cash
  over-collection correction.
- Refund owns the resulting liability lifecycle after the source event creates
  `LIABILITY_OPEN`.
- Refund liability creation is not a launch user command at the Refund boundary.
- Customer or staff support requests do not create Refund liabilities unless
  Finance posts a valid cash over-collection correction source event.
- Additional refund-liability source workflows require explicit launch-scope
  entry with a named source owner before they can create Refund liabilities.
- Refund has no amount-based approval threshold at launch.
- Refund amount does not create a separate amount-based hold state.
- Once a valid source event creates the liability, the refund may receive a
  collection code without amount-band hold.
- Payout permission remains separate from liability creation and collection-code
  issuance.

### Collection Eligibility

- Only the original account holder may collect a refund at launch.
- Delegated or proxy collection is not a standard supported workflow.
- Any exceptional override requires an audited exception record, Super Admin
  handling under the active exception policy, explicit reason and evidence
  notes, and full audit trail.
- Outlet Managers cannot independently approve proxy collection.

### Payout Process

At payout, the actor must be an Outlet Manager or Outlet Cashier with explicit
refund-payout permission. The actor must:

1. Verify the one-time collection code.
2. Confirm the collector is the original account holder tied to the source order
   by matching the collector's Identity account to the immutable customer
   account ID captured on the refund/order snapshot.
3. Confirm the phone presented for collection matches the immutable
   refund/order phone snapshot before cash is marked paid.

### Payout Identity Facts

- Refund owns payout eligibility and uses immutable refund/order identity facts
  as the payout authority.
- The payout identity facts are refund ID, source order ID, original customer
  account ID, and the customer/order phone snapshot captured before placement.
- The current Identity verified phone is authentication and contact context. It
  is not the payout authority and cannot replace refund/order snapshot facts.
- Identity account recovery, phone relink, or authentication-link changes do not
  rewrite the refund/order payout identity facts.
- If the current Identity verified phone differs from the refund/order phone
  snapshot, normal payout fails closed. A payout may proceed only through an
  audited Super Admin identity-mismatch exception with reason, evidence, before
  and after identity facts, and the approving actor recorded.
- If the collector's current Identity account cannot be linked to the original
  customer account ID, the case is treated as proxy or exceptional collection
  and follows the collection exception policy.

### Payout Authorization

- A permitted payout actor may disburse any collectible refund within the owning
  outlet's scope.
- Outlet Manager payout authority requires explicit refund-payout permission;
  the Outlet Manager role alone is insufficient.
- Launch refund payouts have no per-refund or per-outlet-business-day cash cap.
- Refund payout permission does not authorize creating, voiding, or writing off
  a refund liability.

### Payout Records

- Every payout attempt captures actor, role, outlet, amount, order/refund ID,
  reason, source record status, active source policy version, current Identity
  account ID when available, current verified phone when available, and
  timestamp.
- Paid cash events also capture the immutable refund/order identity snapshot,
  collection-code verification result, attestation note, and any approved
  identity-mismatch exception reference.

### Delivery Agents

- Delivery Agents do not issue cash refunds.
- When a delivery fails, they record the failure and return physical goods.
- Any refund remains in the outlet cash-refund lifecycle.

### Collection Code Handling

- The collection code is available only through the authenticated customer
  experience.
- The code must not appear in push notifications, email bodies, or SMS.
- An Outlet Manager or Outlet Cashier with explicit refund-payout permission may
  verify a customer-presented code.

### Daily Closing

- Daily closing may complete with open refund liabilities.
- The Outlet Manager must explicitly acknowledge open liabilities and carry them
  forward.

### Outlet Responsibility

- The outlet that collected the original cash payment is responsible for
  disbursing the refund.
- Another outlet cannot pay the refund on the collecting outlet's behalf.

## Permissions

Refund owns authorization decisions for Refund-owned commands and reads. Related
rows are shown here for context; rows enforced by another aggregate remain with
that aggregate. Identity supplies actor, account, grant, and outlet-scope facts
from [identity-auth.md](identity-auth.md).

| Capability | P-01 | P-03 | P-06 | P-10 |
| --- | --- | --- | --- | --- |
| Refund payout cash at outlet | - | Scoped with explicit permission | Scoped with explicit permission | Full |

## Authorization Edge Cases

**E-05**: Refund liabilities have no amount-based approval threshold at launch.
A valid authorized and posted Finance-owned cash over-collection correction that
confirms customer cash collected above expected COD creates a refund liability
eligible for collection-code issuance regardless of amount. Finance owns source
authorization and any required separation-of-duty rule for the source correction.

**E-07**: An Outlet Manager may disburse collectible refunds within outlet scope
only when explicitly granted refund-payout permission. Outlet Managers cannot
create Refund liabilities directly, pay refunds for another outlet, void or
write off refund liabilities, bypass Finance source authorization, or approve
their own Finance source correction when source approval is required.
