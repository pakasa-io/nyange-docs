# Delivery

**Intent**: Define delivery lifecycle behavior, delivery assignment, custody,
PIN confirmation, field evidence, agent cash handling, delivery fees, and the
field leg of refill exchange requests.

**Reader task**: Use this document to determine when delivery state can move,
which facts are committed at the doorstep, how exceptions are handled, and which
delivery facts block financial closure.

**Sources**: §6.3 Delivery Lifecycle, §6.9 Refill Exchange Request Lifecycle
field leg, §7.5 Express Delivery Fee, §7.7 Agent Cash Handling, §7.12 Failed
Delivery Fee Waiver, F-03, F-06, F-07

**Related**:
[inventory.md](inventory.md) for returned-cylinder intake and inventory
recognition; [order.md](order.md) for order lifecycle and online delivery flow;
[catalog.md](catalog.md) for product/refill price rules;
[finance.md](finance.md) for forced financial closure;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-08 — Delivery confirmation is all-or-nothing.**

Customer PIN confirmation is the single commit point for:

- delivery status;
- order status;
- outgoing stock commitment;
- returned-cylinder field recording;
- cash collection recording;
- payment status.

```
PIN confirmation commits atomically:
  delivery_status
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
- A customer who brings a 6kg empty cannot receive a 12kg filled as a refill.
- A customer who brings a 12kg empty cannot receive a 6kg filled as a refill.
- Size upgrade or downgrade is a new cylinder purchase, not a refill.
- A returned cylinder whose size differs from the expected size is a
  size-mismatch exception.
- The actual size must be recorded.
- The original refill may continue only through an approved full-order
  adjustment that keeps final incoming and outgoing sizes matched and handles
  any COD/refund delta before PIN confirmation.
- Otherwise, the order fails or converts under the new-cylinder path.

**BI-19 — No order can complete financially until all expected returned
cylinders are accounted for.**

- For refill orders, financial closure requires every expected returned cylinder
  to be accounted for through returned-cylinder intake confirmation or approved
  exception.
- Delivery can be confirmed by the customer before intake is complete.
- Financial closure waits for intake or approved exception.

## Boundary

- Delivery owns delivery task state, delivery assignment, agent custody,
  doorstep field facts, PIN confirmation, failed-delivery outcome, and delivery
  cash collection facts.
- Delivery does not own inventory intake recognition, refund payout, payment
  policy, or financial closure.
- Returned cylinders collected during delivery become inventory only after the
  intake leg in [inventory.md](inventory.md).

## Delivery Lifecycle

```
PENDING
  ├─► CANCELLED
  └─► ASSIGNED
        ├─► CANCELLED
        └─► PICKED_UP
              └─► OUT_FOR_DELIVERY
                    └─► ARRIVED
                          ├─► DELIVERED
                          └─► FAILED
                                └─► RETURNED_TO_OUTLET
```

| State | Meaning |
| --- | --- |
| `PENDING` | Delivery task exists but agent is not assigned. |
| `ASSIGNED` | Delivery agent assigned. |
| `PICKED_UP` | Outlet handover and agent receipt are confirmed. |
| `OUT_FOR_DELIVERY` | Agent has custody and is en route. |
| `ARRIVED` | Agent is at customer location. |
| `DELIVERED` | PIN confirmed; delivery leg terminal; order may still await financial closure. |
| `FAILED` | Delivery attempt failed after pickup or at doorstep. |
| `RETURNED_TO_OUTLET` | Failed-order goods returned to outlet; terminal. |
| `CANCELLED` | Terminal before pickup/custody. |

Terminal states: `CANCELLED`, `DELIVERED`, `RETURNED_TO_OUTLET`.

## Refill Exchange Field Leg

Each `REFILL_EXCHANGE` order line creates one exchange request. Delivery owns the
field leg from creation through `RETURN_RECORDED`. Inventory owns
`INTAKE_PENDING` onward.

```
PENDING
  └─► IN_PROGRESS
        └─► RETURN_RECORDED
              └─► INTAKE_PENDING
