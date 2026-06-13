# Delivery

**Intent**: Define the delivery lifecycle, delivery fee rules, agent cash handling, failed delivery fee waiver, field evidence requirements, and the field leg of the cylinder exchange request lifecycle.

**Sources**: §6.3 Delivery Lifecycle, §6.9 Refill Exchange Request Lifecycle (field leg), §7.5 Express Delivery Fee, §7.7 Agent Cash Handling, §7.12 Failed Delivery Fee Waiver, F-03, F-06, F-07

**Related**: [inventory.md](inventory.md) — §6.9 intake leg (INTAKE_PENDING through COMPLETED/FAILED); [order.md](order.md) — order lifecycle and F-01; [catalog.md](catalog.md) — product/refill price guardrails; [finance.md](finance.md) — F-05 forced financial closure; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-08 — Delivery confirmation is all-or-nothing.**
Customer confirmation of delivery (via PIN) is the single commit point for: delivery status, order status, outgoing stock commitment, returned-cylinder field recording, cash collection recording, and payment status. These succeed together or none do.

**BI-10 — Incoming and outgoing cylinder sizes match on refill exchange.**
A refill order exchanges a customer's empty cylinder for a filled cylinder of the same size. A customer who brings a 6kg empty cannot receive a 12kg filled, and vice versa. A size upgrade or downgrade is a new cylinder purchase, not a refill. A returned cylinder whose size differs from the expected size is a size-mismatch exception: the actual size must be recorded, and the original refill may continue only through an approved full-order adjustment that keeps the final incoming and outgoing sizes matched and handles any COD/refund delta before PIN confirmation; otherwise the order fails or converts under the new-cylinder path.

**BI-19 — No order can complete financially until all expected returned cylinders are accounted for.**
For refill orders, financial closure requires every expected returned cylinder to be accounted for through returned-cylinder intake confirmation or an approved exception when the cylinder is missing, rejected, or otherwise fails intake. Delivery can be confirmed by the customer before intake is complete, but financial closure waits.

---

## State

### Delivery Lifecycle (§6.3)

```
PENDING
  ├─► CANCELLED (terminal before pickup/custody)
  └─► ASSIGNED (delivery agent assigned)
        ├─► CANCELLED (terminal before pickup/custody)
        └─► PICKED_UP (outlet handover and agent receipt confirmed)
              └─► OUT_FOR_DELIVERY
                    └─► ARRIVED (agent at customer location)
                          ├─► DELIVERED (PIN confirmed; delivery leg terminal; order may still await financial closure)
                          └─► FAILED
                                └─► RETURNED_TO_OUTLET (terminal)
```

Terminal states: `CANCELLED` (before pickup/custody), `DELIVERED`, `RETURNED_TO_OUTLET`

### §6.9 Refill Exchange Request Lifecycle — Field Leg

Each `REFILL_EXCHANGE` order line creates one exchange request record. This record tracks the operational lifecycle of the physical cylinder exchange independently of the parent order status.

The field leg covers the stages from order creation through `RETURN_RECORDED`. The intake leg (INTAKE_PENDING onward) is in [inventory.md](inventory.md).

```
PENDING (created at order placement; expected cylinder vendor/size recorded)
  └─► IN_PROGRESS (order out for delivery; agent en route to customer)
        └─► RETURN_RECORDED (agent records returned cylinder vendor, size, condition at doorstep)
              └─► INTAKE_PENDING (delivery confirmed; cylinder awaiting outlet intake confirmation — see inventory.md)
```

Cross-reference: for the intake leg (INTAKE_PENDING through COMPLETED/FAILED), see [inventory.md](inventory.md).

**Field leg rules:**
- One exchange request exists per refill order line. An order with three refill lines has three exchange requests.
- `RETURN_RECORDED` happens at the doorstep before PIN confirmation. The agent's recorded condition is the field fact of record.
- `INTAKE_PENDING` persists after the customer confirms delivery (PIN). The order can be `DELIVERED` while exchange requests are still `INTAKE_PENDING`.
- Financial closure for a refill order is blocked until all exchange requests for that order reach `COMPLETED` or `FAILED` with an approved exception (BI-19).

---

## Business Rules

### Delivery Modes and Assignment

**Express deliveries:**
- Do not use delivery windows.
- Dispatched as soon as possible within outlet operating hours after the outlet accepts, picking is complete, and an eligible delivery agent is available.
- No scheduled window and no window-selection step for the customer.

**Batched deliveries:**
- Customer selects a window at checkout from the outlet's active delivery-window policy within the active maximum advance-booking window.
- At launch, global fallback: 2-hour delivery blocks.
- Future batched orders are allocated and inventory-reserved at placement, and the order is held until the selected window opens for dispatch.

