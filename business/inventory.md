# Inventory

**Intent**: Define the inventory reservation lifecycle, outlet transfer lifecycle, vendor refill batch lifecycle,
cylinder exchange intake leg, stock count behaviour, and low-stock alert rules for the Nyange platform.

**Sources**: §6.4 Inventory Reservation Lifecycle, §6.5 Outlet Transfer Lifecycle, §6.8 Vendor Refill Batch Lifecycle,
§6.9 Refill Exchange Request Lifecycle (intake leg), §7.17 Stock Count Behaviour, §7.18 Low-Stock Alerts

**Related**: [delivery.md](delivery.md) — §6.9 field leg (PENDING through RETURN_RECORDED); [order.md](order.md) —
reservation triggers; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-01 — Available stock never goes below zero.**
Stock committed to orders cannot exceed physical available stock. When an order claims stock, the claim is atomic
relative to other concurrent claims. A claim that cannot be satisfied is rejected.

**BI-06 — Reserved stock is unavailable.**
Stock reserved for any active order — whether online or walk-in — cannot be allocated to any other order. This applies
equally to every sales channel sharing the same physical inventory.

**BI-11 — Outlet stock is isolated.**
Stock at Outlet A cannot be drawn down by an order assigned to Outlet B. Stock transfers between outlets follow an
explicit, audited transfer workflow. No order can implicitly access another outlet's inventory.

Cross-reference for delivery-and-inventory shared invariant:

- **BI-08** (delivery confirmation all-or-nothing) is in [delivery.md](delivery.md). The returned-cylinder field
  recording committed at PIN confirmation feeds into the intake leg here.
- **BI-19** (financial closure blocked until cylinders accounted for) is in [delivery.md](delivery.md); enforced at the
  inventory intake gate.

---

## State

### Inventory Reservation Lifecycle (§6.4)

```
(no reservation)
  └─► RESERVED — stock held for an active order
        ├─► COMMITTED — delivery confirmed; stock deducted
        ├─► RELEASED — order cancelled; stock returned to available
        └─► REASSIGNMENT_HOLD — transitional during outlet reassignment
              ├─► RESERVED (promoted on reassignment acceptance)
              └─► RELEASED (if outlet times out, customer rejects, or reassignment otherwise fails)

TRANSFER_HOLD — stock locked for an in-transit outlet-to-outlet transfer
  └─► (released when transfer received)

PICK_REVERSAL_PENDING — stock locked after a post-picking exception; picked goods await
                        authorized inventory/custody confirmation that they are physically back and sellable
  └─► RELEASED (authorized inventory/custody actor confirms physical return and sellable condition; stock returns to available)

PENDING_RELEASE — release entry failed to post; stock remains unavailable until corrected
```

**Rules:**

- Reserved and held stock is not visible as available to any new order or walk-in sale.
- Only available stock can be reserved.
- Reservation age alone does not release stock. Stock release occurs only through an explicit policy-defined lifecycle
  such as unpaid mobile-money cancellation, failed delivery return, order cancellation, or another audited release
  workflow.
- A reservation that cannot be committed (delivery failed) must be released, not left open.
- Incoming customer cylinders (from refill delivery) are tracked separately as `PENDING_INTAKE` and do not enter
  available empty stock until outlet intake is confirmed.
- `PICK_REVERSAL_PENDING` is entered when an order is pulled back after picking has begun (Outlet Manager or Super Admin
  exception workflow only). The picked stock remains unavailable to all new orders until an authorized inventory/custody
  actor explicitly confirms it is physically back at the outlet and sellable. If the item is damaged or missing, the
  normal damaged/lost stock workflow applies instead of release to available. A recovery outlet may proceed on its own
  reserved stock before the original reversal resolves if the exception is approved, but financial closure of the
  original order waits for the outcome.

---

### Outlet Transfer Lifecycle (§6.5)

```
REQUESTED (by receiving outlet)
  └─► APPROVED (sending Outlet Manager approves; sender holds transfer stock outside sale availability)
        └─► IN_TRANSIT (stock has left sender availability and is not yet received)
              └─► RECEIVED (receiving outlet confirms receipt; stock added to receiver)

  REJECTED   (terminal — sender declines from `REQUESTED`; no inventory impact)
```

