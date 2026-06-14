# Order

**Intent**: Define order behavior for placement, lifecycle state, fulfillment
outlet assignment, COD fulfillment, and cancellation.

**Reader task**: Use this document to decide whether an online COD order can be
placed, accepted for fulfillment, prepared, picked up, delivered, failed, or
cancelled.

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
[refund.md](refund.md) for Finance-sourced over-collection refund liabilities
and payout lifecycle;
[finance.md](finance.md) for cash handover, counted-cash acceptance, outlet
cash custody, variance records, and receipts;
[notifications.md](notifications.md) for notification fanout side effects;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-02 — Price snapshot is write-once.**

- Order placement freezes product, price, delivery-fee, tax, and total fields
  on the order.
- Order placement uses the global online pricing and address-based delivery-fee
  basis, independent of the assigned fulfilling outlet.
- Subsequent catalog, price, outlet, tax, or delivery-fee changes do not alter
  historical orders.
- COD, receipts, refund liabilities, and reports derive from the frozen
  snapshot, not current rules.

**BI-07 — COD fulfillment proceeds before cash collection.**

- Online orders are COD.
- Fulfillment acceptance, ready-for-pickup marking, delivery assignment, and
  pickup may proceed without customer payment action.
- Cash is collected at the doorstep during delivery.

**BI-09 — All-or-nothing delivery; no partial completion.**

- Every line in an order delivers together or the whole order fails.
- Partial item acceptance, partial delivery completion, split orders, and
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

- customer cancellation from `PENDING`, `ACCEPTED`, or `READY_FOR_PICKUP`;
- fulfillment acceptance cancellation from `ACCEPTED` or `READY_FOR_PICKUP`;
- delivery task cancellation before pickup.

A cancellation missing any required field is a data integrity violation.

**BI-22 — Outlet disable is blocked by active orders.**

- An outlet cannot be disabled while it has an assigned pending order, active
  reservation, active delivery task, or active order custody.
- Terminal order states for this rule are `DELIVERED`,
  `CUSTOMER_CANCELLED`, and `DELIVERY_FAILED`.
- `PENDING` orders assigned to the outlet block disablement until cancelled,
  reassigned by Super Admin, or accepted and completed through a terminal state.
- Disablement is allowed only after every order with an active or retained outlet
  association has reached a terminal state and no related custody, reservation,
  or cash handover blocker remains.

**BI-23 — Critical mutations are idempotent.**

- Critical one-time mutations require idempotency keyed by actor, endpoint, and
  client key.
- Covered mutations include order creation, fulfillment acceptance, fulfillment
  acceptance cancellation, customer cancellation, ready-for-pickup marking, pickup,
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
  cancellation, fulfillment outlet assignment, fulfillment acceptance, and
  immutable price snapshot.
- Order has exactly one assigned fulfilling outlet while it is `PENDING`.
- Fulfillment acceptance by the assigned outlet creates the active accepted
  outlet association.
- Fulfillment acceptance cancellation releases the reservation and returns the
  order to `PENDING` with the same assigned fulfilling outlet unless a Super
  Admin records a manual assignment correction.
- Successful delivery and failed delivery retain the fulfilling outlet
  association for reporting and cash/custody traceability.
- Inventory owns stock availability, reservations, stock commitment, release,
  and returned-empty intake recognition.
- Delivery owns delivery task state, assignment, pickup, agent custody,
  doorstep field facts, failed delivery, and delivery completion.
- Payment owns expected COD amount, collected cash fact, zero-collection fact,
  and payment outcome.
- Refund owns approved Finance-sourced over-collection refund liabilities and
  payout lifecycle.
- Refund liabilities do not reopen orders, change order state, or rewrite the
  frozen order total.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody,
  variance records, and receipt records.
