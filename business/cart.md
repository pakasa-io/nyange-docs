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
[identity-auth.md](identity-auth.md) for the full access matrix.

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
ACTIVE
  -> CHECKED_OUT
  -> ABANDONED
      -> SUMMARY_RETAINED

CLEARED
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

### Availability

- Out-of-stock cart items are marked unavailable and remain in the cart until
  the customer removes them or stock returns.
- Disabled products are not orderable or sellable, even if stock exists.
- Any cart item referencing a disabled product is marked unavailable.
- The customer must resolve unavailable items before checkout can proceed.
- Already-placed orders referencing a disabled product are not affected by later
  disablement.

### Quote Semantics

- Cart and checkout quote prices are estimates until order placement.
- Quote generation is server-computed.
- Client-submitted line totals, fees, taxes, discounts, and order totals are
  ignored.
- Quote generation evaluates current catalog, pricing, delivery-fee, tax, and
  availability rules.
- Pricing is locked only when Order creates the immutable order snapshot.
- No stock is reserved by cart creation, cart update, quote generation, or
  checkout readiness.

### Catalog and Price Changes

- Catalog or pricing changes affecting cart lines require customer review and
  acknowledgement before quote or checkout can proceed.
- Catalog or pricing changes include price changes, product disablement, and
  bundle composition changes.
- If a product's price changes while a cart is active, cart quotes use the
  current price and the customer receives an explicit price-change notice before
  checkout.
- If a bundle's composition changes while a cart is active, any cart line
  containing that bundle is flagged invalid.
- The customer must review and acknowledge the bundle change before quote or
  checkout can proceed.

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
- every cart line is currently available;
- every catalog combination is priceable;
- the selected delivery address has resolved coordinates;
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

## Permissions

Trimmed access matrix rows relevant to carts. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-10 |
| --- | --- | --- |
| Cart creation and update | Full | Full |
| Cart quote | Full | Full |
| Cart checkout | Full | Full |
| Cart cleanup policy administration | - | Full |