**Rules:**

- Only available filled cylinders, available accessories, and confirmed empty cylinders may be transferred. Reserved,
  damaged, quarantined, `IN_TRANSIT`, and `IN_REFILL` stock is not transferable.
- Super Admin can bypass the transfer approval step only through an audited override with reason, before/after stock
  impact, and affected outlets.
- Both sides must participate: the receiving outlet requests the transfer, the sending outlet approves and releases
  stock, and the receiving outlet confirms receipt.
- Every transfer state transition is audit-logged with the actor, outlet scope, timestamp, and reason where applicable.
- Inter-outlet stock transfers do not involve financial settlement. Only inventory movements are tracked. No financial
  settlement entry is created between outlets for a stock transfer. This distinguishes transfers from post-payment order
  reassignments, which do create an internal settlement at financial closure.

---

### Vendor Refill Batch Lifecycle (§6.8)

```
EMPTY (confirmed empty cylinders at outlet)
  └─► IN_REFILL (authorized vendor-refill actor dispatches cylinders to vendor depot; unavailable)
        └─► FILLED (authorized vendor-refill actor confirms returned filled cylinders and condition)
```

**Rules:**

- The launch inventory-state transition is `EMPTY -> IN_REFILL -> FILLED`: confirmed empty cylinders enter `IN_REFILL`
  when dispatched to the vendor, and they become filled stock only after an authorized vendor-refill actor confirms
  return intake. The vendor depot work itself is external; the business record tracks audited inventory state
  transitions and ledger movements, not the vendor's internal process.
- `IN_REFILL` cylinders are excluded from available stock and cannot be sold or transferred.
- The vendor, size, and quantity sent must be recorded at dispatch; depot name, external reference, and notes may be
  recorded when known.
- Upon return, an authorized vendor-refill actor records received quantity and condition before confirming intake. Any
  shortage (fewer returned than dispatched) or overage (more returned than dispatched) must be reconciled through the
  standard inventory adjustment approval and ledger-posting lifecycle, with reason code, audit trail, and
  `INVENTORY_ADJUSTMENT_LAUNCH_V1` threshold evaluation before any ledger movement. Overage cylinders remain pending and
  unavailable for sale, transfer, or reassignment until the inventory adjustment is posted.

---

### Refill Exchange Request Lifecycle — Intake Leg (§6.9)

Cross-reference: for the field leg (PENDING through RETURN_RECORDED, doorstep recording rules),
see [delivery.md](delivery.md).

The intake leg begins after delivery confirmation. The shared lifecycle diagram is:

```
PENDING (created at order placement; expected cylinder vendor/size recorded)
  └─► IN_PROGRESS (order out for delivery; agent en route to customer)
        └─► RETURN_RECORDED (agent records returned cylinder vendor, size, condition at doorstep)
              └─► INTAKE_PENDING (delivery confirmed; cylinder awaiting outlet intake confirmation)
                    ├─► COMPLETED (returned-cylinder intake actor confirms intake; cylinder enters outlet inventory)
                    └─► FAILED (intake rejected or cylinder not returned; exception raised)

  CANCELLED  (order cancelled before delivery; no cylinder exchange occurred)
```

The intake leg covers INTAKE_PENDING through COMPLETED/FAILED.

**Intake leg rules:**

- Each expected returned cylinder requires its own outlet intake confirmation before that cylinder becomes outlet empty
  inventory.
- `INTAKE_PENDING` persists after the customer confirms delivery (PIN). The order can be `DELIVERED` while exchange
  requests are still `INTAKE_PENDING`.
- Financial closure for a refill order is blocked until all exchange requests for that order reach `COMPLETED` or
  `FAILED` with an approved exception (BI-19).
- A `FAILED` intake (wrong cylinder returned, damaged beyond acceptance, or cylinder not returned) creates an inventory
  exception and may supply default support-case context, but no support case exists until a permissioned Customer
  Support Agent, in-scope Outlet Manager, or Super Admin creates it. The failed intake does not by itself undo the
  completed customer delivery.