```

| State | Meaning |
| --- | --- |
| `PENDING` | Created at order placement; expected cylinder vendor/size recorded. |
| `IN_PROGRESS` | Order out for delivery; agent en route to customer. |
| `RETURN_RECORDED` | Agent records returned cylinder vendor, size, and condition at doorstep. |
| `INTAKE_PENDING` | Delivery confirmed; cylinder awaits outlet intake confirmation in inventory. |

### Field Leg Rules

- One exchange request exists per refill order line.
- An order with three refill lines has three exchange requests.
- `RETURN_RECORDED` happens at the doorstep before PIN confirmation.
- The agent's recorded condition is the field fact of record.
- `INTAKE_PENDING` persists after customer PIN confirmation.
- The order can be `DELIVERED` while exchange requests are still
  `INTAKE_PENDING`.
- Financial closure for a refill order is blocked until all exchange requests
  reach `COMPLETED` or `FAILED` with approved exception.

## Delivery Modes and Assignment

### Express Delivery

- Express deliveries do not use delivery windows.
- Express orders dispatch as soon as possible within outlet operating hours
  after outlet acceptance, picking completion, and eligible agent availability.
- Customers do not select a scheduled window for express delivery.

### Batched Delivery

- Customers select a window at checkout from the outlet's active
  delivery-window policy within the active maximum advance-booking window.
- Launch global fallback is 2-hour delivery blocks.
- Future batched orders are allocated and inventory-reserved at placement.
- The order is held until the selected window opens for dispatch.

### Agent Eligibility

At launch, an eligible agent must:

- be an active delivery agent;
- have delivery scope for the run outlet;
- have no active picked-up run.

Formal shift schedules, live location, and route-distance optimization do not
add eligibility requirements, expand scope, or change assignment ranking at
launch.

### Default Assignment Ranking

```
rank agents by:
  1. queued_assignment_load  ASC
  2. least_recent_assignment ASC
  3. deterministic_tie_breaker
