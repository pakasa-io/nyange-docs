# Inventory

**Intent**: Define inventory behavior for reservations, outlet transfers,
vendor refill batches, returned-cylinder intake, stock counts, and low-stock
alerts.

**Reader task**: Use this document to decide whether stock can be reserved,
released, committed, transferred, counted, replenished, or recognized as
returned-cylinder inventory.

**Sources**: §6.4 Inventory Reservation Lifecycle, §6.5 Outlet Transfer
Lifecycle, §6.8 Vendor Refill Batch Lifecycle, §6.9 Refill Exchange Request
Lifecycle intake leg, §7.17 Stock Count Behaviour, §7.18 Low-Stock Alerts

**Related**:
[order.md](order.md) for reservation triggers;
[delivery.md](delivery.md) for the refill exchange field leg and delivery
custody;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-01 — Available stock never goes below zero.**

- Stock committed to orders cannot exceed physical available stock.
- When an outlet claims an order, the stock claim is atomic relative to other
  concurrent claims.
- A claim that cannot reserve every requested stock item is rejected.

**BI-06 — Reserved stock is unavailable.**

- Stock reserved for an active order cannot be claimed, sold, transferred, or
  otherwise consumed by another workflow.
- Reserved stock remains unavailable until it is explicitly committed or
  released.

**BI-11 — Outlet stock is isolated.**

- Stock at Outlet A cannot be drawn down by an order claimed by Outlet B.
- Stock transfers between outlets follow an explicit, audited transfer
  workflow.
- No order can implicitly access another outlet's inventory.

### Cross-Aggregate Invariants

- **BI-08** is defined in [delivery.md](delivery.md). Delivery completion
  commits returned-cylinder field recording that later feeds inventory intake.
- **BI-19** is defined in [delivery.md](delivery.md). Returned cylinders require
  intake before inventory recognition.

## Boundary

- Inventory owns physical stock availability, reservation state, transfer state,
  vendor refill movement state, and returned-cylinder intake recognition.
- Inventory does not own delivery completion, customer payment, refund payout,
  receipt issuance, or outlet cash custody.
- Incoming customer cylinders from refill delivery do not become outlet empty
  inventory until intake is confirmed.

## Inventory Reservation Lifecycle

```
(no reservation)
  -> RESERVED
      -> COMMITTED
      -> RELEASED

TRANSFER_HOLD
  -> released when transfer received
```

| State | Meaning |
| --- | --- |
| `RESERVED` | Stock held for an active claimed order. |
| `COMMITTED` | Delivery succeeded; stock deducted from saleable inventory. |
| `RELEASED` | Claim cancellation, customer cancellation, or failed delivery returned stock to availability when valid. |
| `TRANSFER_HOLD` | Stock locked for an in-transit outlet transfer. |

### Reservation Rules

- No stock is reserved by cart creation, cart update, quote generation,
  checkout readiness, or order placement.
- Reservation occurs only when an active permitted outlet claims an order.
- Claiming reserves every requested stock item atomically.
- Partial item reservation is prohibited.
- Only available stock can be reserved.
- Reserved and held stock is not visible as available to any new order.
- Reservation age alone does not release stock.
- Stock release occurs only through an explicit lifecycle event.
- Outlet claim cancellation before pickup releases the reservation.
- Customer cancellation before pickup releases the reservation.
- Delivery failure after pickup releases stock only after picked-up goods return
  to outlet stock or are covered by an approved custody exception.
- Delivery success commits reserved outgoing stock.
- Incoming customer cylinders from refill delivery are tracked separately as
  `INTAKE_PENDING`.
- `INTAKE_PENDING` cylinders do not enter available empty stock until outlet
  intake is confirmed.

## Outlet Transfer Lifecycle

```
REQUESTED
  -> APPROVED
      -> IN_TRANSIT
          -> RECEIVED

REJECTED
```

| State | Meaning |
| --- | --- |
| `REQUESTED` | Receiving outlet requests transfer. |
| `APPROVED` | Sending Outlet Manager approves; sender holds stock outside sale availability. |
| `IN_TRANSIT` | Stock has left sender availability and is not yet received. |
| `RECEIVED` | Receiving outlet confirms receipt; stock added to receiver. |
| `REJECTED` | Terminal sender decline from `REQUESTED`; no inventory impact. |

### Transfer Rules

- Only available filled cylinders, available accessories, and confirmed empty
  cylinders may be transferred.
- Reserved, damaged, quarantined, `IN_TRANSIT`, and `IN_REFILL` stock is not
  transferable.
- Super Admin can bypass the transfer approval step only through an audited
  override with reason, before/after stock impact, and affected outlets.
- Both sides must participate: receiving outlet requests, sending outlet
  approves and releases stock, and receiving outlet confirms receipt.
- Every transfer transition is audit-logged with actor, outlet scope, timestamp,
  and reason where applicable.
- Inter-outlet stock transfers do not involve financial settlement.
- No financial settlement entry is created between outlets for a stock transfer.
- Launch outlet transfers do not create internal settlement records.

## Vendor Refill Batch Lifecycle

```
EMPTY
  -> IN_REFILL
      -> FILLED
```

| State | Meaning |
| --- | --- |
| `EMPTY` | Confirmed empty cylinders at outlet. |
| `IN_REFILL` | Cylinders dispatched to vendor depot; unavailable. |
| `FILLED` | Returned filled cylinders confirmed by authorized actor. |

### Vendor Refill Rules

