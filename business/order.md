# Order

**Intent**: Define order behavior for placement, lifecycle state, outlet
claiming, COD fulfillment, and cancellation.

**Reader task**: Use this document to decide whether an online COD order can be
placed, claimed, prepared, picked up, delivered, failed, or cancelled.

**Sources**: BI-02, BI-07, BI-09, BI-20, BI-21, BI-22, BI-23

**Related**:
[cart.md](cart.md) for customer cart state, quote readiness, catalog-change
acknowledgements, and checkout readiness;
[catalog.md](catalog.md) for orderable products and priceability;
[inventory.md](inventory.md) for stock reservation, commitment, release, and
returned-empty intake recognition;
[delivery.md](delivery.md) for delivery task state, assignment, pickup,
custody, field facts, and failure handling;
[payment.md](payment.md) for expected COD, collected cash facts, and
zero-collection facts;
[refund.md](refund.md) for post-collection refund liabilities and payout
lifecycle;
[finance.md](finance.md) for cash handover, counted-cash acceptance, outlet
cash custody, variance records, and receipts;
[notifications.md](notifications.md) for notification fanout side effects;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-02 — Price snapshot is write-once.**

- Order placement freezes product, price, delivery-fee, tax, and total fields
  on the order.
- Order placement uses the global online pricing and address-based delivery-fee
  basis, independent of the outlet that later claims the order.
- Subsequent catalog, price, outlet, tax, or delivery-fee changes do not alter
  historical orders.
- COD, receipts, refund liabilities, and reports derive from the frozen
  snapshot, not current rules.

**BI-07 — No pre-payment gate before fulfillment.**

- Online orders are COD.
- No payment reference, provider verification, wallet, card, mobile-money, or
  prepaid settlement gate exists before fulfillment.
- Outlet claiming, ready-for-pickup marking, delivery assignment, and pickup
  may proceed without customer payment action.
- Cash is collected at the doorstep during delivery.

**BI-09 — All-or-nothing delivery; no partial completion.**

- Every line in an order delivers together or the whole order fails.
- Partial item claims, partial delivery completion, split orders, and
  multi-outlet fulfillment are prohibited.
- Failed delivery records a whole-order failed outcome.

**BI-20 — Order totals are immutable after placement.**

- Server-computed line totals and order totals are frozen when the order enters
  `PENDING`.
- Client-submitted values never set authoritative price, fee, tax, discount, or
  total fields.
- Later catalog, price, outlet, tax, stock, or delivery-fee changes do not
  rewrite the placed order total.

**BI-21 — Every cancellation must be fully attributed.**

Every cancellation record must capture:

- actor identity;
- actor role;
- reason code;
- reason note;
- timestamp.

Covered cancellations include:

- customer cancellation from `PENDING`, `CLAIMED`, or `READY_FOR_PICKUP`;
- outlet claim cancellation from `CLAIMED` or `READY_FOR_PICKUP`;
- delivery task cancellation before pickup.

A cancellation missing any required field is a data integrity violation.

**BI-22 — Outlet disable is blocked by active orders.**

- An outlet cannot be disabled while it has an active claim, active reservation,
  active delivery task, or active order custody.
- Terminal order states for this rule are `DELIVERED`,
  `CUSTOMER_CANCELLED`, and `DELIVERY_FAILED`.
- Unclaimed `PENDING` orders do not block any outlet from being disabled.
- Disablement is allowed only after every order with an active or retained
  outlet association has reached a terminal state and no related custody,
  reservation, or cash handover blocker remains.

**BI-23 — Critical mutations are idempotent.**

- Critical one-time mutations require idempotency keyed by actor, endpoint, and
  client key.
- Covered mutations include order creation, order claim, outlet claim
  cancellation, customer cancellation, ready-for-pickup marking, pickup,
  delivery completion, failed delivery, returned-empty reconciliation, COD
  collection reporting, and cash handover.
- Identical replays return the original result.
- Same-key/different-body replays are rejected.
- In-progress duplicates return a retryable response.
- Domain uniqueness rules remain permanent even after operational replay records
  expire.

### Cross-Aggregate Invariants

- **BI-01** is defined in [inventory.md](inventory.md). Available stock never
  goes below zero.
- **BI-06** is defined in [inventory.md](inventory.md). Reserved stock is
  unavailable.

## Boundary

- Order owns order placement, order state, customer-visible status,
  cancellation, claim association, and immutable price snapshot.
- Order has no fulfilling outlet while it is `PENDING`.
- Claiming creates exactly one claimed fulfilling outlet association.
- Outlet claim cancellation clears the fulfilling outlet association and returns
  the order to `PENDING`.
- Successful delivery and failed delivery retain the fulfilling outlet
  association for reporting and cash/custody traceability.
- Inventory owns stock availability, reservations, stock commitment, release,
  and returned-empty intake recognition.
- Delivery owns delivery task state, assignment, pickup, agent custody,
  doorstep field facts, failed delivery, and delivery completion.
- Payment owns expected COD amount, collected cash fact, zero-collection fact,
  and payment outcome.
