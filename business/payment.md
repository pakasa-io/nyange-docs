# Payment

**Intent**: Define payment behavior for online COD orders, walk-in POS cash
sales, durable COD expectations, collection facts, zero-collection facts, and
payment boundaries.

**Reader task**: Use this document to decide whether a cash collection,
zero-collection fact, or POS cash fact can satisfy a launch payment requirement,
and which exceptions hand off to delivery, POS Sale, refund, or finance.

**Sources**: §6.2 Payment Lifecycle, §7.7 Agent Cash Handling

**Related**:
[order.md](order.md) for order lifecycle;
[pos.md](pos.md) for walk-in POS sale completion and void triggers;
[delivery.md](delivery.md) for doorstep cash collection;
[refund.md](refund.md) for cash refund liabilities;
[finance.md](finance.md) for daily closing and cash ledger posting;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Invariants

**BI-03 — Payment facts are cash facts only.**

- Online delivery orders are cash-on-delivery.
- Walk-in POS sales are immediate cash at the outlet.
- A payment fact records COD cash collection at delivery, POS cash collection at
  counter completion, or zero collection after failed delivery.

## Boundary

- Payment owns the customer payment state needed by Order, POS Sale, and Finance:
  durable COD expectation records, expected COD amount, collected cash fact,
  POS cash fact, zero-collection fact, and payment outcome.
- Order owns order placement, cancellation, `UNCLAIMABLE`, `DELIVERED`, and
  `DELIVERY_FAILED` outcomes that create, retire, or complete the Payment-owned
  COD expectation.
- Delivery owns agent field collection and the delivery completion commit point.
- POS Sale owns counter-sale completion, abandonment, and same-day void
  eligibility.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody, cash
  variance records, daily closing, ledger posting, and receipts.
- Refund owns customer refund liabilities and payout lifecycle.

## State

```
PENDING_COLLECTION -> COLLECTED
PENDING_COLLECTION -> ZERO_COLLECTION
PENDING_COLLECTION --retired by Order CUSTOMER_CANCELLED|UNCLAIMABLE--> (no payment terminal state)
(no expectation) -> POS_CASH_COLLECTED
POS_CASH_COLLECTED -> POS_CASH_VOIDED
```

Terminal states: `COLLECTED`, `ZERO_COLLECTION`, and `POS_CASH_VOIDED`.
`POS_CASH_COLLECTED` is terminal after the same-day POS void window closes
without a void.

| State | Meaning |
| --- | --- |
| `PENDING_COLLECTION` | Durable active COD expectation derived from the placed order; no customer cash fact has been recorded and the expectation has not been retired. |
| `COLLECTED` | Terminal successful cash collection fact; may include an approved short/over variance link. |
| `ZERO_COLLECTION` | Terminal fact for a failed delivery finalized with no customer cash collected. |
| `POS_CASH_COLLECTED` | Successful immediate cash collection fact for a completed POS sale; voidable until the same-day POS void window closes. |
| `POS_CASH_VOIDED` | Terminal linked reversal fact for an allowed same-day POS void. |

Retired COD expectations remain durable records, but retirement is not a Payment
state and does not create a terminal payment fact.

## Business Rules

### Supported Payment Methods

- Online delivery orders use COD cash.
- Active online-fulfillment outlets are expected to support COD cash.
- Walk-in POS sales use exact immediate `POS_CASH`.
- Mobile-money, card, customer credit, partial payment, short POS collection,
  and over POS collection are not supported at launch.

### COD Expectation Creation

- Order placement is the business trigger for COD expectation creation.
- When Order commits a new online COD order in `PENDING`, Payment creates one
  durable `PENDING_COLLECTION` expectation for that order in the same atomic
  placement commit.
- If the Payment-owned COD expectation cannot be created, order placement fails
  and no order is committed.
- The expectation stores:
  - order ID and public order number;
  - customer identity reference;
  - frozen order snapshot reference;
  - expected COD amount from the frozen order total;
  - currency;
  - payment method `COD_CASH`;
  - expectation state `PENDING_COLLECTION`;
  - placement idempotency key and correlation ID;
  - creation timestamp.
- Expected COD amount and currency are immutable after expectation creation.
- A placed order has exactly one Payment-owned COD expectation.
- COD expectation creation is not an independent customer or staff command.
- Order creation idempotency covers Payment expectation creation.
- Identical order-placement replays return the original order and original COD
  expectation.
- Same-key/different-body order-placement replays are rejected.

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

### POS Cash Collection

- POS Sale completion is the business trigger for POS cash collection.
- POS does not create a COD expectation.
- Payment records one immediate `POS_CASH_COLLECTED` fact in the same atomic
  completion commit that completes the POS sale, commits stock, records Finance
  outlet cash custody, and issues the receipt.
