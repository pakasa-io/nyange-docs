# Payment

**Intent**: Define launch payment behavior for cash-on-delivery, walk-in cash
sales, zero-collection facts, and payment boundaries.

**Reader task**: Use this document to decide whether a cash collection or
zero-collection fact can satisfy an order payment requirement, and which
exceptions hand off to delivery, refund, or finance.

**Sources**: §6.2 Payment Lifecycle, §7.7 Agent Cash Handling

**Related**:
[order.md](order.md) for order lifecycle;
[delivery.md](delivery.md) for doorstep cash collection and PIN commit;
[refund.md](refund.md) for cash refund liabilities;
[finance.md](finance.md) for daily closing and cash ledger posting;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-03 — Launch payment facts are cash facts only.**

- Online delivery orders are cash-on-delivery at launch.
- Walk-in POS sales are immediate cash sales at launch.
- Electronic prepayment, wallet, card, bank transfer, provider reference reuse,
  and merchant-account settlement are not launch payment methods.
- An external provider reference cannot satisfy an order payment requirement in
  launch scope.
- Adding any external prepayment rail requires an explicit scope decision and
  coordinated updates to order, payment, refund, finance, delivery, and
  authorization rules.

## Boundary

- Payment owns the customer payment state needed by Order and Finance:
  expected cash due, collected cash fact, zero-collection fact, and payment
  outcome.
- Delivery owns agent field collection and the PIN-confirmation commit point.
- Finance owns cash handover, daily closing, ledger posting, and receipts.
- Refund owns customer refund liabilities and payout lifecycle.
- Payment does not own external payment reference submission, provider
  verification, merchant-account configuration, provider refunds, customer
  wallets, or stored external payment credentials at launch.

## State

```
PENDING_COLLECTION
  ├─► COLLECTED
  ├─► ZERO_COLLECTION
  └─► CANCELLED
```

| State | Meaning |
| --- | --- |
| `PENDING_COLLECTION` | Cash is expected but has not yet been collected. |
| `COLLECTED` | Terminal successful cash collection fact. |
| `ZERO_COLLECTION` | Terminal fact for failed delivery or cancellation where no customer cash was collected. |
| `CANCELLED` | Order or POS sale did not proceed before cash collection. |

## Business Rules

### Supported Payment Methods

- Online delivery orders use COD cash.
- Walk-in POS sales use immediate cash collected at sale completion.
- No external payment instruction, merchant-account selection, transaction
  reference field, provider statement check, or late-reference reuse exists in
  launch customer, staff, or admin workflows.
- Outlet payment-method support is not an allocation criterion at launch.
- Active online-fulfillment outlets are expected to support COD cash.
- POS-capable outlets are expected to support walk-in cash.

### COD Collection

- The Delivery Agent collects cash at the doorstep before PIN confirmation.
- PIN confirmation atomically commits payment status with delivery, order,
  outgoing stock, returned-cylinder field recording, and cash collection facts.
- Expected cash due can include the order total, delivery-fee increase,
  conversion delta, and doorstep price-recalculation delta.
- Approved reductions lower cash due before collection.
- No refund is created for a reduction that is applied before cash is
  collected.
- Agents record collected cash but cannot alter expected cash due.

### Failed Delivery and Cancellation

- Failed delivery records a zero-collection payment fact tied to the failed
  delivery reason.
- Cancellation before delivery collection records no customer payment.
- Because launch online orders are not prepaid, cancellation and failed
  delivery do not create prepaid refund liabilities.

### Cash Discrepancies

- Short and over collection by Delivery Agents follows the delivery cash
  variance policy in [delivery.md](delivery.md).
- Approved post-collection overage correction or post-collection price
  adjustment may create a refund liability in [refund.md](refund.md).
- Payment records preserve expected amount, collected amount, variance link,
  actor, outlet, run, and order identity needed for audit and reconciliation.

### Walk-In POS Cash

- A walk-in sale is not complete until cash collection and inventory commitment
  both succeed.
- The walk-in receipt is issued only after sale completion.
- If cash is not collected, the sale does not proceed.

## Permissions

Trimmed access matrix rows relevant to payment. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-06 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Delivery execution pickup, PIN, COD | - | Own | - | - | Full |
| POS / walk-in sales | - | - | Scoped with explicit permission | Scoped with explicit permission | Full |
| Agent cash handover | - | Own | - | Scoped receive | Full |
| Daily cash closing | - | - | - | Scoped | Full |

## Authorization Edge Cases

**E-01**: Any launch attempt to submit, verify, reuse, or administer an
external payment reference is denied because external prepayment rails are out
of scope. Runtime authorization treats those capabilities as absent, not as
hidden fallback permissions.
