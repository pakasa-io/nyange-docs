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
- Customers can order online for outlet delivery or buy through walk-in POS.
- Online orders can mix new cylinder purchases, refill exchanges, and accessory
  purchases in one cart.
- Online delivery orders are cash-on-delivery at launch and are collected by
  the Delivery Agent at the doorstep.
- Walk-in POS sales are immediate cash sales at the outlet.
- Delivery modes are express and batched.
- Walk-in POS sales have no delivery leg.

## Outlet Model

- Each outlet is a company-owned branch within the same legal entity.
- Outlet independence is operational and reporting-oriented, not legal or
  financial separation.
- Each outlet has its own inventory, pricing within guardrails, staff, and cash
  ledger.
- Outlet cash is company cash.
- Outlets are not franchisees or marketplace merchants.
- Outlet stocking is vendor-specific and outlet-owned.
- Outlets may hold filled cylinders from any globally supported vendor.
- Each outlet procures stock directly from vendors.
- There is no central warehouse or central purchasing pool in launch scope.
- An order belongs to exactly one outlet.
- Financial performance is tracked per outlet.

## Refill Exchange Model

- A refill exchange means the customer surrenders an empty cylinder and receives
  a filled cylinder.
- The outgoing cylinder defaults to the outlet's configured default vendor when
  present; otherwise it defaults to the global Vengas default.
- The customer may choose another currently fulfillable outgoing vendor,
  including same-vendor refill when outlet policy allows it and stock exists.
- Refill pricing varies by incoming vendor, outgoing vendor, and cylinder size.

## Aggregate Index

| File | Aggregate | Authoritative for |
| --- | --- | --- |
| [identity-auth.md](identity-auth.md) | Identity & Authorization | Personas, access matrix, authentication, authorization, edge cases E-01-E-10 |
| [catalog.md](catalog.md) | Catalog & Pricing | Refill pricing, bundle pricing, price guardrails, launch commercial programs |
| [order.md](order.md) | Order | Order lifecycle, outlet allocation, cascade and reassignment, cart behavior, COD fulfillment, POS rules |
| [payment.md](payment.md) | Payment | COD cash collection, walk-in cash sales, zero-collection facts, payment boundaries |
| [delivery.md](delivery.md) | Delivery | Delivery lifecycle, delivery fee rules, agent cash handling, failed-delivery fee waiver, cylinder exchange field leg |
| [inventory.md](inventory.md) | Inventory | Reservation lifecycle, outlet transfer, vendor refill batch, cylinder exchange intake leg, stock counts, low-stock alerts |
| [refund.md](refund.md) | Refund | Refund lifecycle, approval thresholds, collection codes, cash payout constraints |
| [finance.md](finance.md) | Finance | Daily closing, expense controls, delivery cost reporting, deferred settlement boundary, forced financial closure |
| [support.md](support.md) | Support | Launch support fallback boundaries and operational risk alerts |
| [notifications.md](notifications.md) | Notifications | Notification channel boundaries and event-to-channel assignments |

## Cross-Aggregate Ownership

- `§6.9 Refill Exchange Request Lifecycle` spans delivery and inventory.
  [delivery.md](delivery.md) owns the field leg from `PENDING` through
  `RETURN_RECORDED`. [inventory.md](inventory.md) owns the intake leg from
  `INTAKE_PENDING` through `COMPLETED` or `FAILED`.
- Deferred customer prepayment workflows, including post-payment outlet
  reassignment settlement, are tracked outside the launch business documents.
- `§7.15 Cart Behaviour` is in [order.md](order.md). [catalog.md](catalog.md)
  references it for catalog-change and price-change effects on open carts.
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
