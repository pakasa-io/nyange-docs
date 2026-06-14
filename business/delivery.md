# Delivery

**Intent**: Define delivery task behavior, delivery assignment, pickup,
custody, doorstep field facts, COD collection acknowledgement, failed delivery,
delivery fees, and the field leg of refill exchange requests.

**Reader task**: Use this document to determine when a delivery task can move,
which facts are committed at the doorstep, how failed delivery is handled, and
which delivery facts feed inventory, payment, finance, and order state.

**Sources**: §6.3 Delivery Lifecycle, §6.9 Refill Exchange Request Lifecycle
field leg, §7.5 Express Delivery Fee, §7.7 Agent Cash Handling, §7.12 Failed
Delivery Fee Waiver

**Related**:
[order.md](order.md) for order lifecycle and COD fulfillment;
[inventory.md](inventory.md) for returned-cylinder intake and inventory
recognition;
[catalog.md](catalog.md) for product/refill price rules;
[payment.md](payment.md) for payment facts;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-08 — Delivery completion is all-or-nothing.**

Delivery completion is the single commit point for:

- delivery task terminal status;
- order terminal status;
- outgoing stock commitment;
- returned-cylinder field recording;
- cash collection recording;
- payment status.

```
delivery completion commits atomically:
  delivery_task_status
  order_status
  outgoing_stock_commitment
  returned_cylinder_field_recording
  cash_collection_recording
  payment_status
// all succeed or none do
```

**BI-10 — Incoming and outgoing cylinder sizes match on refill exchange.**

- A refill order exchanges a customer's empty cylinder for a filled cylinder of
  the same size.
- A customer who returns a 6kg empty cannot receive a 12kg filled as a refill.
- A customer who returns a 12kg empty cannot receive a 6kg filled as a refill.
- Size upgrade or downgrade is a new cylinder purchase, not a refill.
- A returned cylinder whose size differs from the expected size is a
  size-mismatch exception.
- The actual size must be recorded.
- A size mismatch blocks successful refill delivery and the whole delivery
  fails.

**BI-19 — Returned cylinders require intake before inventory recognition.**

- Returned cylinders recorded at the doorstep do not become outlet empty stock
  at delivery completion.
- Each returned cylinder becomes outlet empty stock only after outlet intake
  confirmation in [inventory.md](inventory.md).
- A failed intake creates an inventory exception and does not rewrite the
  delivered order.

## Boundary

- Delivery owns delivery task state, delivery assignment, pickup, agent custody,
  doorstep field facts, failed-delivery outcome, and delivery cash collection
  facts.
- Delivery does not own inventory intake recognition, refund payout, payment
  policy, receipt issuance, or outlet cash custody.
- Returned cylinders collected during delivery become inventory only after the
  intake leg in [inventory.md](inventory.md).

## Delivery Task Lifecycle

```
READY_FOR_PICKUP
  -> ASSIGNED
  -> PICKED_UP
  -> DELIVERED

READY_FOR_PICKUP|ASSIGNED -> CANCELLED
PICKED_UP -> FAILED
```

| State | Meaning |
| --- | --- |
| `READY_FOR_PICKUP` | Claimed outlet marked the order ready and a delivery task exists. |
| `ASSIGNED` | Delivery agent is assigned and can read the full delivery address while assignment is active. |
| `PICKED_UP` | Outlet handover and agent receipt are confirmed; agent has custody and the order is `OUT_FOR_DELIVERY`. |
| `DELIVERED` | Terminal successful task state committed with order `DELIVERED`, stock commitment, returned-cylinder field facts, and COD fact. |
| `FAILED` | Terminal failed task state after pickup; order becomes `DELIVERY_FAILED` and zero collection is recorded. |
| `CANCELLED` | Terminal task-only operational cancellation before pickup; the order remains `READY_FOR_PICKUP`. |

Terminal states: `CANCELLED`, `DELIVERED`, `FAILED`.

## Refill Exchange Field Leg

Each `REFILL_EXCHANGE` order line creates one exchange request. Delivery owns the
field leg from creation through `RETURN_RECORDED`. Delivery completion hands off
the returned-cylinder intake record to Inventory in `INTAKE_PENDING`.

```
PENDING
  -> IN_PROGRESS
  -> RETURN_RECORDED
  -> INTAKE_PENDING

CANCELLED
```

