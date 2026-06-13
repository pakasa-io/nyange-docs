# MVP Order Checkout Fulfillment Flow

**Intent**: Define a simple MVP checkout-to-delivery flow for online COD orders
without importing the full launch reassignment, batching, PIN fallback, and
financial-closure model.

**Status**: Scratch proposal, not an authority document.

**Sources to borrow from**:
[order](../business/order.md), [delivery](../business/delivery.md),
[inventory](../business/inventory.md), [payment](../business/payment.md), and
[catalog](../business/catalog.md).

## Simplification Principle

Borrow the current flow's correctness rules, not its full orchestration.

The MVP keeps one customer order, one claiming outlet, one delivery task, one
delivery agent, COD at doorstep, and whole-order outcomes. It deliberately
collapses the current launch states for outlet assignment, outlet acceptance,
picking, delivery assignment, delivery execution, payment collection, and
financial closure into a smaller customer-order lifecycle.

## Current Flow Rules Worth Borrowing

| Borrowed rule | MVP use |
| --- | --- |
| Price snapshot is write-once. | Checkout freezes product, price, delivery-fee, tax, and total fields used by COD. |
| Order totals are immutable after placement. | Later catalog, price, outlet, or delivery-fee changes do not rewrite the order. |
| Online orders are COD at launch. | No payment reference, provider verification, wallet, card, or mobile-money gate exists before fulfillment. |
| Delivery confirmation is all-or-nothing. | COD collection, delivery status, order terminal status, outgoing stock commitment, and returned-cylinder field facts commit together. |
| Fulfilled orders have no partial completion state. | All lines deliver together or the whole order fails. |
| Available stock never goes below zero. | Claiming an order reserves every requested stock item atomically. |
| Reserved stock is unavailable. | Claimed stock cannot be sold, claimed, or transferred by another workflow. |
| Failed delivery records zero collection. | Failed orders do not collect COD and do not create prepaid refund liability. |
| Returned cylinders require intake before inventory recognition. | Returned empties are recorded at delivery but become outlet empty stock only after outlet reconciliation. |
| Critical mutations are idempotent. | Order creation, claim, cancellation, pickup, delivery completion, failed delivery, returned-empty reconciliation, and COD reporting require replay-safe command handling. |

## Deliberate MVP Collapses

| Current flow detail | MVP collapse |
| --- | --- |
| `PLACED -> ASSIGNED_TO_OUTLET -> AWAITING_OUTLET_ACCEPTANCE -> ACCEPTED_BY_OUTLET -> PICKING` | `PENDING -> CLAIMED` |
| Outlet allocation ranking and cascade/reassignment | A permitted outlet claims from a pending pool. No automatic cascade. |
| Batched delivery runs and delivery windows | One order creates one delivery task. Route batching is post-MVP. |
| Full delivery task lifecycle through `ARRIVED` | The order-visible flow uses `OUT_FOR_DELIVERY`; arrival can be an internal delivery event. |
| `DELIVERED -> COMPLETED` financial closure | `DELIVERED` is the order terminal state for MVP. Cash and empty-cylinder reconciliation continue as back-office tasks. |
| Doorstep conversion and price-delta exceptions | Post-MVP unless explicitly needed for launch refill viability. |
| PIN fallback, evidence retention, and shift reconciliation detail | Keep only the minimal commit rule and audit hooks in this flow. |

## Open Compatibility Gaps

- The current authority flow assigns an outlet and reserves inventory at order
  placement. This MVP proposal delays reservation until an outlet claims the
  order. That is simpler operationally but changes stock-risk behavior.
- The current authority flow computes final pricing after outlet allocation.
  If outlet-specific prices or outlet-specific delivery fees remain required,
  this simplified flow must either select the outlet before placement or limit
  MVP checkout to globally priceable items and fees.
- The current authority flow distinguishes customer-facing delivery completion
  from internal financial closure. This proposal keeps that distinction as
  back-office reconciliation, not as an order state.

## Checkout

Online checkout creates a Cash on Delivery order in `PENDING`.

`POST /orders`:

- requires an authenticated customer account;
- requires a resolved delivery address with usable coordinates;
- rejects disabled products, unavailable cart lines, and unpriceable catalog
  combinations;
- computes line totals and order totals server-side;
- snapshots product, price, delivery-fee, tax, and total fields;
- stores the structured delivery address;
- allocates an immutable `ORD-%08d` public order number;
- grants the customer order-scoped read, status, and cancel permissions.

No stock is reserved at checkout in this proposal. While an order is `PENDING`,
permitted outlets may read the pending pool only with redacted delivery-area
details.

Pending-pool reads must not expose:

- full delivery address;
- recipient name;
- recipient phone;
- delivery instructions.

## Main Flow

1. Customer checks out and the order enters `PENDING`.
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
   unavailable, creates full-cylinder custody for the agent, moves the delivery
   task to `PICKED_UP`, and moves the order to `OUT_FOR_DELIVERY`.
6. The delivery agent completes delivery. Completion requires
   `acknowledged_cash_collected=true`, derives COD from the persisted order
   total, records returned-cylinder field facts when applicable, records the
   COD collection fact, commits outgoing stock, completes the claim, and marks
   the task and order `DELIVERED`.
7. Returned empties are reconciled separately from delivery completion. Outlet
   intake moves each returned-empty record into outlet empty stock and marks
   those custody rows reconciled.
8. COD cash remains delivery-agent-owned until handover and reconciliation. A
   supervisor accepts counted cash, transfers custody to outlet cash, and
   records any approved variance.

## State Flow

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

Order terminal states are `DELIVERED`, `CUSTOMER_CANCELLED`, and
`DELIVERY_FAILED`.

## Alternate Paths

- Outlet claim cancellation before pickup releases the reservation, cancels any
  pre-pickup delivery task, cancels the active claim, clears the claimed
  outlet, and returns the order to `PENDING`.
- Customer cancellation before pickup is terminal. If the order is already
  claimed or ready, the workflow releases the reservation, cancels the active
  claim, cancels any pre-pickup delivery task, records the required
  cancellation attribution, and marks the order `CUSTOMER_CANCELLED`. The order
  never returns to the pending pool.
- Delivery failure after pickup returns all picked-up full-cylinder custody to
  outlet stock, records a zero-collection payment fact, completes the claim,
  marks the task `FAILED`, and marks the order `DELIVERY_FAILED`.
  Re-delivery and returning failed orders to pending are post-MVP.

## Boundary Rules

- Order owns order placement, order state, customer-visible status,
  cancellation, claim association, and immutable price snapshot.
- Inventory owns stock availability, reservations, stock commitment, release,
  and returned-empty intake recognition.
- Delivery owns delivery task state, assignment, pickup, agent custody,
  doorstep field facts, failed delivery, and delivery completion.
- Payment owns expected COD amount, collected cash fact, zero-collection fact,
  and payment outcome.
- Finance owns cash handover, counted-cash acceptance, outlet cash custody,
  variance records, and receipt or closure records if added later.
- Order claims are whole-order only.
- At most one active claim may exist for an order.
- A claimed order requires an active reservation.
- Inventory and cash ledgers are append-only.
- Pickup creates agent custody before the order becomes `OUT_FOR_DELIVERY`.
- Delivery completion commits outgoing stock.
- Delivery completion records COD, returned-cylinder facts, task/order terminal
  state, and custody changes in one transaction.
- Client requests cannot override COD amount, currency, catalog-derived prices,
  authoritative delivery timestamps, partial payments, refunds, fees, taxes, or
  discounts in MVP.
- Delivery-agent profiles are operational eligibility gates for assignment and
  pickup; authorization remains fine-grained Cedar permissions.

## Out of Scope for This MVP Flow

- automatic outlet allocation, ranking, cascade, and customer reassignment
  acceptance;
- delivery batching, route optimization, and scheduled delivery windows;
- external prepayment, mobile money, card, wallet, or provider reference
  workflows;
- customer-visible `COMPLETED` financial-closure state;
- delivery PIN fallback and evidence-retention detail;
- doorstep conversion, price-delta, refund, and post-delivery return workflows;
- partial order fulfillment, split orders, and multi-outlet fulfillment.