- Notifications own fanout attempts triggered by order and delivery events.
- Fulfillment acceptance is whole-order only.
- At most one active fulfillment acceptance may exist for an order.
- An accepted order requires an active reservation.
- Inventory owns append-only inventory ledger entries.
- Finance owns append-only cash and financial ledger entries.
- Pickup creates agent custody before the order becomes `OUT_FOR_DELIVERY`.
- Delivery coordinates delivery completion as the doorstep completion command.
- Order participates in delivery completion by moving the order from
  `OUT_FOR_DELIVERY` to `DELIVERED` and completing the active fulfillment
  acceptance in the same atomic commit.
- Delivery completion records COD, returned-cylinder facts, task and order
  terminal state, outgoing stock commitment, payment status, receipt issuance,
  and custody changes in one transaction.
- Client requests cannot override COD amount, currency, catalog-derived prices,
  authoritative delivery timestamps, partial payments, refunds, fees, taxes, or
  discounts.
- Delivery-agent profiles are operational eligibility gates for assignment and
  pickup; authorization remains controlled by explicit permissions.

## Order Lifecycle

```text
PENDING
  -> ACCEPTED
  -> READY_FOR_PICKUP
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PENDING|ACCEPTED|READY_FOR_PICKUP -> CUSTOMER_CANCELLED
ACCEPTED|READY_FOR_PICKUP -> PENDING   # acceptance cancellation before pickup
OUT_FOR_DELIVERY -> DELIVERY_FAILED
```

Terminal states: `DELIVERED`, `CUSTOMER_CANCELLED`, `DELIVERY_FAILED`.

| State | Meaning |
| --- | --- |
| `PENDING` | Order placement succeeded; one fulfilling outlet is assigned and no stock is reserved. |
| `ACCEPTED` | The assigned fulfilling outlet accepted the whole order and inventory reserved all requested stock. |
| `READY_FOR_PICKUP` | Accepted outlet has marked the whole order ready and a delivery task exists. |
| `OUT_FOR_DELIVERY` | Delivery agent has picked up the whole order and holds outgoing goods custody. |
| `DELIVERED` | Delivery succeeded; COD fact, stock commitment, fulfillment acceptance completion, task state, order state, and returned-cylinder field facts committed together. |
| `CUSTOMER_CANCELLED` | Customer cancellation completed before pickup; the order is terminal. |
| `DELIVERY_FAILED` | Delivery failed after pickup; picked-up goods custody was resolved by physical return or approved custody exception, and zero collection was recorded. |

## State Rules

### Order Placement (`POST /orders` -> `PENDING`)

- Order placement consumes a checkout-ready cart.
- Order placement requires an authenticated customer account.
- The delivery address must have resolved coordinates.
- An unresolved delivery address blocks order placement.
- Coarse serviceability by address must resolve one active online-fulfillment
  outlet assigned to the delivery area.
- Order placement avoids per-outlet SKU/vendor stock filtering before
  fulfillment acceptance.
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
- Order placement stores the assigned fulfilling outlet.
- Order placement allocates an immutable `ORD-%08d` public order number.
- Order placement grants the customer order-scoped read, status, and cancel
  permissions.
- No stock is reserved during cart activity or order placement.
- Stock reservation occurs only when the assigned outlet accepts the order.
- While an order is `PENDING`, only the assigned fulfilling outlet and Super
  Admin may read full pending order detail needed to decide whether to accept
  and fulfill the order.
- Pending order reads expose the full structured delivery address, resolved
  coordinates, recipient name, recipient phone, delivery instructions, frozen
  order contents, frozen customer-visible totals, and serviceability facts.
- Outlets that are not assigned to the order cannot read the full pending order
  detail or accept the order.

### Fulfillment Acceptance (`PENDING` -> `ACCEPTED`)

- Only the assigned fulfilling outlet may accept a pending order.
- Fulfillment acceptance is whole-order only.
- Partial item acceptance is prohibited.
- Fulfillment acceptance reserves all requested stock atomically.
- At most one active fulfillment acceptance may exist per order.
- An accepted order requires an active reservation.
- The acceptance and reservation are inseparable.
- The assigned fulfilling outlet must be able to fulfill every frozen product,
  vendor, quantity, delivery address, and COD term on the placed order.