**Agent eligibility (at launch):**
- An eligible agent must be an active delivery agent with delivery scope for the run outlet and no active picked-up run.
- Formal shift schedules, live location, and route-distance optimization do not add eligibility requirements, expand scope, or change assignment ranking at launch.

**Default assignment policy:**
1. Queued assignment load (lowest first).
2. Least-recent assignment.
3. Deterministic tie-breaker.

- A Dispatcher or Outlet Manager may manually assign or override an assignment before pickup within their outlet scope with a recorded reason.
- Operational risk alerts may be shown to permissioned Outlet Managers or Super Admins during manual assignment, but they do not block assignment or change assignment priority.

**Batched delivery runs:**
- Group eligible orders for the same outlet, service zone, and selected delivery window.
- Obey the outlet's active maximum batch size; use the global default when no outlet override exists.
- Orders beyond that limit overflow into additional runs for the same window.
- A Dispatcher or Outlet Manager may manually split, merge, or regroup batched runs within outlet scope, with audit, but manual adjustment does not waive pickup, custody, cash, or run-closure rules.

**Agent capacity:**
- A delivery agent may have multiple queued assignments but only one active delivery run in progress at a time.
- Batched delivery is the mechanism for multi-order runs; an express delivery is always a single-order run.
- The agent works through queued assignments sequentially.

### Field Action Requirements

- Delivery-agent field actions must be recorded live at launch: pickup, arrival, cash collection, returned-cylinder recording, PIN confirmation, failed delivery, and handover.
- If the agent cannot record the action live, the action has not completed for business purposes.
- Dead-zone handling is an operational field concern and does not create a deferred-completion business path.
- Every delivery run has a custody manifest. A batched run uses one batch-level custody handover with per-order item lines; an express run uses the same custody semantics for a one-order run.

### Cancellation Before Pickup

- `CANCELLED` is a delivery-task terminal state only before pickup/custody, when the underlying order or delivery task is cancelled.
- After pickup, cancellation is no longer the delivery-task outcome; failed-delivery, return-to-outlet, and custody reconciliation rules apply.

### Pickup Handover

- Transition to `PICKED_UP` requires two confirmations: (1) an authorized outlet handover actor records the handover of all outgoing items to the agent, and (2) the agent confirms receipt of those items. Both must be recorded before the delivery run departs.
- A discrepancy between the handover record and the agent's receipt confirmation is a custody exception requiring Outlet Manager resolution before departure.
- Missing agent receipt confirmation can be overridden only by an Outlet Manager or Super Admin with reason and audit.
- After `PICKED_UP`, outgoing stock remains in the agent's custody until delivery confirmation, failed-order return receipt at the outlet, or outlet reconciliation covers an approved exception.

**Agent reassignment:**
- A Dispatcher or Outlet Manager may reassign the delivery agent within outlet scope until `PICKED_UP`, with a recorded reason.
- After pickup, reassignment requires Outlet Manager or Super Admin exception.

### Run Closure

- A delivery run cannot close until all included deliveries have a terminal status or an approved Outlet Manager exception.
- Failed orders with physical goods must be returned to the outlet and receipt-confirmed by an authorized outlet return-receipt actor before run closure.
- For runs that include refill deliveries, closure additionally requires an authorized run-level returned-cylinder receipt actor to confirm that all customer-returned cylinders collected during the run have been physically received at the outlet. This cylinder receipt confirmation is a run-level gate, separate from the order-level intake and financial closure workflow.
- For refill deliveries, returned-cylinder collection is captured before PIN confirmation and committed at `DELIVERED`; returned cylinders do not become outlet inventory until outlet intake confirmation (a separate workflow).

### PIN Confirmation

- PIN confirmation is the customer's final acceptance of the delivered order and amount due.
- For refill deliveries, the agent must record every expected returned cylinder's vendor, size, condition, any approved conversion, COD delta or cash collection, and delivery exception before requesting PIN confirmation.
- **PIN attempt limits**: PIN entry is rate-limited. Launch defaults: 5 invalid attempts within a 15-minute window triggers a 15-minute lockout. After 2 lockouts, or after 10 lifetime invalid attempts against the active PIN, the agent cannot retry without fallback by a permissioned Outlet Cashier, Outlet Manager, Customer Support Agent, or Super Admin. The fallback requires customer verification note, reason code, and Safety/Audit record, regenerates a new PIN (invalidating the old one), notifies the customer via push/email, and may reveal the replacement PIN once only to one of those fallback actors with explicit reveal permission after customer verification. Delivery agents cannot request fallback, perform fallback, or view/reveal PINs.

