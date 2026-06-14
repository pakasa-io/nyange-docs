# Order

**Intent**: Define order behavior for placement, lifecycle state, outlet
claiming, claim-block exceptions, COD fulfillment, and cancellation.

**Reader task**: Use this document to decide whether an online COD order can be
placed, claimed, claim-blocked, prepared, picked up, delivered, failed,
closed as unclaimable, or cancelled.

**Sources**: BI-02, BI-07, BI-09, BI-20, BI-21, BI-22, BI-23

**Related**:
[cart.md](cart.md) for customer cart state, quote readiness, catalog-change
acknowledgements, and checkout readiness;
[pos.md](pos.md) for separate walk-in outlet counter sales;
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
  basis, independent of the outlet that later claims the order.
- Subsequent catalog, price, outlet, tax, or delivery-fee changes do not alter
  historical orders.
- COD, receipts, refund liabilities, and reports derive from the frozen
  snapshot, not current rules.

**BI-07 — COD fulfillment proceeds before cash collection.**

- Online orders are COD.
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

- customer cancellation from `PENDING`, `CLAIM_BLOCKED`, `CLAIMED`, or
  `READY_FOR_PICKUP`;
- outlet claim cancellation from `CLAIMED` or `READY_FOR_PICKUP`;
- delivery task cancellation before pickup.

A cancellation missing any required field is a data integrity violation.

Claim-block closure to `UNCLAIMABLE` is a terminal Order-owned exception
outcome, not customer cancellation, but it must capture the same actor,
role, reason code, reason note, timestamp, and audit trail fields.

**BI-22 — Outlet disable is blocked by active orders.**

- An outlet cannot be disabled while it has an active claim, active reservation,
  active delivery task, or active order custody.
- Terminal order states for this rule are `DELIVERED`,
  `CUSTOMER_CANCELLED`, `DELIVERY_FAILED`, and `UNCLAIMABLE`.
- Unclaimed `PENDING` and `CLAIM_BLOCKED` orders do not block any outlet from
  being disabled because they have no claimed fulfilling outlet, active
  reservation, active delivery task, or active order custody.
- Disablement is allowed only after every order with an active or retained
  outlet association has reached a terminal state and no related custody,
  reservation, or cash handover blocker remains.

**BI-23 — Critical mutations are idempotent.**

- Critical one-time mutations require idempotency keyed by actor, endpoint, and
  client key.
- Covered mutations include order creation, order claim, claim-block marking,
  claim-block reopening, unclaimable closure, outlet claim cancellation,
  customer cancellation, ready-for-pickup marking, pickup, delivery completion,
  failed delivery, returned-empty reconciliation, COD collection reporting, and
  cash handover.
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
  cancellation, claim-block exception state, unclaimable closure, claim
  association, and immutable price snapshot.
- Order does not own walk-in POS sales, POS sale numbers, POS sale completion,
  POS cash collection, or POS same-day voids.
- Order has no fulfilling outlet while it is `PENDING`.
- Order has no fulfilling outlet while it is `CLAIM_BLOCKED`.
- Claiming creates exactly one claimed fulfilling outlet association.
- Claim-block marking removes the order from the claimable pending pool until
  Order reopens it to `PENDING`.
- Unclaimable closure terminates an order before pickup without stock,
  delivery, payment terminal state, receipt, refund, or fulfilling outlet
  effects.
- Outlet claim cancellation clears the fulfilling outlet association and returns
  the order to `PENDING`.
- Successful delivery and failed delivery retain the fulfilling outlet
  association for reporting and cash/custody traceability.
- Inventory owns stock availability, reservations, stock commitment, release,
  and returned-empty intake recognition.
- Delivery owns delivery task state, assignment, pickup, agent custody,
  doorstep field facts, failed delivery, and delivery completion.
- Payment owns durable COD expectation records, expected COD amount, collected
  cash fact, zero-collection fact, and payment outcome.
- Refund owns approved Finance-sourced over-collection refund liabilities and
  payout lifecycle.
- Refund liabilities do not reopen orders, change order state, or rewrite the
  frozen order total.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody,
  variance records, and receipt records.