```

- A Dispatcher or Outlet Manager may manually assign or override assignment
  before pickup within outlet scope with a recorded reason.
- Operational risk alerts may be shown to permissioned Outlet Managers or Super
  Admins during manual assignment.
- Risk alerts do not block assignment or change assignment priority.

### Batched Delivery Runs

- Batched runs group eligible orders for the same outlet, service zone, and
  selected delivery window.
- Runs obey the outlet's active maximum batch size.
- The global default applies when no outlet override exists.
- Orders beyond the limit overflow into additional runs for the same window.
- A Dispatcher or Outlet Manager may manually split, merge, or regroup batched
  runs within outlet scope with audit.
- Manual run adjustment does not waive pickup, custody, cash, or run-closure
  rules.

### Agent Capacity

- A delivery agent may have multiple queued assignments.
- A delivery agent may have only one active delivery run in progress at a time.
- Batched delivery is the mechanism for multi-order runs.
- Express delivery is always a single-order run.
- The agent works through queued assignments sequentially.

## Field Action Requirements

- Delivery-agent field actions must be recorded live at launch.
- Required live actions include pickup, arrival, cash collection,
  returned-cylinder recording, PIN confirmation, failed delivery, and handover.
- If the agent cannot record the action live, the action has not completed for
  business purposes.
- Dead-zone handling is an operational field concern and does not create a
  deferred-completion business path.
- Every delivery run has a custody manifest.
- A batched run uses one batch-level custody handover with per-order item lines.
- An express run uses the same custody semantics for a one-order run.

## Cancellation Before Pickup

- `CANCELLED` is a delivery-task terminal state only before pickup/custody.
- After pickup, cancellation is no longer the delivery-task outcome.
- Failed-delivery, return-to-outlet, and custody reconciliation rules apply
  after pickup.

## Pickup Handover

Transition to `PICKED_UP` requires both:

1. An authorized outlet handover actor records handover of all outgoing items to
   the agent.
2. The agent confirms receipt of those items.

### Handover Rules

- Both confirmations must be recorded before the delivery run departs.
- A discrepancy between handover record and agent receipt confirmation is a
  custody exception requiring Outlet Manager resolution before departure.
- Missing agent receipt confirmation can be overridden only by an Outlet Manager
  or Super Admin with reason and audit.
- After `PICKED_UP`, outgoing stock remains in the agent's custody until
  delivery confirmation, failed-order return receipt at the outlet, or outlet
  reconciliation covers an approved exception.

### Agent Reassignment

- A Dispatcher or Outlet Manager may reassign the delivery agent within outlet
  scope until `PICKED_UP`, with a recorded reason.
- After pickup, reassignment requires Outlet Manager or Super Admin exception.

## Run Closure

- A delivery run cannot close until all included deliveries have terminal status
  or an approved Outlet Manager exception.
- Failed orders with physical goods must be returned to the outlet and
  receipt-confirmed by an authorized outlet return-receipt actor before run
  closure.
- For runs that include refill deliveries, closure also requires an authorized
  run-level returned-cylinder receipt actor to confirm that all
  customer-returned cylinders collected during the run have been physically
  received at the outlet.
- Run-level returned-cylinder receipt is separate from order-level intake and
  financial closure.
- For refill deliveries, returned-cylinder collection is captured before PIN
  confirmation and committed at `DELIVERED`.
- Returned cylinders do not become outlet inventory until outlet intake
  confirmation.

## PIN Confirmation

- PIN confirmation is the customer's final acceptance of the delivered order and
  amount due.
- For refill deliveries, before requesting PIN confirmation the agent must
  record every expected returned cylinder's vendor, size, condition, approved
  conversion, COD delta or cash collection, and delivery exception.

### PIN Attempt Limits

```
if count(invalid_attempts, window=15min) >= 5                        → lockout(15min)
if lockout_count >= 2
   OR count(lifetime_invalid_attempts_against_active_PIN) >= 10      → fallback required
