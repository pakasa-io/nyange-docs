# Cart

**Intent**: Define customer cart behavior, checkout readiness, cart quote
semantics, catalog-change handling, and abandoned-cart cleanup.

**Reader task**: Use this document to decide whether a customer cart can be
created, changed, quoted, checked out, abandoned, or cleaned up.

**Sources**: §7.15 Cart Behaviour

**Related**:
[catalog.md](catalog.md) for orderable products, refill combinations, bundle
pricing, and price rules;
[order.md](order.md) for order placement and immutable order snapshots;
[inventory.md](inventory.md) for reservation behavior after outlet claim;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Boundary

- Cart owns customer-selected pre-order shopping state.
- Cart owns cart line contents, quantities, selected delivery address reference,
  customer acknowledgements for catalog or price changes, quote readiness, and
  abandoned-cart cleanup state.
- Cart does not own order lifecycle state, outlet claim state, stock
  reservation, payment facts, delivery tasks, receipts, refunds, inventory
  ledgers, or audit ledgers.
- Checkout is a command from a checkout-ready cart to Order.
- A successful checkout creates a new order in `PENDING`.
- A failed checkout leaves the cart active for correction unless the customer
  explicitly clears it.

## Cart Lifecycle

```
ACTIVE -> CHECKED_OUT
ACTIVE -> CLEARED
ACTIVE -> ABANDONED -> SUMMARY_RETAINED
```

| State | Meaning |
| --- | --- |
| `ACTIVE` | Customer can view, change, quote, and prepare the cart for checkout. |
| `CHECKED_OUT` | Cart produced a placed order and is no longer mutable as active cart detail. |
| `ABANDONED` | Cart has had no customer fetch, mutation, or quote activity for the configured inactivity period. |
| `SUMMARY_RETAINED` | Abandoned cart item detail is no longer retained as active cart detail; safe summary remains. |
| `CLEARED` | Customer explicitly removed active cart contents. |

Terminal cart states: `CHECKED_OUT`, `SUMMARY_RETAINED`, `CLEARED`.

## Business Rules

### Account Requirement

- Cart creation, cart changes, quotes, and checkout require an authenticated
  customer account.
- Anonymous guest cart and checkout are not launch behavior.
- Carts persist across customer sessions and devices for the authenticated
  customer.

### Cart Contents

- A cart may contain new cylinder purchases, refill exchange lines, accessories,
  and bundles.
- A cart may contain multiple refill exchange lines with different incoming
  vendors or cylinder sizes.
- A cart may contain multiple bundles or multiple quantities of the same bundle.
- Cart contents are customer-derived commerce state.
- Cart contents are not durable order, payment, ledger, custody, receipt, refund,
  or audit facts.

### Pre-Order Orderability

- Disabled products are not orderable or sellable, even if stock exists.
- Any cart item referencing a disabled product or invalid bundle is marked not
  orderable.
- The customer must resolve not-orderable items before checkout can proceed.
- Already-placed orders referencing a disabled product are not affected by later
  disablement.
- Pre-order orderability uses enabled catalog items, valid bundles, global
  refill-pair eligibility, priceability, and coarse serviceability by address.
- Pre-order orderability does not use per-outlet SKU/vendor stock filtering.
- Outlet stock and vendor fulfillment constraints are enforced at claim time
  after order placement.
- If claim-time outlet stock or vendor constraints prevent any eligible outlet
  from claiming a placed order, Order owns the `CLAIM_BLOCKED` and
  `UNCLAIMABLE` outcomes.
- A claim-blocked or unclaimable order does not reopen the checked-out cart.

### Quote Semantics

- Cart and checkout quote prices are estimates until order placement.
- Quote generation is server-computed.
- Client-submitted line totals, fees, taxes, discounts, and order totals are
  ignored.
- Quote generation evaluates current catalog, global online pricing,
  address-based delivery-fee, tax, and pre-order orderability rules.
- Quote generation uses coarse serviceability by address: at least one active
  online-fulfillment outlet serves the delivery area.
- Quote generation avoids per-outlet SKU/vendor filtering before order
  placement.
- Pricing is locked only when Order creates the immutable order snapshot.
- No stock is reserved by cart creation, cart update, quote generation, or
  checkout readiness.

### Catalog and Price Changes

- Catalog or pricing changes affecting cart lines require customer review and
  valid line-level acknowledgement before quote readiness or checkout readiness
  can pass.
- Catalog or pricing changes include price changes, product disablement, and
  bundle composition changes.
- If a product's price changes while a cart is active, cart quotes use the
  current price and the customer receives an explicit price-change notice before
  checkout.
- If a bundle's composition changes while a cart is active, any cart line
  containing that bundle is flagged invalid.
- The customer must review and acknowledge the bundle change before quote or
  checkout can proceed.
- Cart owns catalog-change acknowledgement records for active cart lines.
- Each acknowledgement record stores affected cart line ID, change reason,
  Catalog-owned catalog item version, Catalog-owned price rule version, bundle
  composition version when applicable, acknowledged line quantity,
  acknowledged server-computed line price, customer Identity account ID, and
  acknowledgement timestamp.
- A catalog-change acknowledgement is valid only for the exact affected line,
  version tuple, quantity, and server-computed line price recorded in the
  acknowledgement.
- Any later catalog item version, price rule version, bundle composition
  version, line quantity, line composition, or server-computed line price change
  invalidates the acknowledgement and requires customer re-acknowledgement.
- Acknowledgement does not make a disabled, unorderable, or unpriceable line
  valid. The customer must remove or change the line before quote readiness or
  checkout readiness can pass.

### Delivery Address Readiness

- A cart may reference a customer delivery address.
- Checkout readiness requires the selected delivery address to have resolved
  coordinates.
- An unresolved delivery address blocks checkout until coordinates are resolved
  or the customer selects a different address.
- Cart may store unresolved address selection as customer preference state, but
  it cannot make the cart checkout-ready.

## Checkout Readiness

A cart is checkout-ready only when:

- the customer is authenticated;
- at least one cart line exists;
- every cart line references an enabled product or valid bundle;
- every cart line is orderable under pre-order orderability rules;
- every catalog combination is priceable;
- the selected delivery address has resolved coordinates;
- coarse serviceability by address is satisfied;
- required catalog-change and price-change acknowledgements are complete;
- the server can compute all line totals and order totals.

Checkout sends the validated cart contents to Order. Order revalidates the same
business requirements before creating the order snapshot.

## Abandoned-Cart Cleanup

- A cart with no customer fetch, mutation, or quote activity for 90 days is
  marked `ABANDONED`.
- After an additional 30 days, abandoned cart item detail is no longer retained
  as active cart detail.
- A safe cart summary remains.
- Checked-out carts and placed orders are never part of abandoned-cart cleanup.
- Claim-blocked or unclaimable placed orders are Order lifecycle outcomes, not
  abandoned-cart cleanup outcomes.

## Permissions

Cart owns authorization decisions for Cart-owned commands and reads. Related
rows are shown here for context; rows enforced by another aggregate remain with
that aggregate. Identity supplies actor, account, grant, and outlet-scope facts
from [identity-auth.md](identity-auth.md).

| Capability | P-01 | P-10 |
| --- | --- | --- |
| Cart creation and update | Full | Full |
| Cart quote | Full | Full |
| Cart checkout | Full | Full |
| Cart cleanup policy administration | - | Full |
