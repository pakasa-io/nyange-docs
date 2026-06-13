summary, not an authority document;

## Checkout

Online checkout creates a Cash on Delivery order. `POST /orders` creates a `PENDING` order from active catalog products and active
prices, snapshots product and price fields, computes line totals and order totals server-side, stores the structured delivery
address, allocates an immutable `ORD-%08d` public order number, and grants the customer order-scoped read/status/cancel permissions.

No stock is reserved at checkout. While an order is `PENDING`, permitted outlets may read the pending pool, but only with redacted
delivery-area details. Pending-pool reads must not expose full delivery address, recipient name, recipient phone, or delivery
instructions.

## Main Flow

1. Customer checks out and the order enters `PENDING`.
2. An active outlet claims the whole order. The order moves `PENDING -> CLAIMED`; inventory reserves all requested stock atomically.
   Partial order-item claims are prohibited.
3. The claimed outlet marks the order ready. The order moves `CLAIMED -> READY_FOR_PICKUP`; fulfillment creates a
   `READY_FOR_PICKUP` delivery task and ready-order notification fanout is attempted.
4. A delivery agent self-assigns or picks up the task. Assignment grants the agent order-read access for the full delivery address.
   Pickup consumes the reservation, creates full-cylinder custody for the agent, and moves the delivery task to `PICKED_UP` and the
   order to `OUT_FOR_DELIVERY`.
5. The delivery agent completes delivery. The workflow requires `acknowledged_cash_collected=true`, derives COD from the persisted
   order total, records the COD receipt, records swap and returned-empty custody, completes the order claim, and marks the task and
   order `DELIVERED`.
6. Returned empties are reconciled separately by task. The outlet reconciliation command moves returned-empty custody into outlet
   empty stock and marks those custody rows reconciled.
7. COD cash remains delivery-agent-owned until reporting. The delivery agent submits selected `RECORDED` COD receipts into a cash
   report, then a supervisor accepts counted cash, transfers custody to outlet cash, and records any over/short variance.

## State Flow

```text
PENDING
  -> CLAIMED
  -> READY_FOR_PICKUP
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PENDING|CLAIMED|READY_FOR_PICKUP -> CUSTOMER_CANCELLED
OUT_FOR_DELIVERY -> DELIVERY_FAILED
CLAIMED|READY_FOR_PICKUP -> PENDING   # outlet claim cancel before pickup
```

Order terminal states are `DELIVERED`, `CUSTOMER_CANCELLED`, and `DELIVERY_FAILED`.

## Alternate Paths

- Outlet claim cancellation before pickup releases the reservation, cancels any pre-pickup delivery task, cancels the active claim,
  clears the claimed outlet, and returns the order to `PENDING`.
- Customer cancellation before pickup is terminal. If the order is already claimed or ready, the workflow releases the reservation,
  cancels the active claim, cancels any pre-pickup delivery task, and marks the order `CUSTOMER_CANCELLED`. The order never returns
  to the pending pool.
- Delivery failure after pickup returns all picked-up full-cylinder custody to outlet stock, records no COD receipt, completes the
  claim, marks the task `FAILED`, and marks the order `DELIVERY_FAILED`. Re-delivery and returning failed orders to pending are
  post-MVP.

## Boundary Rules

- Order claims are whole-order only.
- At most one active claim may exist for an order.
- A claimed order requires an active reservation.
- Inventory and cash ledgers are append-only.
- Pickup consumes the reservation before the order becomes `OUT_FOR_DELIVERY`.
- Delivery completion records COD, swaps, task/order terminal state, and custody changes in one transaction.
- Client requests cannot override COD amount, currency, catalog-derived prices, authoritative delivery timestamps, partial payments,
  refunds, fees, taxes, or discounts in MVP.
- Delivery-agent profiles are operational eligibility gates for assignment and pickup; authorization remains fine-grained Cedar
  permissions.