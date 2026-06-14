# Nyange Business Domain

**Intent**: Orient agents and contributors to the Nyange business domain
documents and identify the authoritative file for each aggregate.

**Reader task**: Use this file to find the correct business document before
changing requirements, API contracts, module specs, implementation plans, or
tests.

**Version**: 2026-05-24

## Facts

- Nyange distributes gas cylinders, refills, and accessories through
  company-owned outlets.
- Supported launch cylinder sizes are 6kg and 12kg.
- Customers can order online for outlet delivery.
- Customer carts can mix new cylinder purchases, refill exchanges, and
  accessory purchases before order placement.
- Online delivery orders are cash-on-delivery at launch and are collected by
  the Delivery Agent at the doorstep.
- Online orders create single-order delivery tasks.

## Outlet Model

- Each outlet is a company-owned branch within the same legal entity.
- Outlet independence is operational and reporting-oriented, not legal or
  financial separation.
- Each outlet has its own inventory, staff, and cash ledger.
- Online COD checkout uses the global online pricing and address-based delivery
  fee basis.
- Coarse serviceability by address means at least one active online-fulfillment
  outlet serves the delivery area.
- Pre-order validation avoids per-outlet SKU/vendor filtering before order
  placement.
- Pending-pool visibility and claim attempts are limited to active
  online-fulfillment outlets that serve the delivery area and hold claim
  permission.
- Outlet cash is company cash.
- Outlets are not franchisees or marketplace merchants.
- Outlet stocking is vendor-specific and outlet-owned.
- Outlets may hold filled cylinders from any globally supported vendor.
- Each outlet procures stock directly from vendors.
- There is no central warehouse or central purchasing pool in launch scope.
- A placed order has no fulfilling outlet while it is `PENDING`.
- A placed order can move to `CLAIM_BLOCKED` when the pending-claim timeout
  expires or every eligible outlet has a current claim-blocking reason.
- A `CLAIM_BLOCKED` order has no fulfilling outlet and no stock reservation.
- `CLAIM_BLOCKED` can reopen to `PENDING`, be customer-cancelled, or close as
  terminal `UNCLAIMABLE`.
- Claiming creates exactly one claimed fulfilling outlet for the order.
- Outlet claim cancellation clears the fulfilling outlet and returns the order to
  the pending pool.
- Successful delivery and failed delivery retain the fulfilling outlet for
  reporting and cash/custody traceability.
- Financial performance is tracked per outlet.

## Refill Exchange Model

- A refill exchange means the customer surrenders an empty cylinder and receives
  a filled cylinder.
- Online COD checkout defaults the outgoing cylinder to the global online
  outgoing-vendor default, Vengas at launch.
- Outlet configured default vendors do not apply before order placement.
- If launch checkout exposes outgoing-vendor choice, the customer may choose
  another globally supported and priceable outgoing vendor.
- Pre-order outgoing-vendor choices do not depend on per-outlet vendor policy or
  stock filtering.
- Refill pricing varies by incoming vendor, outgoing vendor, and cylinder size.

## Aggregate Index

| File | Aggregate | Authoritative for |
| --- | --- | --- |
| [identity-auth.md](identity-auth.md) | Identity & Authorization | Personas, access matrix, authentication, authorization, edge cases E-02-E-09 |
| [catalog.md](catalog.md) | Catalog & Pricing | Refill pricing, bundle pricing, global online pricing, launch commercial programs |
| [cart.md](cart.md) | Cart | Customer cart state, quote readiness, catalog-change handling, checkout readiness, abandoned-cart cleanup |
| [order.md](order.md) | Order | Order placement, lifecycle state, outlet claiming, COD fulfillment, and cancellation |
| [payment.md](payment.md) | Payment | COD cash collection, zero-collection facts, payment boundaries |
| [delivery.md](delivery.md) | Delivery | Delivery lifecycle, delivery fee rules, agent cash handling, failed-delivery fee waiver, cylinder exchange field leg |
| [inventory.md](inventory.md) | Inventory | Reservation lifecycle, outlet transfer, vendor refill batch, cylinder exchange intake leg, stock availability |
| [refund.md](refund.md) | Refund | Refund lifecycle, collection codes, cash payout constraints |
| [finance.md](finance.md) | Finance | Daily closing, expense controls, delivery cost reporting, receipts, and cash custody reporting |
| [notifications.md](notifications.md) | Notifications | Notification channel boundaries and event-to-channel assignments |

## Cross-Aggregate Ownership

- `§6.9 Refill Exchange Request Lifecycle` spans delivery and inventory.
  [delivery.md](delivery.md) owns the field leg from `PENDING` through
  `RETURN_RECORDED`, plus Delivery-owned field-leg `FAILED`.
  [inventory.md](inventory.md) owns the intake leg from `INTAKE_PENDING` through
  `INTAKE_CONFIRMED` or Inventory-owned intake `FAILED`.
- Cart owns customer-selected pre-order state and checkout readiness.
  [order.md](order.md) owns order placement and the immutable order lifecycle
  after checkout succeeds.
- Catalog orderability and priceability rules are referenced by [cart.md](cart.md)
  for quote readiness and by [order.md](order.md) for placement revalidation.
- The complete access matrix is in [identity-auth.md](identity-auth.md). Each
  aggregate file includes only the permission rows directly relevant to that
  aggregate.

## Use Rules

- Treat the aggregate file listed above as authoritative for its business
  lifecycle.
- Do not move mutable lifecycle ownership across files without updating this
  index and the related aggregate documents together.
- Mark unresolved business facts as open questions in the owning aggregate
  document instead of inventing missing policy.