- Refund owns approved post-collection refund liabilities and payout lifecycle.
- Refund liabilities do not reopen orders, change order state, or rewrite the
  frozen order total.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody,
  variance records, and receipt records.
- Notifications own fanout attempts triggered by order and delivery events.
- Order claims are whole-order only.
- At most one active claim may exist for an order.
- A claimed order requires an active reservation.
- Inventory and cash ledgers are append-only.
- Pickup creates agent custody before the order becomes `OUT_FOR_DELIVERY`.
- Delivery completion commits outgoing stock.
- Delivery completion records COD, returned-cylinder facts, task and order
  terminal state, and custody changes in one transaction.
- Client requests cannot override COD amount, currency, catalog-derived prices,
  authoritative delivery timestamps, partial payments, refunds, fees, taxes, or
  discounts.
- Delivery-agent profiles are operational eligibility gates for assignment and
  pickup; authorization remains controlled by explicit permissions.

## Order Lifecycle

```text
PENDING
  -> CLAIMED
  -> READY_FOR_PICKUP
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PENDING|CLAIMED|READY_FOR_PICKUP -> CUSTOMER_CANCELLED
CLAIMED|READY_FOR_PICKUP -> PENDING   # outlet claim cancel before pickup
OUT_FOR_DELIVERY -> DELIVERY_FAILED
```

Terminal states: `DELIVERED`, `CUSTOMER_CANCELLED`, `DELIVERY_FAILED`.

| State | Meaning |
| --- | --- |
| `PENDING` | Order placement succeeded; no outlet has claimed the order and no stock is reserved. |
| `CLAIMED` | One active permitted outlet has claimed the whole order and inventory has reserved all requested stock. |
| `READY_FOR_PICKUP` | Claimed outlet has marked the whole order ready and a delivery task exists. |
| `OUT_FOR_DELIVERY` | Delivery agent has picked up the whole order and holds outgoing goods custody. |
| `DELIVERED` | Delivery succeeded; COD fact, stock commitment, claim completion, task state, order state, and returned-cylinder field facts committed together. |
| `CUSTOMER_CANCELLED` | Customer cancellation completed before pickup; the order is terminal and never returns to the pending pool. |
| `DELIVERY_FAILED` | Delivery failed after pickup; picked-up goods custody was resolved by physical return or approved custody exception, and zero collection was recorded. |

## State Rules

### Order Placement (`POST /orders` -> `PENDING`)

- Order placement consumes a checkout-ready cart.
- Order placement requires an authenticated customer account.
- The delivery address must have resolved coordinates.
- An unresolved delivery address blocks order placement.
- Coarse serviceability by address must pass: at least one active
  online-fulfillment outlet serves the delivery area.
- Order placement avoids per-outlet SKU/vendor filtering before claim.
- Order placement rejects disabled products.
- Order placement rejects cart lines that are not orderable under pre-order
  orderability rules.
- Order placement rejects unpriceable catalog combinations.
- The server computes all line totals and order totals.
- Client-submitted line totals, fees, taxes, discounts, and order totals are
  ignored.
- The server uses global online catalog, price, delivery-fee, and tax rules for
  order placement.
- Order placement snapshots product, price, delivery-fee, tax, and total fields.
- The snapshot is write-once.
- Order placement stores the structured delivery address.
- Order placement allocates an immutable `ORD-%08d` public order number.
- Order placement grants the customer order-scoped read, status, and cancel
  permissions.
- No stock is reserved during cart activity or order placement.
- Stock reservation occurs only when an outlet claims the order.
- While an order is `PENDING`, only active online-fulfillment outlets that serve
  the delivery area and hold claim permission may read full pending-pool order
  detail needed to decide whether to claim and fulfill the order.
- Pending-pool reads expose the full structured delivery address, resolved
  coordinates, recipient name, recipient phone, delivery instructions, frozen
  order contents, frozen customer-visible totals, and serviceability facts.
- Pending-pool reads are address-serviceable outlet-scoped permissioned reads;
  no delivery-area redaction is applied to eligible claiming outlets.
- Outlets outside the delivery area scope cannot read the full pending-pool
  detail or attempt the claim.

### Claiming (`PENDING` -> `CLAIMED`)

- Any active online-fulfillment outlet that serves the delivery area and holds
  claim permission may claim an order from the pending pool.
- Claim is whole-order only.
- Partial item claims are prohibited.
- Claiming reserves all requested stock atomically.
- At most one active claim may exist per order.
- A claimed order requires an active reservation.
- The claim and reservation are inseparable.
- The claiming outlet must be able to fulfill every frozen product, vendor,
  quantity, delivery address, and COD term on the placed order.
- Claiming cannot change product price, delivery fee, tax, discount, order
  total, or expected COD.
- No automatic outlet allocation, ranking, or outlet-change chain is defined.

### Ready for Pickup (`CLAIMED` -> `READY_FOR_PICKUP`)

- Only the claimed outlet may mark the order ready.
- Marking ready creates a `READY_FOR_PICKUP` delivery task.
- Ready-order notification fanout is attempted.
- Failed or delayed notification fanout does not block the committed state
  transition.