| State | Meaning |
| --- | --- |
| `PENDING` | Created when the order is placed; expected cylinder vendor/size recorded. |
| `IN_PROGRESS` | Order is out for delivery; agent is responsible for collecting the expected returned cylinder. |
| `RETURN_RECORDED` | Agent records returned cylinder vendor, size, and condition at the doorstep. |
| `INTAKE_PENDING` | Inventory-owned handoff state after successful delivery; cylinder awaits outlet intake confirmation. |
| `CANCELLED` | Order was cancelled before pickup; no cylinder exchange occurred. |

### Field Leg Rules

- One exchange request exists per refill order line.
- An order with three refill lines has three exchange requests.
- `RETURN_RECORDED` happens at the doorstep before delivery completion.
- The agent's recorded condition is the delivery field fact of record.
- `INTAKE_PENDING` is owned by Inventory and persists after delivery completion.
- The order can be `DELIVERED` while exchange requests are still
  `INTAKE_PENDING`.
- Inventory intake determines when returned cylinders become outlet empty stock.

## Delivery Assignment

### Task Creation

- A delivery task is created when the claimed outlet marks the order
  `READY_FOR_PICKUP`.
- Each order has at most one active delivery task.
- A task-only cancellation before pickup allows the claimed outlet to create or
  assign a replacement delivery task for the same `READY_FOR_PICKUP` order.
- The task is not a route plan and does not group multiple orders.

### Agent Eligibility

At launch, an eligible agent must:

- be an active delivery agent;
- have delivery scope for the task outlet;
- have no active picked-up task.

Formal shift schedules, live location, and route-distance optimization do not
add eligibility requirements, expand scope, or change assignment ranking.

### Default Assignment Ranking

```
rank agents by:
  1. queued_assignment_load  ASC
  2. least_recent_assignment ASC
  3. deterministic_tie_breaker
```

- A Dispatcher or Outlet Manager may assign or change the assigned delivery
  agent before pickup within outlet scope with a recorded reason.
- Operational risk alerts may be shown to permissioned Outlet Managers or Super
  Admins during manual assignment.
- Risk alerts do not block assignment or change assignment priority.

## Pickup Handover

Transition to `PICKED_UP` requires both:

1. An authorized outlet handover actor records handover of all outgoing items to
   the agent.
2. The agent confirms receipt of those items.

### Handover Rules

- Both confirmations must be recorded before the agent departs.
- A discrepancy between handover record and agent receipt confirmation is a
  custody exception requiring Outlet Manager resolution before departure.
- Missing agent receipt confirmation can be overridden only by an Outlet Manager
  or Super Admin with reason and audit.
- After `PICKED_UP`, outgoing stock remains in the agent's custody until
  delivery completion, failed-order return receipt at the outlet, or an approved
  custody exception covers the item.
- After `PICKED_UP`, the assigned delivery agent cannot be changed as a normal
  assignment action.
- A Super Admin may record a manual custody exception for an abnormal after-pickup
  custody issue, but the exception is a break-glass custody resolution, not a
  normal reassignment workflow.
- Pickup keeps reserved stock unavailable.
- Pickup creates outgoing goods custody for the agent.
- Pickup moves the order to `OUT_FOR_DELIVERY`.

## Delivery Completion

- Delivery completion requires `acknowledged_cash_collected=true`.
- COD derives from the persisted order total.
- The agent cannot modify expected COD.
- Client requests cannot override COD.
- Returned-cylinder field facts are recorded when applicable.
- COD collection fact is recorded.
- Outgoing stock is committed.
- The active order claim is completed.
- The delivery task is marked `DELIVERED`.
- The order is marked `DELIVERED`.
- All completion effects commit in one transaction.

## Failed Deliveries

- Delivery agents may mark a delivery `FAILED` after pickup.
- A controlled reason code and audit trail are required.
- An optional note may be added.
- Failed delivery returns all picked-up outgoing goods custody to outlet stock.
- Failed delivery records a zero-collection payment fact.
- Failed delivery completes the active order claim.
- Failed delivery marks the order `DELIVERY_FAILED`.
- The agent cannot reschedule the failed order.
- If the customer still wants the goods, the customer places a new order.

## Returned-Cylinder Field Facts

- For refill deliveries, the agent records each expected returned cylinder's
  vendor, size, and condition at the doorstep.
- Missing, damaged, unacceptable, unexpected-vendor, or size-mismatch returns
  are recorded as controlled field facts.