### Doorstep Cylinder Exceptions

**Damaged or unacceptable returned cylinder (see F-03):**
- If the returned cylinder is damaged or otherwise unacceptable for refill (rusted, valve broken, etc.): the agent's recorded condition decision controls the delivery outcome; there is no separate doorstep dispute flow at launch.
- The agent offers the customer a full-order conversion. If refused, the order status becomes `FAILED`. If accepted and Outlet Manager approval is recorded, the conversion is recorded as an approved full-order adjustment while the original refill line and price snapshot remain immutable, and the COD delta is collected.

**Unexpected vendor, correct size, acceptable condition (see F-07):**
- If the returned cylinder at the doorstep is an unexpected vendor but the correct size and in acceptable condition: the agent accepts it.
- The actual incoming vendor becomes the exchange fact for intake and pricing; the refill price is recalculated using the actual same-size combination, and any delta is recorded as an explicit adjustment.
- If the recalculated price is higher than the original paid or COD amount due, the delta is collected as a COD top-up (Outlet Manager-approved before PIN confirmation).
- If lower, unpaid/COD orders reduce the amount due, while prepaid mobile-money orders create a cash refund liability at financial closure. The exchange proceeds; this path does not fail the order.

**Size mismatch (see BI-10):**
- If the returned cylinder is acceptable but a different size from the expected refill size: the agent records the actual size and treats it as a size-mismatch exception.
- The original refill cannot complete silently; it may continue only through an Outlet Manager-approved full-order adjustment that keeps incoming and outgoing sizes matched and settles any COD/refund delta before PIN confirmation.
- If no such adjusted exchange is approved and accepted, the delivery fails or follows the new-cylinder conversion path.

**Defective outgoing cylinder at outlet (see F-06):**
- If an outgoing cylinder is found defective at the outlet before the agent departs: agent records "defective product," the unit is quarantined, and the outlet attempts an immediate replacement from available stock.
- For express orders, an available replacement proceeds immediately; for batched orders, the replacement proceeds only when it does not disrupt the route or delivery window.
- If replacement is unavailable, or if a batched replacement would be operationally disruptive, the order is moved to the next eligible dispatch opportunity or batch/window. An Outlet Manager or Super Admin may override the reschedule decision.

**Cylinder categories:**
- Customer-returned damaged or unacceptable cylinders and outlet-owned defective outgoing cylinders remain separate exception categories for reporting and review.

### Failed Deliveries

- Delivery agents may mark a delivery `FAILED` without Outlet Manager approval. A controlled reason code and audit trail are required; an optional note may be added.
- Failed delivery cancels the order at launch — the agent cannot reschedule. If the customer still wants the goods, they must place a new order.
- Agents return any undelivered physical goods to the outlet; goods are not re-dispatched.

### Field Proof and Evidence

- The same launch field-proof policy applies to express and batched deliveries.
- An authoritative timestamp is always recorded.
- **GPS required** for: arrival, failed delivery, doorstep defect, and terminal delivery attempts when the device can provide it; otherwise the agent must record a controlled location-unavailable reason and note, and the action may proceed unless the active manager/outlet override policy requires additional approval.
- **Photos required** for:
  - Damaged returned cylinders.
  - Unexpected-vendor or wrong-size returned cylinders when a physical cylinder is present.
  - Refused returned cylinders when a physical cylinder is present.
  - Every defective outgoing cylinder report.
  - Failed-delivery reasons involving a physical defect or safety issue when safe to capture.
- **No photo required** for missing returned cylinders (time/location facts and a reason note are required), or customer-unavailable, refusal, PIN-failure, cash-mismatch, PIN-fallback, handover, and support-review paths (structured facts, reason notes, and audit required) unless they also involve a physical defect or custody exception.

**Evidence retention:**
- Required or voluntarily supplied photos are linked to the relevant delivery exception.
- Completed delivery evidence linked to delivery proof history is retained for seven years unless a legal hold or approved legal/accounting retention policy changes the window.
- Pending evidence capture that never produces completed evidence expires after 30 days.
- Evidence files are unavailable through ordinary Customer Support Agent or outlet-operational access until safety review clears.
- Evidence that fails safety review is quarantined from those ordinary access paths and may be inspected only through Super Admin-level safety/audit access; the related delivery proof fact remains durable with a review marker for support and audit review.

---

## Agent Cash Handling (§7.7)

Agents collect exact cash due (COD amount plus any approved underpayment top-up, delivery-fee delta, conversion delta, or doorstep price-recalculation delta).