- During intake, a returned-cylinder intake actor (Inventory Clerk, Outlet Manager, or Super Admin within scope) may
  correct an agent recording mismatch in returned-cylinder vendor or size (e.g., the agent recorded Shell 6kg but the
  actual cylinder received is Total 6kg). The correction requires a reason code, before/after values, and an audit
  trail; material differences require Outlet Manager or Super Admin approval. A correction cannot create an unapproved
  size-mismatch adjustment after delivery; if the physical cylinder size does not match the accepted delivery facts,
  intake follows the `FAILED` intake and approved-exception path.

---

## Stock Count Behaviour (§7.17)

- Launch inventory accountability is aggregate by outlet, vendor, cylinder size, filled status, condition, and item/SKU
  as applicable.
- Individual cylinder serial-number tracking is not required at launch and is not a prerequisite for stock counts,
  reservations, transfers, custody reconciliation, or vendor refill movements.

**Fill lifecycle vs availability:**

- Cylinder fill lifecycle (`FILLED`, `EMPTY`, `IN_REFILL`) is independent from availability and condition status such as
  available, reserved, damaged, quarantined, in transit, lost, sold, or returned.
- Stock is sellable, reservable, or transferable only when both its fill lifecycle and its availability/condition status
  permit that business action.

**Count rules:**

- Stock counts do not freeze outlet operations. When a count begins, expected quantities are fixed as the count-start
  basis.
- Ledger movements (reservations, releases, sales, intake) that occur during the count window are tracked and used to
  calculate variance at count close.
- Orders may continue to be placed, accepted, and fulfilled while a count is in progress.

---

## Low-Stock Alerts (§7.18)

Low-stock alerts are based on available stock only.

**Launch defaults:**

| Stock type                           | Alert threshold                                                |
|--------------------------------------|----------------------------------------------------------------|
| Saleable filled cylinders            | Available quantity at or below 2                               |
| Saleable accessories                 | Available quantity at or below 1                               |
| Empty cylinders (non-saleable stock) | Disabled by default (threshold 0) unless explicitly configured |

These defaults may be overridden per outlet, product, or stock item.

**Rules:**

- Alerts are scoped to the outlet and stock item.
- Repeated alerts for the same outlet/stock item are deduplicated while the item remains below threshold during the
  four-hour launch threshold window.
- Alerts are visible to permissioned Outlet Managers, Area Managers, and Super Admins within their authorized scope.
- Alerts may show relevant `IN_REFILL` and incoming-transfer context, but they do not reserve stock, block orders, alter
  allocation, or forecast demand.

---

## Permissions

Trimmed access matrix rows relevant to inventory. Full matrix: [identity-auth.md](identity-auth.md).

| Capability                      | P-04                              | P-05 | P-06                                          | P-07   | P-08                    | P-10 |
|---------------------------------|-----------------------------------|------|-----------------------------------------------|--------|-------------------------|------|
| Inventory viewing               | Scoped                            | –    | Scoped                                        | Scoped | Read (assigned outlets) | Full |
| Inventory adjustments (submit)  | Scoped (Request)                  | –    | Scoped (Policy-limited post; above = Request) | –      | –                       | Full |
| Inventory adjustments (approve) | –                                 | –    | –                                             | –      | –                       | Full |
| Outlet-to-outlet transfers      | Scoped (Request)                  | –    | Scoped (Request/approve/receive)              | –      | –                       | Full |
| Pick reversal confirmation      | Scoped (with explicit permission) | –    | Scoped (with explicit permission)             | –      | –                       | Full |
| Returned cylinder intake        | Scoped                            | –    | Scoped                                        | –      | –                       | Full |
| Post-delivery return intake     | Scoped (with explicit permission) | –    | Scoped (with explicit permission)             | –      | –                       | Full |
| Vendor refill batch management  | Scoped                            | –    | Scoped                                        | –      | –                       | Full |
| Low-stock alerts                | –                                 | –    | Scoped                                        | –      | Read (assigned outlets) | Full |
| Outlet picking                  | Scoped (with explicit permission) | –    | Scoped (with explicit permission)             | –      | –                       | Full |