- The POS cash fact stores:
  - POS sale ID and public POS sale number;
  - selling outlet;
  - cashier actor identity;
  - optional customer name or phone when captured by POS;
  - POS sale snapshot reference;
  - expected amount from the POS sale total;
  - collected amount;
  - currency;
  - payment method `POS_CASH`;
  - completion idempotency key and correlation ID;
  - completion timestamp.
- POS collected amount must equal expected amount.
- Payment rejects POS cash collection if collected amount, currency, or POS sale
  snapshot basis differs from the POS completion request.
- If the POS completion transaction fails before commit, no POS cash fact is
  recorded.
- `POS_CASH_COLLECTED` remains voidable through the allowed same-day POS void
  window and becomes terminal only when that window closes without a void.
- An allowed POS same-day void appends a linked `POS_CASH_VOIDED` reversal fact.
- A POS void does not create a Refund-owned liability or collection code.

### Failed Delivery, Cancellation, and Unclaimable Closure

- Terminal failed delivery records a zero-collection payment fact tied to the
  failed delivery reason.
- A Delivery-owned `RETURN_PENDING` failed-attempt state records no terminal
  payment fact; the COD expectation remains `PENDING_COLLECTION` until terminal
  failed delivery commits.
- `PENDING_COLLECTION` is an active expectation, not a durable terminal payment
  fact.
- Cancellation before delivery collection creates no payment terminal state and
  records no customer payment fact.
- Cancellation before delivery collection retires the COD expectation through the
  authoritative Order cancellation outcome.
- Order cancellation is the authoritative outcome for cancelled orders before
  collection.
- `CLAIM_BLOCKED` creates no payment terminal state and records no customer
  payment fact; the COD expectation remains `PENDING_COLLECTION` while the order
  can still reopen or be cancelled.
- `UNCLAIMABLE` before delivery collection creates no payment terminal state and
  retires the COD expectation through the authoritative Order exception outcome.
- Payment does not create `CANCELLED`, `NOT_COLLECTED`, or equivalent terminal
  payment state for pre-collection cancellations or unclaimable closures.
- COD expectation retirement is durable and records:
  - retired order outcome, either `CUSTOMER_CANCELLED` or `UNCLAIMABLE`;
  - order outcome timestamp;
  - triggering actor identity or system policy identity;
  - actor role when a human actor triggered the outcome;
  - reason code;
  - reason note when required by Order;
  - idempotency key and correlation ID;
  - retirement timestamp.
- Order cancellation and `UNCLAIMABLE` closure commit Payment expectation
  retirement in the same atomic commit as the Order-owned outcome.
- If the Payment retirement participant rejects the mutation or the transaction
  fails before commit, the Order-owned cancellation or `UNCLAIMABLE` outcome is
  not committed.
- Retiring an expectation does not alter expected amount, currency, order total,
  order snapshot, collected amount, variance link, or any financial ledger.
- Queries by order ID must return the retired COD expectation with its retired
  order outcome and must not synthesize a Payment `CANCELLED`, `NOT_COLLECTED`,
  or `ZERO_COLLECTION` state.
- Because launch online orders are COD, cancellation, unclaimable closure, and
  terminal failed delivery do not create refund liabilities from prior customer
  collection.

### Cash Discrepancies

- Short and over collection by Delivery Agents follows the Finance-owned cash
  variance policy in [finance.md](finance.md).
- POS short and over collection are blocked before POS completion and do not use
  the COD variance policy.
- Approved short/over collection variance does not create a separate payment
  terminal state.
- Payment links to the approved Finance-owned variance record but does not own
  the variance record.
- An authorized and posted Finance-owned cash over-collection correction that
  confirms customer cash collected above expected COD creates a Refund-owned
  liability source event in [refund.md](refund.md), but does not rewrite the
  frozen order total or payment expectation.
- Payment records preserve expected amount, collected amount, variance link,
  actor, outlet, task, and order identity needed for audit and reconciliation.

## Permissions

Payment owns authorization decisions for Payment-owned commands and reads.
Related rows are shown here for context; rows enforced by another aggregate
remain with that aggregate. Identity supplies actor, account, grant, and
outlet-scope facts from [identity-auth.md](identity-auth.md).

| Capability | P-02 | P-03 | P-06 | P-10 |
| --- | --- | --- | --- | --- |
| Delivery execution pickup and COD | Own | - | - | Full |
| POS cash collection | - | Scoped | - | Full |
| POS cash void reversal | - | - | Scoped | Full |
| Agent cash handover | Own | - | Scoped receive | Full |
| Daily cash closing | - | - | Scoped | Full |
