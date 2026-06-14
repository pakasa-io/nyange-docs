# Inventory

**Intent**: Define inventory behavior for reservations, outlet transfers,
vendor refill batches, returned-cylinder intake, and stock availability.

**Reader task**: Use this document to decide whether stock can be reserved,
released, committed, transferred, replenished, or recognized as returned-cylinder
inventory.

**Sources**: §6.4 Inventory Reservation Lifecycle, §6.5 Outlet Transfer
Lifecycle, §6.8 Vendor Refill Batch Lifecycle, §6.9 Refill Exchange Request
Lifecycle intake leg

**Related**:
[order.md](order.md) for reservation triggers;
[delivery.md](delivery.md) for the refill exchange field leg and delivery
custody;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-01 — Available stock never goes below zero.**

- Stock committed to orders cannot exceed physical available stock.
- When an outlet accepts an assigned order, the reservation is atomic relative
  to other concurrent reservations.
- An acceptance that cannot reserve every requested stock item is rejected.

**BI-06 — Reserved stock is unavailable.**

- Stock reserved for an active order cannot be sold, transferred, or otherwise
  consumed by another workflow.
- Reserved stock remains unavailable until it is explicitly committed or
  released.

**BI-11 — Outlet stock is isolated.**

- Stock at Outlet A cannot be drawn down by an order assigned to Outlet B.
- Stock transfers between outlets follow an explicit, audited transfer
  workflow.
- No order can implicitly access another outlet's inventory.

**BI-27 — Inventory ledger entries are append-only.**

- Inventory owns inventory ledger entries for stock reservations, commitments,
  releases, adjustments, transfers, vendor refill movements, returned-cylinder
  intake, and other stock movements.
- Inventory ledger entries are never modified or deleted after creation.
- Errors are corrected by appending compensating inventory ledger entries with
  explicit linkage to the erroneous entry.
- Shared append-only ledger storage or tooling is infrastructure and does not
  transfer logical inventory ledger ownership to Finance or another module.

### Cross-Aggregate Invariants

- **BI-08** is defined in [delivery.md](delivery.md). Delivery completion
  commits returned-cylinder field recording that later feeds inventory intake.
- **BI-19** is defined in [delivery.md](delivery.md). Returned cylinders require
  intake before inventory recognition.

## Boundary

- Inventory owns physical stock availability, reservation state, transfer state,
  vendor refill movement state, returned-cylinder intake recognition, inventory
  adjustment policy, and inventory ledger posting.
- Inventory does not own delivery completion, customer payment, refund payout,
  receipt issuance, or outlet cash custody.
- Incoming customer cylinders from refill delivery do not become outlet empty
  inventory until intake is confirmed.

## Inventory Adjustment Policy

Inventory owns the launch adjustment threshold policy, approval outcomes, and
ledger-posting rules for stock corrections.

### Launch Threshold

- At launch, inventory adjustment authority is policy-driven.
- Outlet Managers may post without separate Super Admin approval only
  single-unit damage or quarantine adjustments that do not increase available
  stock and carry a source reference, such as delivery, order, intake, vendor
  refill, or transfer record.
- This single-unit rule is the launch default Outlet Manager posting threshold.
- No higher Outlet Manager quantity threshold exists at launch.
- Every loss, missing-source adjustment, manual correction, positive
  available-stock increase, or absolute delta greater than one unit is
  `PENDING_APPROVAL`.
- `PENDING_APPROVAL` inventory adjustments require Super Admin approval before
  ledger movement is posted.
- Every adjustment requires reason, note, active policy code, ledger correlation
  when posted, and audit trail.

### Reconciliation Paths

- Inventory reconciliation supports source-referenced adjustments with reason
  codes.
- Adjustment records are audited and follow the active Inventory-owned approval
  threshold before stock is recognized as changed.

## Inventory Reservation Lifecycle

```
(no reservation)
  -> RESERVED
      -> COMMITTED
      -> RELEASED
```

| State | Meaning |
| --- | --- |
| `RESERVED` | Stock held for an active accepted order. |
| `COMMITTED` | Delivery succeeded; stock deducted from saleable inventory. |
| `RELEASED` | Claim cancellation, customer cancellation, or failed delivery returned stock to availability when valid. |

### Reservation Rules

- No stock is reserved by cart creation, cart update, quote generation,
  checkout readiness, or order placement.
- Reservation occurs only when an active permitted assigned outlet accepts an
  order.
- Fulfillment acceptance reserves every requested stock item atomically.
- Partial item reservation is prohibited.
- Only available stock can be reserved.
- Reserved and held stock is not visible as available to any new order.
- Reservation age alone does not release stock.
- Stock release occurs only through an explicit lifecycle event.
- Fulfillment acceptance cancellation before pickup releases the reservation.
- Customer cancellation before pickup releases the reservation.
- Delivery failure after pickup releases physically returned goods to
  availability only after outlet return receipt.
- Goods covered by an approved custody exception remain non-available until
  Inventory posts the appropriate adjustment or resolution record.
- Delivery success commits reserved outgoing stock.
- Incoming customer cylinders from refill delivery are tracked separately as
  `INTAKE_PENDING`.
