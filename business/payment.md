# Payment

**Intent**: Define payment behavior for online COD orders, zero-collection
facts, and payment boundaries.

**Reader task**: Use this document to decide whether a cash collection or
zero-collection fact can satisfy an online order payment requirement, and which
exceptions hand off to delivery, refund, or finance.

**Sources**: §6.2 Payment Lifecycle, §7.7 Agent Cash Handling

**Related**:
[order.md](order.md) for order lifecycle;
[delivery.md](delivery.md) for doorstep cash collection;
[refund.md](refund.md) for cash refund liabilities;
[finance.md](finance.md) for daily closing and cash ledger posting;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-03 — Launch payment facts are cash facts only.**

- Online delivery orders are cash-on-delivery at launch.
- Electronic payment, wallet, card, bank transfer, provider reference reuse,
  and merchant-account settlement are not launch payment methods.
- An external provider reference cannot satisfy an order payment requirement in
  launch scope.
- Adding any external payment rail requires an explicit scope decision and
  coordinated updates to order, payment, refund, finance, delivery, and
  authorization rules.

## Boundary

- Payment owns the customer payment state needed by Order and Finance:
  expected COD amount, collected cash fact, zero-collection fact, and payment
  outcome.
- Delivery owns agent field collection and the delivery completion commit point.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody, cash
  variance records, daily closing, ledger posting, and receipts.
- Refund owns customer refund liabilities and payout lifecycle.
- Payment does not own external payment reference submission, provider
  verification, merchant-account configuration, provider refunds, customer
  wallets, or stored external payment credentials at launch.

## State

```
PENDING_COLLECTION -> COLLECTED
PENDING_COLLECTION -> ZERO_COLLECTION
```

Terminal states: `COLLECTED`, `ZERO_COLLECTION`.

| State | Meaning |
| --- | --- |
| `PENDING_COLLECTION` | Active COD expectation derived from the placed order; no customer cash fact has been recorded. |
| `COLLECTED` | Terminal successful cash collection fact; may include an approved short/over variance link. |
| `ZERO_COLLECTION` | Terminal fact for failed delivery where no customer cash was collected. |

## Business Rules

### Supported Payment Methods

- Online delivery orders use COD cash.
- No external payment instruction, merchant-account selection, transaction
  reference field, provider statement check, or late-reference reuse exists in
  launch customer, staff, or admin workflows.
- Active online-fulfillment outlets are expected to support COD cash.

### COD Collection

- COD is collected at the doorstep while the order is `OUT_FOR_DELIVERY`.
- COD is recorded when the order reaches `DELIVERED`.
- The Delivery Agent collects cash before marking delivery complete.
- Delivery coordinates completion. Payment participates by recording the COD
  collection fact and moving payment to `COLLECTED`.
- Delivery completion atomically commits payment status with delivery task
  state, order state, outgoing stock, returned-cylinder field facts, receipt
  issuance, and cash collection facts.
- Expected COD derives from the persisted order total.
- Agents record collected cash but cannot alter expected COD.
- Client requests cannot override expected COD.
- Payment becomes `COLLECTED` when collected cash matches expected COD or when a
  delivery-approved short/over collection variance is attached.
- Approved variance preserves expected amount, collected amount, and variance
  link; it does not rewrite expected COD or the frozen order total.
- If the completion transaction fails before commit, payment remains
  `PENDING_COLLECTION` and no collected cash fact is recorded.

### Failed Delivery and Cancellation

- Failed delivery records a zero-collection payment fact tied to the failed
  delivery reason.
- `PENDING_COLLECTION` is an active expectation, not a durable terminal payment
  fact.
- Cancellation before delivery collection creates no payment terminal state and
  records no customer payment fact.
- Cancellation before delivery collection retires the COD expectation through the
  authoritative Order cancellation outcome.
- Order cancellation is the authoritative outcome for cancelled orders before
  collection.
- Payment does not create `CANCELLED`, `NOT_COLLECTED`, or equivalent terminal
  payment state for pre-collection cancellations.
- Because launch online orders are COD, cancellation and failed delivery do not
  create refund liabilities from prior customer collection.

### Cash Discrepancies

- Short and over collection by Delivery Agents follows the Finance-owned cash
  variance policy in [finance.md](finance.md).
- Approved short/over collection variance does not create a separate payment
  terminal state.
- Payment links to the approved Finance-owned variance record but does not own
  the variance record.
- Approved post-collection overage correction or post-collection price
  adjustment may create a refund liability in [refund.md](refund.md), but does
  not rewrite the frozen order total or payment expectation.
- Payment records preserve expected amount, collected amount, variance link,
  actor, outlet, task, and order identity needed for audit and reconciliation.

## Permissions

Trimmed access matrix rows relevant to payment. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-02 | P-06 | P-10 |
| --- | --- | --- | --- |
| Delivery execution pickup and COD | Own | - | Full |
| Agent cash handover | Own | Scoped receive | Full |
| Daily cash closing | - | Scoped | Full |

## Authorization Edge Cases

**E-01**: Any launch attempt to submit, verify, reuse, or administer an
external payment reference is denied because external payment rails are out of
scope. Runtime authorization treats those capabilities as absent, not as hidden
fallback permissions.