### Out for Delivery (`READY_FOR_PICKUP` -> `OUT_FOR_DELIVERY`)

- A delivery agent is assigned to the delivery task.
- Assignment grants the agent order-scoped read access to the full delivery
  address for the duration of the assignment.
- Pickup keeps reserved stock unavailable.
- Pickup creates outgoing goods custody for the assigned agent.
- Pickup moves the delivery task to `PICKED_UP`.
- Pickup moves the order to `OUT_FOR_DELIVERY`.

### Delivery Completion (`OUT_FOR_DELIVERY` -> `DELIVERED`)

- Completion requires `acknowledged_cash_collected=true`.
- COD derives from the persisted order total.
- Client requests cannot override COD.
- Returned-cylinder field facts are recorded when applicable.
- COD collection fact is recorded.
- Outgoing stock is committed.
- The active claim is completed.
- The delivery task is marked `DELIVERED`.
- The order is marked `DELIVERED`.
- All completion effects commit in one transaction.
- `DELIVERED` is the terminal successful order state.
- No later order state follows `DELIVERED`.

### Alternate Paths

- Outlet claim cancellation before pickup releases the reservation, cancels any
  pre-pickup delivery task, cancels the active claim, clears the claimed outlet,
  and returns the order to `PENDING`.
- Standalone delivery task cancellation before pickup is task-only operational
  cancellation.
- Task-only cancellation keeps the order in `READY_FOR_PICKUP`, keeps the active
  claim and reservation, and allows the claimed outlet to create or assign a
  replacement delivery task.
- If the intended outcome is to release the reservation or return the order to
  the pending pool, the outlet claim cancellation path must be used instead.
- Customer cancellation before pickup is terminal.
- Customer cancellation releases any active reservation, cancels the active
  claim, cancels any pre-pickup delivery task, records required cancellation
  attribution, and marks the order `CUSTOMER_CANCELLED`.
- A customer-cancelled order never returns to the pending pool.
- Delivery failure after pickup resolves all picked-up outgoing goods custody by
  physical outlet return or approved custody exception, records a
  zero-collection payment fact, completes the claim, marks the task `FAILED`,
  and marks the order `DELIVERY_FAILED`.
- Failed delivery does not by itself make custody-exception goods available for
  new sale.

## Main Flow

1. Customer places an order from a checkout-ready cart and the order enters
   `PENDING`.
2. An active permitted outlet claims the whole order. The order moves
   `PENDING -> CLAIMED`; inventory reserves all requested stock atomically.
   Partial order-item claims are prohibited.
3. The claimed outlet marks the order ready. The order moves
   `CLAIMED -> READY_FOR_PICKUP`; fulfillment creates a `READY_FOR_PICKUP`
   delivery task and attempts ready-order notification fanout.
4. A delivery agent is assigned to the task. Assignment grants the agent
   order-scoped read access to the full delivery address while the assignment
   is active.
5. The delivery agent picks up the order. Pickup keeps the reserved stock
   unavailable, creates outgoing goods custody for the agent, moves the delivery
   task to `PICKED_UP`, and moves the order to `OUT_FOR_DELIVERY`.
6. The delivery agent completes delivery. Completion requires
   `acknowledged_cash_collected=true`, derives COD from the persisted order
   total, records returned-cylinder field facts when applicable, records the COD
   collection fact, commits outgoing stock, completes the claim, and marks the
   task and order `DELIVERED`.
7. Returned empties are reconciled separately from delivery completion. Outlet
   intake moves each returned-empty record into outlet empty stock and marks
   those custody rows reconciled.
8. COD cash remains delivery-agent-owned until handover and reconciliation. A
   supervisor accepts counted cash, transfers custody to outlet cash, and
   records any approved variance.

## Out of Scope

- Automatic outlet allocation, outlet ranking, outlet-change chains, and
  customer acceptance of outlet changes.
- Delivery batching, route optimization, and scheduled delivery windows.
- External prepayment, mobile money, card, wallet, or provider reference
  workflows.
- A separate customer-visible order state after delivery.
- Doorstep code fallback and evidence-retention detail.
- Order-state mutation workflows for doorstep conversion, delivery-time
  price-delta or refund negotiation, and returns after delivery.
- Partial order fulfillment, split orders, and multi-outlet fulfillment.
- Counter-sale workflows.

## Permissions

Trimmed access matrix rows relevant to orders. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-04 | P-05 | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Order placement | Full | - | - | - | - | - | - | - | Full |
| Order status tracking | Own | Own assigned | Scoped | - | Scoped | Scoped | Read assigned outlets | Read | Full |
| Outlet claiming | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Ready-for-pickup marking | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Delivery assignment | - | - | - | - | Scoped | Scoped | - | - | Full |
| Delivery completion | - | Own assigned | - | - | - | - | - | - | Full |
| COD recording | - | Own assigned | - | - | - | - | - | - | Full |
| Cancellation | Own before pickup | - | - | - | - | Scoped outlet claim cancellation | - | - | Full |