- A required returned cylinder that cannot be accepted for the placed refill
  order causes the whole delivery to fail.
- Returned cylinders do not become outlet inventory until outlet intake
  confirmation.

## Field Proof

- Delivery-agent field actions must be recorded live.
- Required live actions include pickup, cash collection acknowledgement,
  returned-cylinder recording, failed delivery, and cash handover.
- If the agent cannot record the action live, the action has not completed for
  business purposes.
- An authoritative timestamp is always recorded.
- GPS, photo, or note requirements are controlled by active field-proof policy.
- Field-proof policy must not allow partial delivery completion.

## Agent Cash Handling

Agents collect exact COD due at the doorstep.

### Short and Over Collection

```
if abs_variance_per_order <= 10,000 UGX
   AND shift_cumulative_variance <= 50,000 UGX  -> Outlet Manager approval
else                                             -> Super Admin approval
```

- Required approval must be recorded before delivery can be marked complete.
- Over-collection excess is recorded as a cash discrepancy and reconciled at
  close.

### Agent Rules

- Agents record cash collected.
- Agents cannot modify the expected amount.
- Expected amount is determined by the persisted order total.
- COD cash remains delivery-agent-owned until handover and reconciliation.
- A supervisor accepts counted cash, transfers custody to outlet cash, and
  records any approved variance.
- Cash reconciliation requires all cash collected to match expected cash or have
  an approved variance record for each order.
- Open cash discrepancy or unaccounted item blocks full shift close.

## Delivery Fee Rules

Delivery fee is a customer-visible charge component frozen into the order at
order placement.

### Fee Calculation

- The active global online delivery-fee rule must produce a fee before order
  placement.
- If no authoritative fee can be computed for the resolved delivery address and
  order contents, order placement is rejected as unpriceable.
- The delivery fee is computed from the resolved delivery address and online
  fee policy, not from the eventual claiming outlet.
- Delivery fee is part of the order total.
- Delivery fee is not folded into product, refill, or accessory prices.
- Later delivery-fee changes do not rewrite the placed order.

### Launch Zone Defaults

| Zone | Radius | Base delivery fee |
| --- | --- | --- |
| CORE | 0 km <= distance < 5 km | UGX 3,000 |
| STANDARD | 5 km <= distance < 10 km | UGX 5,000 |
| EXTENDED | 10 km <= distance < 15 km | UGX 8,000 |

- These fees apply unless an active global online delivery-fee rule overrides
  them.
- Delivery-fee guardrail is the smaller of 15% or UGX 2,000 from the current
  approved basis.
- Above-guardrail delivery-fee changes require Super Admin approval.

## Failed Delivery Fee Waiver

No delivery fee is charged on any failed delivery, regardless of failure reason.

### Covered Failure Scenarios

- Customer unavailable.
- Customer refused delivery.
- Safety issue.
- Missing required returned cylinder.
- Damaged or unacceptable returned cylinder.
- Any other terminal failure outcome.

### Payment Effects

- For COD orders, the waived fee is not collected from the customer by the
  agent.
- Failed COD deliveries keep a zero-collection payment fact tied to the failed
  delivery reason.
- Failed deliveries do not create prepaid refund liabilities because online
  orders are not prepaid.

## Permissions

Trimmed access matrix rows relevant to delivery. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-02 | P-04 | P-05 | P-06 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Delivery assignment | - | - | Scoped | Scoped | Full |
| Delivery execution pickup and COD | Own | - | - | - | Full |
| Agent cash handover | Own | - | - | Scoped receive | Full |
| Outlet handover confirmation | - | Scoped with explicit permission | - | Scoped with explicit permission | Full |
| Failed-order return receipt | - | Scoped with explicit permission | - | Scoped with explicit permission | Full |
| Returned-cylinder receipt | - | Scoped with explicit permission | - | Scoped with explicit permission | Full |

## Authorization Edge Cases

**E-02**: A Delivery Agent can see the customer's phone number only while an
order is assigned to them and active. They lose this access when the delivery
reaches a terminal state. Phone-number access is scoped to the active assignment
and is audit-sensitive under the active audit policy.

**E-08**: A Dispatcher can change the assigned delivery agent within their
outlet until pickup with a recorded reason. After pickup, no normal reassignment
is allowed. A Super Admin may record a manual custody exception for an abnormal
after-pickup custody issue, with reason, note, known goods/cash status, and audit.
