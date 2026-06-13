# Nyange Business Domain

**Intent**: Navigation index and one-page domain overview for the Nyange platform business specification.

**Version**: 2026-05-24

---

## Domain in One Page

Nyange distributes gas cylinders (6kg, 12kg) and accessories through company-owned outlets. Customers can order online for delivery from the nearest eligible outlet or buy through walk-in POS at an outlet. Orders are either new cylinder purchases, cylinder refill exchanges, or accessory purchases, and online orders can mix them in one cart.

**Two payment rails:** mobile money (pre-pay, staff-verified against provider reference) and cash on delivery (collected at doorstep by agent).

**Two delivery modes:** express (as soon as possible) and batched (customer selects a delivery window). Walk-in POS sales have no delivery leg.

**Refill exchange:** customer surrenders an empty cylinder and receives a filled one. The outgoing cylinder defaults to the outlet's configured default vendor when present, otherwise to the global Vengas default. The customer may choose another currently fulfillable outgoing vendor, including same-vendor where outlet policy allows it and the outlet has stock. Pricing varies by incoming vendor, outgoing vendor, and cylinder size.

**Outlet independence:** each outlet is a company-owned branch within the same legal entity, operating as an outlet-level accountability unit — its own inventory, pricing (within guardrails), staff, payment accounts, and cash ledger. Outlet independence is operational and reporting-oriented, not legal or financial separation: outlet cash is company cash, and outlets are not franchisees or marketplace merchants. Outlet stocking is vendor-specific and outlet-owned: outlets may hold filled cylinders from any globally supported vendor, not only Vengas, and each outlet procures stock directly from vendors rather than from a central warehouse or central purchasing pool. An order always belongs to exactly one outlet. Financial performance is tracked per outlet.

---

## Aggregate Index

| File | Aggregate | Key Domains |
|---|---|---|
| [identity-auth.md](identity-auth.md) | Identity & Authorization | Personas, access matrix, auth model, edge cases E-01–E-10 |
| [catalog.md](catalog.md) | Catalog & Pricing | Refill pricing, bundle pricing, price guardrails, express delivery fee |
| [order.md](order.md) | Order | Order lifecycle, outlet allocation, cascade & reassignment, cart behaviour, mobile money expiry |
| [payment.md](payment.md) | Payment | Payment lifecycle, late payment references |
| [delivery.md](delivery.md) | Delivery | Delivery lifecycle, agent cash handling, failed delivery fee waiver, cylinder exchange field leg |
| [inventory.md](inventory.md) | Inventory | Reservation lifecycle, outlet transfer, vendor refill batch, cylinder exchange intake leg, stock counts, low-stock alerts |
| [refund.md](refund.md) | Refund | Refund lifecycle, collection codes |
| [finance.md](finance.md) | Finance | Daily closing, expense controls, delivery cost reporting, settlements, forced closure |
| [support.md](support.md) | Support | Support case lifecycle, operational risk alerts |
| [notifications.md](notifications.md) | Notifications | Notification channel boundaries |

---

## Cross-Aggregate Notes

- **§6.9 Refill Exchange Request Lifecycle** spans delivery and inventory. The field leg (PENDING → RETURN_RECORDED, doorstep recording) is in [delivery.md](delivery.md). The intake leg (INTAKE_PENDING → COMPLETED/FAILED, outlet intake confirmation) is in [inventory.md](inventory.md). Both files cross-reference each other.
- **F-04 Post-Payment Outlet Reassignment** is in [finance.md](finance.md) (it is primarily a settlement flow) with cross-references from [order.md](order.md) and [payment.md](payment.md).
- **§7.15 Cart Behaviour** is in [order.md](order.md) with a cross-reference in [catalog.md](catalog.md) for catalog-change and price-change cart effects.
- The full access matrix lives in [identity-auth.md](identity-auth.md). Each aggregate file includes only the rows directly relevant to its domain.