```

### PIN Fallback

- Fallback actors are permissioned Outlet Cashier, Outlet Manager, Customer
  Support Agent, or Super Admin.
- Fallback requires customer verification note, reason code, and audit.
- Fallback regenerates a new PIN and invalidates the old PIN.
- The customer is notified by push/email.
- The replacement PIN is revealed once only to a fallback actor with explicit
  reveal permission.
- Delivery Agents cannot request, perform, view, or reveal PIN fallback.

## Doorstep Cylinder Exceptions

### Damaged or Unacceptable Returned Cylinder

- If the returned cylinder is damaged or otherwise unacceptable for refill, the
  agent's recorded condition decision controls the delivery outcome.
- There is no separate doorstep dispute flow at launch.
- The agent offers the customer a full-order conversion.
- If the customer refuses conversion, the order status becomes `FAILED`.
- If the customer accepts and Outlet Manager approval is recorded, the
  conversion is recorded as an approved full-order adjustment.
- The original refill line and price snapshot remain immutable.
- The COD delta is collected before PIN confirmation.

### Unexpected Vendor, Correct Size, Acceptable Condition

- If the returned cylinder is an unexpected vendor but correct size and
  acceptable condition, the agent accepts it.
- The actual incoming vendor becomes the exchange fact for intake and pricing.
- Refill price is recalculated using the actual same-size combination.
- Any delta is recorded as an explicit adjustment.
- If the recalculated price is higher than the original COD amount due, the
  delta is collected as a COD adjustment.
- Outlet Manager approval is required before PIN confirmation for a higher-price
  delta.
- If the recalculated price is lower, the amount due is reduced before COD
  collection.
- This path does not fail the order.

### Size Mismatch

- If the returned cylinder is acceptable but a different size from the expected
  refill size, the agent records the actual size and treats it as a
  size-mismatch exception.
- The original refill cannot complete silently.
- It may continue only through an Outlet Manager-approved full-order adjustment
  that keeps incoming and outgoing sizes matched and settles any COD/refund
  delta before PIN confirmation.
- If no adjusted exchange is approved and accepted, the delivery fails or
  follows the new-cylinder conversion path.

### Defective Outgoing Cylinder at Outlet

- If an outgoing cylinder is found defective before the agent departs, the agent
  records `defective product`.
- The defective unit is quarantined.
- The outlet attempts immediate replacement from available stock.
- For express orders, an available replacement proceeds immediately.
- For batched orders, replacement proceeds only when it does not disrupt the
  route or delivery window.
- If replacement is unavailable, or if batched replacement would be
  operationally disruptive, the order moves to the next eligible dispatch
  opportunity or batch/window.
- An Outlet Manager or Super Admin may override the reschedule decision.

### Exception Categories

- Customer-returned damaged or unacceptable cylinders and outlet-owned defective
  outgoing cylinders are separate exception categories for reporting and review.

## Failed Deliveries

- Delivery agents may mark a delivery `FAILED` without Outlet Manager approval.
- A controlled reason code and audit trail are required.
- An optional note may be added.
- Failed delivery cancels the order at launch.
- The agent cannot reschedule.
- If the customer still wants the goods, they must place a new order.
- Agents return undelivered physical goods to the outlet.
- Goods are not re-dispatched from the failed delivery.

## Field Proof and Evidence

- The same launch field-proof policy applies to express and batched deliveries.
- An authoritative timestamp is always recorded.

```
GPS required when:
  action IN [arrival, failed_delivery, doorstep_defect, terminal_delivery_attempt]
  AND device_can_provide_gps
// if GPS unavailable: agent records controlled location-unavailable reason and note
// action may proceed unless active policy requires additional approval

photo required when:
  damaged_returned_cylinder
  OR (unexpected_vendor OR wrong_size) AND physical_cylinder_present
  OR refused_returned_cylinder AND physical_cylinder_present
  OR defective_outgoing_cylinder
  OR failed_delivery AND (physical_defect OR safety_issue) AND safe_to_capture

photo NOT required when:
  missing_cylinder
  OR action IN [customer_unavailable, refusal, PIN_failure, cash_mismatch,
                PIN_fallback, handover, support_review]
  // unless the action also involves a physical defect or custody exception
```

### Evidence Retention and Access

- Required or voluntarily supplied photos are linked to the relevant delivery
  exception.
- Completed delivery evidence linked to delivery proof history is retained for
  seven years unless legal hold or approved legal/accounting retention policy
  changes the window.
- Pending evidence capture that never produces completed evidence expires after
  30 days.
- Evidence files are unavailable through ordinary Customer Support Agent or
  outlet-operational access until safety review clears.
- Evidence that fails safety review is quarantined from ordinary access paths.
- Quarantined evidence may be inspected only through Super Admin-level
  safety/audit access.
- The related delivery proof fact remains durable with a review marker for
  support and audit review.

## Agent Cash Handling

Agents collect exact cash due. Cash due can include:

- COD amount;
- delivery-fee delta;
- conversion delta;
- doorstep price-recalculation delta.

### Short and Over Collection

```
if abs_variance_per_order <= 10,000 UGX
   AND shift_cumulative_variance <= 50,000 UGX  → Outlet Manager approval