- Notifications own fanout attempts triggered by order and delivery events.
- Order claims are whole-order only.
- At most one active claim may exist for an order.
- A claimed order requires an active reservation.
- Inventory owns append-only inventory ledger entries.
- Finance owns append-only cash and financial ledger entries.
- Pickup creates agent custody before the order becomes `OUT_FOR_DELIVERY`.
- Delivery coordinates delivery completion as the doorstep completion command.
- Order participates in delivery completion by moving the order from
  `OUT_FOR_DELIVERY` to `DELIVERED` and completing the active claim in the same
  atomic commit.
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
  -> CLAIMED
  -> READY_FOR_PICKUP
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PENDING -> CLAIM_BLOCKED
CLAIM_BLOCKED -> PENDING
CLAIM_BLOCKED -> UNCLAIMABLE

PENDING|CLAIM_BLOCKED|CLAIMED|READY_FOR_PICKUP -> CUSTOMER_CANCELLED
CLAIMED|READY_FOR_PICKUP -> PENDING   # outlet claim cancel before pickup
OUT_FOR_DELIVERY -> DELIVERY_FAILED
```

Terminal states: `DELIVERED`, `CUSTOMER_CANCELLED`, `DELIVERY_FAILED`,
`UNCLAIMABLE`.

| State | Meaning |
| --- | --- |
| `PENDING` | Order placement succeeded; no outlet has claimed the order and no stock is reserved. |
| `CLAIM_BLOCKED` | Order cannot currently be claimed because the pending-claim timeout expired or every eligible outlet has a current claim-blocking reason. No stock is reserved and no fulfilling outlet is associated. |
| `CLAIMED` | One active permitted outlet has claimed the whole order and inventory has reserved all requested stock. |
| `READY_FOR_PICKUP` | Claimed outlet has marked the whole order ready and a delivery task exists. |
| `OUT_FOR_DELIVERY` | Delivery agent has picked up the whole order and holds outgoing goods custody. |
| `DELIVERED` | Delivery succeeded; COD fact, stock commitment, claim completion, task state, order state, and returned-cylinder field facts committed together. |
| `CUSTOMER_CANCELLED` | Customer cancellation completed before pickup; the order is terminal and never returns to the pending pool. |
| `DELIVERY_FAILED` | Delivery failed after pickup; picked-up goods custody was resolved by physical return or approved custody exception, and zero collection was recorded. |
| `UNCLAIMABLE` | Order was closed before claim because launch rules could not produce an eligible claiming outlet within the configured claim-block policy. |

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
- Order placement creates one durable Payment-owned `PENDING_COLLECTION` COD
  expectation from the frozen order total in the same atomic commit.
- If the Payment-owned COD expectation cannot be created, order placement fails
  and no order is committed.
- Order placement starts the Order-owned pending-claim timeout.
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

### Claim Block (`PENDING` -> `CLAIM_BLOCKED`)

- Order owns claim-block detection, state transition, customer-visible status,
  audit record, and later resolution outcome.
- Inventory, Catalog, Delivery, Payment, Finance, and Notifications may supply
  facts or receive events, but they do not own claim-block state.
- Claim-block evaluation applies only while the order is `PENDING`.
- The claim-evaluation candidate set is the set of outlets that could claim the
  order under current service-area, online-fulfillment, outlet-status, and
  permission-scope facts.
- `CLAIM_BLOCKED` is entered when any condition is true:
  - the configured pending-claim timeout expires before any outlet claims the
    order;
  - the current claim-evaluation candidate set is empty;
  - every outlet in the current claim-evaluation candidate set has a current
    claim-blocking reason for the frozen order.
- The pending-claim timeout is required Order policy configuration for launch.
- A missing pending-claim timeout configuration is a launch-blocking policy gap.
- The timeout starts at order placement and restarts whenever an order is
  reopened from `CLAIM_BLOCKED` to `PENDING`.
- Claim-blocking reasons are controlled reason codes. Launch reason codes are:
  - `NO_RESERVABLE_STOCK`;
  - `OUTGOING_VENDOR_NOT_FULFILLABLE`;
  - `INCOMING_VENDOR_NOT_ACCEPTED`;
  - `PRODUCT_OR_SKU_NOT_FULFILLABLE`;
  - `NO_CURRENT_SERVICEABLE_OUTLET`;
  - `OUTLET_ONLINE_FULFILLMENT_DISABLED`;
  - `DELIVERY_AREA_NO_LONGER_SERVED`;
  - `COD_TERM_NOT_SUPPORTED`.
- A claim-blocking reason may come from a rejected claim attempt or from an
  authoritative system eligibility check.
- The claim-block record must capture:
  - order ID and public order number;
  - frozen order snapshot reference;
  - evaluated delivery-area facts;
  - candidate outlet set at evaluation time;
  - per-outlet claim-blocking reason code and source;
  - pending-claim timeout value and deadline, when timeout caused the block;
  - actor or system job that detected the block;
  - timestamp;
  - customer-visible status message category;
  - audit correlation ID.
- The transition is atomic and allowed only if the order is still `PENDING` and
  has no active claim.
- No stock is reserved, committed, released, or ledger-posted by claim-block
  marking.
- No delivery task, payment fact, receipt, refund liability, cash custody row,
  or fulfilling outlet association is created by claim-block marking.
- A `CLAIM_BLOCKED` order is removed from the pending-pool claim queue.
- Eligible outlets cannot claim a `CLAIM_BLOCKED` order until Order reopens it
  to `PENDING`.
- The customer can see the claim-blocked status and may cancel the order.
- Notifications attempts the configured claim-blocked customer notification.
- Failed or delayed claim-blocked notification fanout does not block the
  committed state transition.

### Claiming (`PENDING` -> `CLAIMED`)

- Any active online-fulfillment outlet that serves the delivery area and holds
  claim permission may claim an order from the pending pool.
- An order in `CLAIM_BLOCKED`, `CLAIMED`, `READY_FOR_PICKUP`,
  `OUT_FOR_DELIVERY`, or any terminal state cannot be claimed.
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

### Claim Block Resolution

- Claim-block resolution is an Order-owned transition from `CLAIM_BLOCKED`.
- `CLAIM_BLOCKED -> PENDING` reopens the same order to the pending pool.
- A scoped Outlet Manager may reopen only when their outlet is in the current
  claim-evaluation candidate set and Order verifies that outlet can currently
  attempt a claim under the frozen order terms.
- A Super Admin may reopen any claim-blocked order when Order verifies at least
  one current eligible claim path.
- Reopening requires a current Order-owned resolution record proving at least
  one active online-fulfillment outlet serving the delivery area can now attempt
  a claim under the frozen order terms.
- Reopening cannot change product, price, delivery fee, tax, discount, order
  total, expected COD, customer, delivery address, line contents, public order
  number, or original order-placement timestamp.
- Reopening clears the claim-block exception state, records the resolving actor
  or system basis, appends the audit record, and starts a new pending-claim
  timeout.
- Reopening does not reserve stock. Reservation still occurs only when an outlet
  claims the reopened `PENDING` order.
- `CLAIM_BLOCKED -> UNCLAIMABLE` closes the order when launch rules cannot
  produce an eligible claiming outlet and Super Admin confirms that correction
  is not available inside the launch process.
- `UNCLAIMABLE` closure requires:
  - Super Admin actor identity;
  - actor role;
  - reason code;
  - reason note;
  - timestamp;
  - unresolved candidate outlet evidence;
  - customer notification request;
  - audit correlation ID.
- `UNCLAIMABLE` closure is terminal.
- `UNCLAIMABLE` retires the Payment-owned COD expectation without creating a
  payment terminal state, refund liability, receipt, delivery task, inventory
  ledger entry, cash custody row, or fulfilling outlet association.
- If a customer still wants the goods after `UNCLAIMABLE`, the customer places a
  new order.

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

- Delivery is the coordinator for this transition.
- Order accepts the participant mutation only for the active claimed order tied
  to the picked-up delivery task.
- Completion requires `acknowledged_cash_collected=true`.
- COD derives from the persisted order total.
- Client requests cannot override COD.
- Returned-cylinder field facts are recorded when applicable.
- COD collection fact is recorded.
- Outgoing stock is committed.
- The active claim is completed.
- The delivery task is marked `DELIVERED`.
- The order is marked `DELIVERED`.
- Finance issues the immutable receipt in the same transaction.
- All completion effects commit in one transaction.
- If any participant mutation fails, the order remains `OUT_FOR_DELIVERY` and no
  participant mutation is committed.
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
  attribution, retires the COD expectation without creating a payment terminal
  state, and marks the order `CUSTOMER_CANCELLED`.
- Customer cancellation from `CLAIM_BLOCKED` records cancellation attribution
  retires the COD expectation without creating a payment terminal state, and
  marks the order `CUSTOMER_CANCELLED`; there is no reservation, active claim,
  delivery task, payment fact, receipt, refund liability, cash custody row, or
  fulfilling outlet association to release.
- A customer-cancelled order never returns to the pending pool.
- Unclaimable closure from `CLAIM_BLOCKED` records the required exception
  attribution, marks the order `UNCLAIMABLE`, and retires the COD expectation
  without creating a payment terminal state.
- Delivery failure after pickup resolves all picked-up outgoing goods custody by
  physical outlet return or approved custody exception, records a
  zero-collection payment fact, completes the claim, marks the task `FAILED`,
  and marks the order `DELIVERY_FAILED`.
- Failed delivery does not by itself make custody-exception goods available for
  new sale.

## Main Flow

1. Customer places an order from a checkout-ready cart. Order commits the order
   in `PENDING` and Payment creates the durable `PENDING_COLLECTION` COD
   expectation in the same atomic placement commit.
2. If no outlet claims within the pending-claim timeout, or every eligible
   outlet has a current claim-blocking reason, Order moves the order to
   `CLAIM_BLOCKED`.
3. A `CLAIM_BLOCKED` order is either reopened to `PENDING` after correction,
   cancelled by the customer, or closed as `UNCLAIMABLE`.
4. An active permitted outlet claims the whole order. The order moves
   `PENDING -> CLAIMED`; inventory reserves all requested stock atomically.
   Partial order-item claims are prohibited.
5. The claimed outlet marks the order ready. The order moves
   `CLAIMED -> READY_FOR_PICKUP`; fulfillment creates a `READY_FOR_PICKUP`
   delivery task and attempts ready-order notification fanout.
6. A delivery agent is assigned to the task. Assignment grants the agent
   order-scoped read access to the full delivery address while the assignment
   is active.
7. The delivery agent picks up the order. Pickup keeps the reserved stock
   unavailable, creates outgoing goods custody for the agent, moves the delivery
   task to `PICKED_UP`, and moves the order to `OUT_FOR_DELIVERY`.
8. The delivery agent completes delivery. Completion requires
   `acknowledged_cash_collected=true`, derives COD from the persisted order
   total, records returned-cylinder field facts when applicable, records the COD
   collection fact, commits outgoing stock, completes the claim, and marks the
   task and order `DELIVERED`.
9. Returned empties are reconciled separately from delivery completion. Outlet
   intake moves each returned-empty record into outlet empty stock and marks
   those custody rows reconciled.
10. COD cash is physically held by the Delivery Agent under the Finance-owned
   cash custody lifecycle until handover and reconciliation. A permitted
   handover receiver accepts counted cash, transfers custody to outlet cash, and
   records any approved Finance-owned variance.

## Out of Scope

- Automatic outlet allocation, outlet ranking, outlet-change chains, and
  customer acceptance of outlet changes.
- Delivery batching, route optimization, and scheduled delivery windows.
- A separate customer-visible order state after delivery.
- Order-state mutation workflows for doorstep conversion, delivery-time
  price-delta or refund negotiation, and returns after delivery.
- Partial order fulfillment, split orders, and multi-outlet fulfillment.
- Order-owned counter-sale workflows. Walk-in POS sales are a separate lifecycle
  owned by [pos.md](pos.md).

## Permissions

Trimmed access matrix rows relevant to orders. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-04 | P-05 | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Order placement | Full | - | - | - | - | - | - | - | Full |
| Order status tracking | Own | Own assigned | Scoped | - | Scoped | Scoped | Read assigned outlets | Read | Full |
| Outlet claiming | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Claim-block resolution | - | - | - | - | - | Scoped reopen with explicit permission | Read assigned outlets | - | Full |
| Unclaimable closure | - | - | - | - | - | - | - | - | Full |
| Ready-for-pickup marking | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Delivery assignment | - | - | - | - | Scoped | Scoped | - | - | Full |
| Delivery completion | - | Own assigned | - | - | - | - | - | - | Full |
| COD recording | - | Own assigned | - | - | - | - | - | - | Full |
| Cancellation | Own before pickup | - | - | - | - | Scoped outlet claim cancellation | - | - | Full |