**Short collection:**
- Requires Outlet Manager approval, or Super Admin approval where the active threshold policy requires it, before delivery can be marked complete.
- At launch, Outlet Managers may approve short collections up to UGX 10,000 per order and UGX 50,000 per agent shift; above either limit requires Super Admin approval.

**Over collection:**
- The excess is recorded as a cash discrepancy and requires Outlet Manager approval, or Super Admin approval where the active threshold policy requires it, before delivery can be marked complete.
- At launch, Outlet Managers may approve over-collection discrepancies up to UGX 10,000 per order and UGX 50,000 per agent shift; above either limit requires Super Admin approval.
- The approved discrepancy is reconciled at close.

**Agent rules:**
- Agents record cash collected. They cannot modify the expected amount.
- The expected amount is determined by the order total and any approved underpayment top-up, delivery-fee delta, conversion delta, or doorstep price-recalculation delta.
- A delivery run may contain orders with different payment collection needs only when the run manifest makes cash due explicit per order.
- Delivery agents are responsible for one combined cash-due amount (order COD total plus any approved top-up or delta). The business record preserves each component separately — base COD amount, delivery fee delta, underpayment top-up, conversion delta, doorstep price-recalculation delta — for reconciliation and audit; the agent neither sees nor enters component breakdowns. Customer receipts show the delta/payment breakdown after financial closure.

**Shift reconciliation:**
- Agent cash reconciliation is per shift and rolls up into outlet daily closing.
- An agent may end active field work with open custody only when the Outlet Manager acknowledges the pending custody and carries it into reconciliation.
- The agent shift cannot fully close until two reconciliations are complete:
  1. **Cash reconciliation** — all cash collected matches expected or has an Outlet Manager-approved variance record for each order, or Super Admin approval where the active threshold policy requires it.
  2. **Custody reconciliation** — all outgoing items assigned to the agent are accounted for (delivered, returned to outlet, or covered by an approved exception).
- An open cash discrepancy or an unaccounted item blocks full shift close. The Outlet Manager must resolve or explicitly approve each open item before the shift can be fully closed.
- At launch, custody discrepancies within the Outlet Manager threshold are limited to one accessory item or one non-saleable/damaged cylinder discrepancy when documented custody facts identify the item and impact; missing saleable filled cylinders always require Super Admin approval.
- Discrepancies above threshold require Super Admin approval with reason, notes, stock/cash impact, and audit before the shift can fully close.

---

## Delivery Fee Rules (§7.5)

The base delivery fee is determined by the assigned outlet's active service zone for the customer address.

**Fee calculation:**

1. Each active outlet service zone maps to a global zone template.
2. The template's default fee applies unless the outlet has an active service-zone fee override.
3. Express delivery fee applies a global default multiplier of **1.5** against that service-zone base fee.
4. This multiplier is configurable per outlet and per service zone.
5. Outlet Managers may adjust the multiplier within the configured guardrail.

**Launch zone defaults:**

| Zone     | Radius                   | Base delivery fee |
|----------|--------------------------|-------------------|
| CORE     | 0 km ≤ distance < 5 km   | UGX 3,000         |
| STANDARD | 5 km ≤ distance < 10 km  | UGX 5,000         |
| EXTENDED | 10 km ≤ distance < 15 km | UGX 8,000         |

These fees apply unless an outlet has an active service-zone fee override.

**Guardrail (at launch):** Delivery-fee changes — smaller of 15% or UGX 2,000 from the current approved basis. Outlet Managers may change the multiplier within this guardrail; above-guardrail changes require Super Admin approval.

**Additional rules:**

- The express fee premium is presented to the customer at checkout and is reflected in the order total.
- For cascaded orders where the reassigned outlet has a different service zone or base fee, the express fee is recalculated using the new outlet's rate.
- The delivery fee is a separate customer-visible charge component in the order total; it is not folded into product, refill, or accessory prices.
- Delivery-fee adjustments are explicit commercial adjustments; any money movement follows payment, refund, and financial-ledger rules and does not rewrite the placed order's original charge.

---

## Failed Delivery Fee Waiver (§7.12)

No delivery fee is charged on any failed delivery, regardless of the failure reason.

- Applies to all failure scenarios including: customer unavailable, customer refused delivery, PIN lockout, safety issue, damaged returned cylinder where the customer refuses conversion, and any other terminal failure outcome.
- For COD orders, the waived fee means the delivery fee is not collected from the customer by the agent.
- For prepaid mobile-money orders, the waived delivery fee becomes part of the refund liability created when the order transitions to `CANCELLED_PENDING_REFUND`.
- Failed COD deliveries keep a zero-collection payment fact tied to the failed delivery reason.
- Failed prepaid mobile-money deliveries create a cash refund liability for the full paid amount until audited cash refund completion.

