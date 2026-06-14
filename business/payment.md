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
- Finance owns cash handover, daily closing, ledger posting, and receipts.
- Refund owns customer refund liabilities and payout lifecycle.
- Payment does not own external payment reference submission, provider
  verification, merchant-account configuration, provider refunds, customer
  wallets, or stored external payment credentials at launch.

## State

```
PENDING_COLLECTION
  -> COLLECTED
  -> ZERO_COLLECTION
```

Terminal states: `COLLECTED`, `ZERO_COLLECTION`.

| State | Meaning |
| --- | --- |
| `PENDING_COLLECTION` | COD is expected but has not yet been collected. |
| `COLLECTED` | Terminal successful cash collection fact. |
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
- Delivery completion atomically commits payment status with delivery task
  state, order state, outgoing stock, returned-cylinder field facts, and cash
  collection facts.
- Expected COD derives from the persisted order total.
- Agents record collected cash but cannot alter expected COD.
- Client requests cannot override expected COD.

### Failed Delivery and Cancellation

- Failed delivery records a zero-collection payment fact tied to the failed
  delivery reason.
- Cancellation before delivery collection creates no payment terminal state and
  records no customer payment fact.
- Order cancellation is the authoritative outcome for cancelled orders before
  collection.
- Because launch online orders are COD, cancellation and failed delivery do not
  create refund liabilities from prior customer collection.

### Cash Discrepancies

- Short and over collection by Delivery Agents follows the delivery cash
  variance policy in [delivery.md](delivery.md).
- Approved post-collection overage correction or post-collection price
  adjustment may create a refund liability in [refund.md](refund.md).
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