- Launch transition is `EMPTY -> IN_REFILL -> FILLED`.
- Confirmed empty cylinders enter `IN_REFILL` when dispatched to the vendor.
- Cylinders become filled stock only after an authorized vendor-refill actor
  confirms return intake.
- The vendor depot work itself is external.
- The business record tracks audited inventory state transitions and ledger
  movements, not the vendor's internal process.
- `IN_REFILL` cylinders are excluded from available stock and cannot be sold or
  transferred.
- Vendor, size, and quantity sent must be recorded at dispatch.
- Depot name, external reference, and notes may be recorded when known.
- On return, an authorized vendor-refill actor records received quantity and
  condition before confirming intake.
- Shortage or overage must be reconciled through standard inventory adjustment
  approval and ledger posting, with reason code, audit trail, and
  `INVENTORY_ADJUSTMENT_LAUNCH_V1` threshold evaluation before any ledger
  movement.
- Overage cylinders remain pending and unavailable for sale or transfer until
  the inventory adjustment is posted.

## Refill Exchange Request Intake Leg

The field leg is defined in [delivery.md](delivery.md). Inventory owns
`INTAKE_PENDING` through `INTAKE_CONFIRMED` or `FAILED`.

```
PENDING
  -> IN_PROGRESS
      -> RETURN_RECORDED
          -> INTAKE_PENDING
              -> INTAKE_CONFIRMED
              -> FAILED

CANCELLED
```

| State | Meaning |
| --- | --- |
| `PENDING` | Created at order placement; expected cylinder vendor/size recorded. |
| `IN_PROGRESS` | Order out for delivery; agent en route to customer. |
| `RETURN_RECORDED` | Agent records returned cylinder vendor, size, and condition at doorstep. |
| `INTAKE_PENDING` | Delivery succeeded; cylinder awaiting outlet intake confirmation. |
| `INTAKE_CONFIRMED` | Returned-cylinder intake actor confirms intake; cylinder enters outlet inventory. |
| `FAILED` | Intake rejected or cylinder not returned; exception raised. |
| `CANCELLED` | Order cancelled before pickup; no cylinder exchange occurred. |

### Intake Rules

- Each expected returned cylinder requires its own outlet intake confirmation
  before it becomes outlet empty inventory.
- `INTAKE_PENDING` persists after delivery completion.
- The order can be `DELIVERED` while exchange requests are still
  `INTAKE_PENDING`.
- A `FAILED` intake creates an inventory exception and may supply operational
  escalation context.
- A failed intake does not by itself undo completed customer delivery.

### Intake Correction

- During intake, a returned-cylinder intake actor may correct an agent recording
  mismatch in returned-cylinder vendor or size.
- The actor may be an Inventory Clerk, Outlet Manager, or Super Admin within
  scope.
- Correction requires reason code, before/after values, and audit trail.
- Material differences require Outlet Manager or Super Admin approval.
- If the physical cylinder size does not match accepted delivery facts, intake
  follows the `FAILED` intake and approved-exception path.

## Stock Count Behaviour

### Facts

- Launch inventory accountability is aggregate by outlet, vendor, cylinder size,
  filled status, condition, and item/SKU as applicable.
- Individual cylinder serial-number tracking is not required at launch.
- Serial-number tracking is not a prerequisite for stock counts, reservations,
  transfers, custody reconciliation, or vendor refill movements.

### Fill Lifecycle and Availability

- Cylinder fill lifecycle values such as `FILLED`, `EMPTY`, and `IN_REFILL` are
  independent from availability and condition status.
- Availability and condition statuses include available, reserved, damaged,
  quarantined, in transit, lost, sold, and returned.
- Stock is sellable, reservable, or transferable only when both fill lifecycle
  and availability/condition permit that action.

### Count Rules

- Stock counts do not freeze outlet operations.
- When a count begins, expected quantities are fixed as the count-start basis.
- Ledger movements during the count window are tracked and used to calculate
  variance at count close.
- Orders may continue to be placed, claimed, and fulfilled while a count is in
  progress.

## Low-Stock Alerts

Low-stock alerts are based on available stock only.

### Launch Defaults

| Stock type | Alert threshold |
| --- | --- |
| Saleable filled cylinders | Available quantity at or below 2 |
| Saleable accessories | Available quantity at or below 1 |
| Empty cylinders, non-saleable stock | Disabled by default, threshold 0 unless explicitly configured |

### Alert Rules

- Defaults may be overridden per outlet, product, or stock item.
- Alerts are scoped to the outlet and stock item.
- Repeated alerts for the same outlet/stock item are deduplicated while the item
  remains below threshold during the four-hour launch threshold window.
- Alerts are visible to permissioned Outlet Managers, Area Managers, and Super
  Admins within authorized scope.
- Alerts may show relevant `IN_REFILL` and incoming-transfer context.
- Alerts do not reserve stock, block checkout, alter claiming, or forecast
  demand.

## Permissions

Trimmed access matrix rows relevant to inventory. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-04 | P-06 | P-07 | P-08 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Inventory viewing | Scoped | Scoped | Scoped | Read assigned outlets | Full |
| Reservation from order claim | - | Scoped with explicit permission | - | - | Full |
| Inventory adjustments submit | Scoped request | Scoped policy-limited post; above = request | - | - | Full |
| Inventory adjustments approve | - | - | - | - | Full |
| Outlet-to-outlet transfers | Scoped request | Scoped request/approve/receive | - | - | Full |
| Returned cylinder intake | Scoped | Scoped | - | - | Full |
| Vendor refill batch management | Scoped | Scoped | - | - | Full |
| Low-stock alerts | - | Scoped | - | Read assigned outlets | Full |