- Fulfillment acceptance cannot change product price, delivery fee, tax,
  discount, order total, or expected COD.

### Ready for Pickup (`ACCEPTED` -> `READY_FOR_PICKUP`)

- Only the accepted fulfilling outlet may mark the order ready.
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

- Delivery is the coordinator for this transition.
- Order accepts the participant mutation only for the active accepted order tied
  to the picked-up delivery task.
- Completion requires `acknowledged_cash_collected=true`.
- COD derives from the persisted order total.
- Client requests cannot override COD.
- Returned-cylinder field facts are recorded when applicable.
- COD collection fact is recorded.
- Outgoing stock is committed.
- The active fulfillment acceptance is completed.
- The delivery task is marked `DELIVERED`.
- The order is marked `DELIVERED`.
- Finance issues the immutable receipt in the same transaction.
- All completion effects commit in one transaction.
- If any participant mutation fails, the order remains `OUT_FOR_DELIVERY` and no
  participant mutation is committed.
- `DELIVERED` is the terminal successful order state.
- No later order state follows `DELIVERED`.

### Alternate Paths

- Fulfillment acceptance cancellation before pickup releases the reservation,
  cancels any pre-pickup delivery task, cancels the active acceptance, and
  returns the order to `PENDING`.
- Standalone delivery task cancellation before pickup is task-only operational
  cancellation.
- Task-only cancellation keeps the order in `READY_FOR_PICKUP`, keeps the active
  acceptance and reservation, and allows the accepted outlet to create or assign
  a replacement delivery task.
- If the intended outcome is to release the reservation, the fulfillment
  acceptance cancellation path must be used instead.
- Customer cancellation before pickup is terminal.
- Customer cancellation releases any active reservation, cancels the active
  acceptance, cancels any pre-pickup delivery task, records required
  cancellation attribution, and marks the order `CUSTOMER_CANCELLED`.
- Delivery failure after pickup resolves all picked-up outgoing goods custody by
  physical outlet return or approved custody exception, records a
  zero-collection payment fact, completes the fulfillment acceptance, marks the
  task `FAILED`, and marks the order `DELIVERY_FAILED`.
- Failed delivery does not by itself make custody-exception goods available for
  new sale.

## Main Flow

1. Customer places an order from a checkout-ready cart. Order placement assigns
   one fulfilling outlet and the order enters `PENDING`.
2. The assigned fulfilling outlet accepts the whole order. The order moves
   `PENDING -> ACCEPTED`; inventory reserves all requested stock atomically.
   Partial order-item acceptance is prohibited.
3. The accepted outlet marks the order ready. The order moves
   `ACCEPTED -> READY_FOR_PICKUP`; fulfillment creates a `READY_FOR_PICKUP`
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
   collection fact, commits outgoing stock, completes the fulfillment
   acceptance, and marks the task and order `DELIVERED`.
7. Returned empties are reconciled separately from delivery completion. Outlet
   intake moves each returned-empty record into outlet empty stock and marks
   those custody rows reconciled.
8. COD cash is physically held by the Delivery Agent under the Finance-owned
   cash custody lifecycle until handover and reconciliation. A permitted
   handover receiver accepts counted cash, transfers custody to outlet cash, and
   records any approved Finance-owned variance.

## Out of Scope

- Delivery batching, route optimization, and scheduled delivery windows.
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
| Fulfillment acceptance | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Ready-for-pickup marking | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Delivery assignment | - | - | - | - | Scoped | Scoped | - | - | Full |
| Delivery completion | - | Own assigned | - | - | - | - | - | - | Full |
| COD recording | - | Own assigned | - | - | - | - | - | - | Full |
| Cancellation | Own before pickup | - | - | - | - | Scoped acceptance cancellation | - | - | Full |