---

## F-03: Damaged or Unacceptable Returned Cylinder at Doorstep

1. Agent delivers filled cylinder, attempts to collect empty cylinder.
2. Returned cylinder is damaged or otherwise unacceptable for refill (rusted, valve broken, etc.).
3. Agent records condition with required photo evidence. The unacceptable cylinder is refused and does not enter outlet inventory.
4. Agent offers full-order conversion to new cylinder under the approved full-order adjustment path. Customer must pay the price delta by COD.
5. If accepted and Outlet Manager approval is recorded: conversion is recorded as an explicit adjustment while the original refill order line and price snapshot remain immutable; agent collects the COD delta; PIN confirmed; delivery marked delivered.
6. If refused: delivery fails; undelivered outgoing goods are returned to the outlet and receipt-confirmed by an authorized outlet return-receipt actor before run closure; the full order fails under all-or-nothing rule (BI-09). No delivery fee charged (§7.12). Mobile-money prepayment becomes refund liability.

---

## F-06: Defective Outgoing Cylinder at Pickup

1. Delivery agent arrives at outlet to pick up order.
2. Agent inspects outgoing cylinder and finds it defective (dented, rusted, valve broken, expired).
3. Agent records "defective product" with reason and required photo evidence. The defective unit is quarantined in outlet stock.
4. Outlet attempts immediate replacement from available stock of the same vendor and size.
5. If replacement is available for an express order, or available for a batched order without disrupting the delivery route or window: agent takes replacement, delivery proceeds normally.
6. If replacement is unavailable, or if a batched replacement would disrupt the delivery route/window: the order is moved to the next eligible dispatch opportunity or batch/window. Outlet Manager or Super Admin may override the reschedule decision.
7. The original defective cylinder remains quarantined pending outlet inspection and stock adjustment.

---

## F-07: Unexpected Same-Size Vendor Returned Cylinder at Doorstep

1. Delivery agent arrives at customer's address for a refill exchange.
2. Customer presents a returned cylinder. Agent inspects and identifies it as an unexpected vendor, with the correct size and acceptable condition.
3. Agent records the actual incoming vendor with required photo evidence and confirms the size matches the outgoing cylinder.
4. The refill price is recalculated using the actual same-size combination (actual incoming vendor, outgoing vendor, cylinder size).
5. If recalculated price is higher than the original paid or COD amount due: a COD top-up for the delta is required. Outlet Manager approval is required before PIN confirmation and delivery completion.
6. If recalculated price is lower than the original paid or COD amount due: unpaid/COD orders reduce the amount due; prepaid mobile-money orders create a cash refund liability at financial closure. The exchange proceeds immediately without waiting for refund resolution.
7. If recalculated price equals the original paid or COD amount due: exchange proceeds normally with no delta.
8. The actual incoming vendor becomes the exchange fact for intake and pricing. The original order line and price snapshot remain immutable; any delta is recorded as an explicit adjustment.

---

## Permissions

Trimmed access matrix rows relevant to delivery. Full matrix: [identity-auth.md](identity-auth.md).

| Capability | P-02 | P-03 | P-04 | P-05 | P-06 | P-07 | P-10 |
|---|---|---|---|---|---|---|---|
| Delivery assignment | – | – | – | Scoped | Scoped | – | Full |
| Delivery batch management | – | – | – | Scoped | Scoped | – | Full |
| Delivery execution (pickup, PIN, COD) | Own | – | – | – | – | – | Full |
| Delivery evidence safety review | – | – | – | – | – | – | Full |
| Agent cash handover | Own | – | – | – | Scoped (receive) | – | Full |
| Delivery PIN fallback (regenerate) | – | Scoped (with explicit permission) | – | – | Scoped (with explicit permission) | Scoped (with explicit permission) | Full |
| Outlet handover confirmation | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | Full |
| Failed-order return receipt | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | Full |
| Run-level returned-cylinder receipt | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | Full |

---

## Authorization Edge Cases

**E-02**: A Delivery Agent can see the customer's phone number only while an order is assigned to them and active. They lose this access when the delivery reaches a terminal state. Phone-number access is scoped to the active assignment and is audit-sensitive under the active audit policy.

**E-08**: A Dispatcher can reassign delivery agents to orders within their outlet until the point of pickup with a recorded reason. After pickup, only an Outlet Manager or Super Admin can reassign, and the action requires an explicit reason code.