else                                             → Super Admin approval
```

- Required approval must be recorded before delivery can be marked complete.
- Over-collection excess is recorded as a cash discrepancy and reconciled at
  close.

### Agent Rules

- Agents record cash collected.
- Agents cannot modify the expected amount.
- The expected amount is determined by order total and approved delta records.
- A delivery run may contain orders with different collection needs only when
  the run manifest makes cash due explicit per order.
- Delivery agents are responsible for one combined cash-due amount.
- The business record preserves component amounts separately for reconciliation
  and audit.
- Components include base COD amount, delivery fee delta, conversion delta, and
  doorstep price-recalculation delta.
- The agent neither sees nor enters component breakdowns.
- Customer receipts show the delta/payment breakdown after financial closure.

### Shift Reconciliation

- Agent cash reconciliation is per shift and rolls up into outlet daily closing.
- An agent may end active field work with open custody only when the Outlet
  Manager acknowledges pending custody and carries it into reconciliation.
- The agent shift cannot fully close until cash reconciliation and custody
  reconciliation are complete.
- Cash reconciliation requires all cash collected to match expected cash or have
  an Outlet Manager-approved variance record for each order, or Super Admin
  approval where required by threshold policy.
- Custody reconciliation requires all outgoing items assigned to the agent to be
  accounted for as delivered, returned to outlet, or covered by approved
  exception.
- Open cash discrepancy or unaccounted item blocks full shift close.
- The Outlet Manager must resolve or explicitly approve each open item before
  shift close.
- At launch, custody discrepancies within Outlet Manager threshold are limited
  to one accessory item or one non-saleable/damaged cylinder discrepancy when
  documented custody facts identify the item and impact.
- Missing saleable filled cylinders always require Super Admin approval.
- Discrepancies above threshold require Super Admin approval with reason, notes,
  stock/cash impact, and audit before full shift close.

## Delivery Fee Rules

The base delivery fee is determined by the assigned outlet's active service zone
for the customer address.

### Fee Calculation

```
base_delivery_fee  := zone_template_default ?? outlet_service_zone_override
express_fee        := base_delivery_fee × express_multiplier
express_multiplier := outlet_or_zone_override ?? global_default (1.5)
```

### Launch Zone Defaults

| Zone | Radius | Base delivery fee |
| --- | --- | --- |
| CORE | 0 km <= distance < 5 km | UGX 3,000 |
| STANDARD | 5 km <= distance < 10 km | UGX 5,000 |
| EXTENDED | 10 km <= distance < 15 km | UGX 8,000 |

- These fees apply unless an outlet has an active service-zone fee override.
- Outlet Managers may adjust the express multiplier within the configured
  guardrail.
- Launch delivery-fee guardrail is the smaller of 15% or UGX 2,000 from the
  current approved basis.
- Above-guardrail delivery-fee changes require Super Admin approval.

### Fee Rules

- The express fee premium is presented to the customer at checkout.
- The express fee is reflected in the order total.
- For cascaded orders where the reassigned outlet has a different service zone
  or base fee, the express fee is recalculated using the new outlet's rate.
- Delivery fee is a separate customer-visible charge component in the order
  total.
- Delivery fee is not folded into product, refill, or accessory prices.
- Delivery-fee adjustments are explicit commercial adjustments.
- Money movement follows payment, refund, and financial-ledger rules.
- Delivery-fee adjustments do not rewrite the placed order's original charge.

## Failed Delivery Fee Waiver

No delivery fee is charged on any failed delivery, regardless of failure reason.

### Covered Failure Scenarios

- Customer unavailable.
- Customer refused delivery.
- PIN lockout.
- Safety issue.
- Damaged returned cylinder where the customer refuses conversion.
- Any other terminal failure outcome.

### Payment Effects

- For COD orders, the waived fee is not collected from the customer by the
  agent.
- Failed COD deliveries keep a zero-collection payment fact tied to the failed
  delivery reason.
- Failed deliveries do not create prepaid refund liabilities at launch because
  online orders are not prepaid.

## F-03: Damaged or Unacceptable Returned Cylinder at Doorstep

1. Agent delivers filled cylinder and attempts to collect the empty cylinder.
2. Returned cylinder is damaged or unacceptable for refill.
3. Agent records condition with required photo evidence.
4. The unacceptable cylinder is refused and does not enter outlet inventory.
5. Agent offers full-order conversion to new cylinder under the approved
   full-order adjustment path.
6. Customer must pay the price delta by COD.
7. If accepted and Outlet Manager approval is recorded, conversion is recorded
   as an explicit adjustment while the original refill order line and price
   snapshot remain immutable.
8. Agent collects COD delta, PIN is confirmed, and delivery is marked delivered.
9. If refused, delivery fails.
10. Undelivered outgoing goods are returned to the outlet and receipt-confirmed
    by an authorized outlet return-receipt actor before run closure.
11. The full order fails under all-or-nothing rule BI-09.
12. No delivery fee is charged.

## F-06: Defective Outgoing Cylinder at Pickup

1. Delivery agent arrives at outlet to pick up order.
2. Agent inspects outgoing cylinder and finds it defective.
3. Defect examples include dented, rusted, valve broken, or expired.
4. Agent records `defective product` with reason and required photo evidence.
5. The defective unit is quarantined in outlet stock.
6. Outlet attempts immediate replacement from available stock of the same vendor
   and size.
7. If replacement is available for an express order, delivery proceeds normally.
8. If replacement is available for a batched order without disrupting route or
   window, delivery proceeds normally.
9. If replacement is unavailable, or batched replacement would disrupt route or
   window, the order moves to the next eligible dispatch opportunity or
   batch/window.
10. Outlet Manager or Super Admin may override the reschedule decision.
11. The original defective cylinder remains quarantined pending outlet
    inspection and stock adjustment.

## F-07: Unexpected Same-Size Vendor Returned Cylinder at Doorstep

1. Delivery agent arrives at customer's address for a refill exchange.
2. Customer presents a returned cylinder.
3. Agent identifies the cylinder as an unexpected vendor with correct size and
   acceptable condition.
4. Agent records the actual incoming vendor with required photo evidence.
5. Agent confirms the size matches the outgoing cylinder.
6. Refill price is recalculated using the actual incoming vendor, outgoing
   vendor, and cylinder size.
7. If recalculated price is higher than original COD amount due, a COD delta is
   required.
8. Outlet Manager approval is required before PIN confirmation and delivery
   completion for the delta.
9. If recalculated price is lower than original COD amount due, the amount due
   is reduced before collection.
10. If recalculated price equals original COD amount due, exchange
    proceeds normally with no delta.
11. The actual incoming vendor becomes the exchange fact for intake and pricing.
12. The original order line and price snapshot remain immutable.
13. Any delta is recorded as an explicit adjustment.

## Permissions

Trimmed access matrix rows relevant to delivery. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-02 | P-03 | P-04 | P-05 | P-06 | P-07 | P-10 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Delivery assignment | - | - | - | Scoped | Scoped | - | Full |
| Delivery batch management | - | - | - | Scoped | Scoped | - | Full |
| Delivery execution pickup, PIN, COD | Own | - | - | - | - | - | Full |
| Delivery evidence safety review | - | - | - | - | - | - | Full |
| Agent cash handover | Own | - | - | - | Scoped receive | - | Full |
| Delivery PIN fallback regenerate | - | Scoped with explicit permission | - | - | Scoped with explicit permission | Scoped with explicit permission | Full |
| Outlet handover confirmation | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | Full |
| Failed-order return receipt | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | Full |
| Run-level returned-cylinder receipt | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | Full |

## Authorization Edge Cases

**E-02**: A Delivery Agent can see the customer's phone number only while an
order is assigned to them and active. They lose this access when the delivery
reaches a terminal state. Phone-number access is scoped to the active assignment
and is audit-sensitive under the active audit policy.

**E-08**: A Dispatcher can reassign delivery agents to orders within their outlet
until pickup with a recorded reason. After pickup, only an Outlet Manager or
Super Admin can reassign, and the action requires an explicit reason code.