- `INTAKE_PENDING` cylinders do not enter available empty stock until outlet
  intake is confirmed.
- During Delivery-coordinated completion, Inventory participates by committing
  reserved outgoing stock for the active accepted order.
- If the Delivery-coordinated completion fails before commit, outgoing stock
  remains reserved and unavailable; no stock commitment is posted.

## Outlet Transfer Lifecycle

Outlet transfers have separate request status and stock movement state. Request
status records business approval. Stock movement state records physical
availability and custody.

### Transfer Request Status

```
REQUESTED -> APPROVED
REQUESTED -> REJECTED
```

| State | Meaning |
| --- | --- |
| `REQUESTED` | Receiving outlet requests transfer. |
| `APPROVED` | Sending Outlet Manager or Super Admin approves the transfer request. |
| `REJECTED` | Terminal sender decline from `REQUESTED`; no inventory impact. |

### Transfer Stock Movement

```
TRANSFER_HOLD -> IN_TRANSIT -> RECEIVED
```

| State | Meaning |
| --- | --- |
| `TRANSFER_HOLD` | Sender stock is held outside sale availability after transfer approval and before dispatch. |
| `IN_TRANSIT` | Sender stock has left sender availability and is moving to the receiver. |
| `RECEIVED` | Receiver confirms receipt; sender hold is closed and stock is posted to receiver inventory. |

### Transfer Rules

- Only available filled cylinders, available accessories, and confirmed empty
  cylinders may be transferred.
- Reserved, damaged, quarantined, `TRANSFER_HOLD`, `IN_TRANSIT`, and
  `IN_REFILL` stock is not transferable.
- Approval moves the request status to `APPROVED` and creates sender
  `TRANSFER_HOLD` stock movement.
- Rejection moves the request status to `REJECTED` and creates no stock
  movement.
- Sender dispatch moves stock from `TRANSFER_HOLD` to `IN_TRANSIT`.
- Receiver receipt moves stock from `IN_TRANSIT` to `RECEIVED`.
- Super Admin can bypass ordinary transfer approval only through an audited
  override that records the approved request status, creates the sender
  `TRANSFER_HOLD`, and captures reason, before/after stock impact, and affected
  outlets.
- Both sides must participate: receiving outlet requests, sending outlet
  approves and moves stock into `TRANSFER_HOLD`, sender dispatch moves stock to
  `IN_TRANSIT`, and receiving outlet confirms receipt.
- Transfer receipt closes the sender hold and posts received stock to receiver
  inventory; it does not release stock back to sender availability.
- Every request status transition and stock movement transition is audit-logged
  with actor, outlet scope, timestamp, and reason where applicable.
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

The field leg is defined in [delivery.md](delivery.md). Inventory observes the
Delivery handoff after `RETURN_RECORDED` and owns intake from `INTAKE_PENDING`
through `INTAKE_CONFIRMED` or `FAILED`.

```
INTAKE_PENDING
  -> INTAKE_CONFIRMED
  -> FAILED
```

| State | Meaning |
| --- | --- |
| `INTAKE_PENDING` | Inventory-owned handoff state after successful delivery; cylinder awaits outlet intake confirmation. |
| `INTAKE_CONFIRMED` | Returned-cylinder intake actor confirms intake; cylinder enters outlet inventory. |
| `FAILED` | Intake rejected or expected handoff cannot be confirmed; inventory exception raised. |

### Intake Rules

- Each expected returned cylinder requires its own outlet intake confirmation
  before it becomes outlet empty inventory.
- Upstream `PENDING`, `IN_PROGRESS`, `RETURN_RECORDED`, and `CANCELLED` are
  Delivery-owned field-leg states and are not Inventory intake states.
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

## Stock Availability Model

### Facts

- Launch inventory accountability is aggregate by outlet, vendor, cylinder size,
  filled status, condition, and item/SKU as applicable.
- Individual cylinder serial-number tracking is not required at launch.
- Serial-number tracking is not a prerequisite for reservations, transfers,
  custody reconciliation, or vendor refill movements.

### Fill Lifecycle and Availability

- Cylinder fill lifecycle values such as `FILLED`, `EMPTY`, and `IN_REFILL` are
  independent from availability and condition status.
- Availability and condition statuses include available, reserved, damaged,
  quarantined, in transit, lost, sold, and returned.
- Stock is sellable, reservable, or transferable only when both fill lifecycle
  and availability/condition permit that action.

## Permissions

Trimmed access matrix rows relevant to inventory. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-04 | P-06 | P-08 | P-10 |
| --- | --- | --- | --- | --- |
| Inventory viewing | Scoped | Scoped | Read assigned outlets | Full |
| Reservation from fulfillment acceptance | - | Scoped with explicit permission | - | Full |
| Inventory adjustments submit | Scoped request | Scoped policy-limited post; above = request | - | Full |
| Inventory adjustments approve | - | - | - | Full |
| Outlet-to-outlet transfers | Scoped request | Scoped request/approve/receive | - | Full |
| Returned cylinder intake | Scoped | Scoped | - | Full |
| Vendor refill batch management | Scoped | Scoped | - | Full |
