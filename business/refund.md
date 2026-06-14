# Refund

**Intent**: Define the customer refund liability lifecycle, collection-code
rules, and cash payout constraints.

**Reader task**: Use this document to decide when a refund liability exists, who
can pay it, and why collection-code expiry never discharges the liability.

**Source**: §6.6 Refund Lifecycle

**Related**:
[order.md](order.md) for cancellation and failed-delivery outcomes;
[payment.md](payment.md) for cash collection and zero-collection facts;
[delivery.md](delivery.md) for failed-delivery fee waiver;
[finance.md](finance.md) for daily closing and liability reporting;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-12 — A refund liability persists until discharged.**

- A refund owed to a customer remains open until a cash payout event is recorded
  or a Super Admin-approved void/write-off path resolves it.
- Supported launch liability sources include authorized and posted cash
  over-collection correction and authorized and posted post-collection price
  adjustment.
- Code expiry, daily closing, and time passing do not discharge the liability.
- An open liability does not convert to revenue.
- Open refund liabilities appear in the owning outlet's daily closing and
  liability reports until paid, voided, or written off.

**BI-18 — A refund collection code is single-use and perishable.**

- A collection code has a finite validity window.
- At launch, the code validity window is 24 hours after issuance.
- The code is invalidated on first successful presentation.
- Failed attempts are audit-logged with actor, outlet, refund ID, timestamp, and
  safe failure reason.
- Failed verification attempts do not lock the collection code or refund record
  at launch.
- Collection-code lockout, failed-attempt rate limiting, and cooldown workflows
  are not launch behavior.

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
  ├─► PAID
  └─► CODE_EXPIRED
        └─► COLLECTIBLE

LIABILITY_OPEN|COLLECTIBLE|CODE_EXPIRED
  ├─► VOIDED
  └─► WRITTEN_OFF
```

| State | Meaning |
| --- | --- |
| `PENDING_CREATION` | A liability condition exists and the refund record is being created. |
| `LIABILITY_OPEN` | Refund is owed and eligible for collection-code issuance; no currently usable collection code exists. |
| `COLLECTIBLE` | Code issued; customer can present it at the owning outlet. |
| `PAID` | Terminal successful cash payout. |
| `CODE_EXPIRED` | Code expired; liability remains open and a new code can be issued. |
| `VOIDED` | Terminal Super Admin-approved void with reason, note, and audit. |
| `WRITTEN_OFF` | Terminal Super Admin-approved write-off with reason, note, and audit. |

## Business Rules

### Cash-Only Refunds

- All customer refunds are cash-only at launch.
- Electronic refunds, customer wallet credit, and provider refund workflows are
  not supported.
- A cash refund does not create customer credit or an internal transfer.

### No Prepaid Cancellation Refunds at Launch

- Launch online orders are not prepaid.
- Cancellation before doorstep COD collection does not create a refund
  liability.
- Failed delivery does not create a prepaid refund liability.
- If a separate authorized and posted post-collection adjustment creates a
  refund liability, the refund lifecycle begins from `LIABILITY_OPEN`.
- Post-collection refund liabilities do not reopen orders, change order state,
  rewrite the frozen order total, or rewrite the collected payment fact.

### Terminal Exception Outcomes

- `VOIDED` and `WRITTEN_OFF` are terminal Super Admin-approved exception
  outcomes.
- Eligible source states are `LIABILITY_OPEN`, `COLLECTIBLE`, and
  `CODE_EXPIRED`.
- The transition requires actor identity, actor role, reason code, reason note,
  timestamp, and audit trail.
- A void or write-off discharges the refund liability for daily-closing and
  liability-reporting purposes.
- A void or write-off does not create a provider refund, customer wallet credit,
  order mutation, or payment mutation.

### Source Authorization and Collectibility

- A refund liability must originate from an authorized and posted source
  correction, adjustment, or other launch-approved source event.
- Source authorization is owned by the source workflow that creates the refund
  liability.
- Refund has no amount-based approval threshold at launch.
- Refund amount does not create a separate amount-based hold state.
- Once a valid source event creates the liability, the refund may receive a
  collection code without amount-band hold.
- Payout permission remains separate from liability creation and collection-code
  issuance.

### Collection Code Expiry and Regeneration

```
t = 0    → code issued; refund COLLECTIBLE
t = 24h  → if code not used  → CODE_EXPIRED; liability remains open
           // new code can be issued from CODE_EXPIRED
```

- A permissioned Customer Support Agent or Super Admin can regenerate an expired
  code with audit.
- Regeneration when the customer loses access requires customer verification
  through an audited fallback-action record with reason and audit.
- Failed verification attempts do not require Support action before a later
  valid presentation can be paid.
- The refund liability itself does not expire.

### Collection Eligibility

- Only the original account holder may collect a refund at launch.
- Delegated or proxy collection is not a standard supported workflow.
- Any exceptional override requires an audited exception record, Super Admin
  handling or explicitly permissioned Customer Support Agent fallback under the
  active exception policy, explicit reason and evidence notes, and full audit
  trail.
- Outlet Managers cannot independently approve proxy collection.

### Payout Process

At payout, the Outlet Manager or explicitly permissioned Outlet Cashier must:

1. Verify the one-time collection code.
2. Confirm the collector is the original account holder tied to the order.
3. Confirm the customer/order phone matches before cash is marked paid.

### Payout Authorization

- A permitted payout actor may disburse any collectible refund within the owning
  outlet's scope.
- Launch refund payouts have no per-refund or per-outlet-business-day cash cap.
- Refund payout permission does not authorize creating, voiding, or writing off
  a refund liability.

### Payout Records

- Every payout attempt captures actor, role, outlet, amount, order/refund ID,
  reason, source record status, active source policy version, and timestamp.
- Paid cash events also capture the customer/order identity snapshot and
  attestation note.

### Delivery Agents

- Delivery Agents do not issue cash refunds.
- When a delivery fails, they record the failure and return physical goods.
- Any refund remains in the outlet cash-refund lifecycle.

### Collection Code Reveal

- The collection code is revealed only through the authenticated customer
  experience or audited reveal by a permissioned Customer Support Agent or Super
  Admin after customer verification.
- The code must not appear in push notifications, email bodies, or SMS.
- An Outlet Manager or explicitly permissioned Outlet Cashier may verify a
  customer-presented code.
- Outlet Managers and Outlet Cashiers cannot reveal the active code to the
  customer.

### Daily Closing

- Daily closing may complete with open refund liabilities.
- The Outlet Manager must explicitly acknowledge open liabilities and carry them
  forward.

### Outlet Responsibility

- The outlet that collected the original cash payment is responsible for
  disbursing the refund.
- Another outlet cannot pay the refund on the collecting outlet's behalf.

## Permissions

Trimmed access matrix rows relevant to refunds. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-03 | P-06 | P-07 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Refund initiation | Own request | - | Scoped | Request | Full |
| Refund payout cash at outlet | - | Scoped with explicit permission | Scoped | - | Full |
| Refund collection code management | - | - | - | Scoped with explicit permission | Full |

## Authorization Edge Cases

**E-05**: Refund liabilities have no amount-based approval threshold at launch.
A valid authorized and posted source event may create a refund liability
eligible for collection-code issuance regardless of amount. The source workflow
still owns any required source authorization and separation-of-duty rule.

**E-07**: An Outlet Manager may initiate scoped refund liabilities and disburse
collectible refunds within outlet scope when explicitly permissioned. Outlet
Managers cannot pay refunds for another outlet, void or write off refund
liabilities, bypass source-workflow authorization, or approve their own source
submission when the source workflow requires approval.
