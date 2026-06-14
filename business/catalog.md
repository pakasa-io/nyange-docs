# Catalog & Pricing

**Intent**: Define launch product pricing behavior for online checkout pricing,
refill pricing, bundle pricing, outlet price guardrails, and launch
commercial-program limits.

**Reader task**: Use this document to determine whether a catalog item or refill
combination is orderable, how it is priced, and who can change prices.

**Sources**: §7.2 Refill Pricing Matrix, §7.3 Bundle Pricing, §7.4 Price
Guardrails

**Related**:
[cart.md](cart.md) for cart quote readiness, catalog-change handling, and
checkout readiness;
[order.md](order.md) for immutable order price snapshots;
[delivery.md](delivery.md) for delivery fees and failed-delivery fee waivers;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-24 — New cylinder purchases are outright sales.**

- A customer who buys a new filled cylinder owns the cylinder and gas outright.
- There is no deposit, cylinder-return obligation, customer credit account, or
  customer-level cylinder ownership tracking at launch.
- New filled cylinder purchases may use any globally supported vendor available
  through the online checkout catalog.
- The claiming outlet must fulfill the frozen outgoing vendor, product, and
  quantity terms without changing the placed order total.
- A customer who loses or damages their cylinder must buy a new cylinder rather
  than treat the loss as a refill.

**BI-26 — Launch commercial programs are limited to explicit pricing rules.**

- Customer loyalty points are not supported at launch unless explicitly added to
  launch scope.
- Subscriptions are not supported at launch unless explicitly added to launch
  scope.
- Advanced promotion workflows are not supported at launch unless explicitly
  added to launch scope.
- Customer discounts and price reductions come only from active price rules,
  bundle component pricing, approved adjustments, or another rule named in this
  document.

## Refill Pricing

### Pricing Dimensions

Refill price is determined by three dimensions.

| Dimension | Meaning |
| --- | --- |
| Incoming vendor | The cylinder the customer owns and surrenders |
| Outgoing vendor | The filled cylinder the customer receives |
| Cylinder size | 6kg or 12kg |

### Online Vendor Defaults

- Online COD checkout defaults the outgoing refill vendor to the global online
  outgoing-vendor default.
- At launch, the global online outgoing-vendor default is Vengas.
- Outlet configured default vendors do not apply before order placement.
- The claiming outlet must fulfill the frozen outgoing vendor.

### Supported Vendors

- Refill vendor choices are limited to globally configured supported vendors at
  launch.
- Outlets may request vendor additions through Super Admin escalation.
- The supported-vendor list remains globally managed.
- `Other` and `Unknown` incoming cylinder vendors are not accepted for checkout,
  refill pricing, or doorstep exchange handling.

### Pair Eligibility

- Incoming-to-outgoing refill pair eligibility is global at launch.
- Outlets may restrict which incoming vendors they accept at claim time.
- Outlets do not define outlet-specific incoming-to-outgoing pair overrides.
- A refill pair is orderable online only when the global pair is eligible and
  priceable.
- Online pre-order validation uses coarse serviceability by address: at least one
  active online-fulfillment outlet serves the delivery area.
- Online pre-order validation avoids per-outlet SKU/vendor filtering before
  order placement.
- The claiming outlet must accept the frozen incoming vendor and fulfill the
  frozen outgoing vendor, size, and quantity terms.

### Required Pricing Rules

| Pair type | Required rules |
| --- | --- |
| Same-vendor | Base refill price for outgoing vendor and cylinder size |
| Cross-vendor | Base refill price and a positive cross-vendor surcharge rule for incoming vendor, outgoing vendor, and cylinder size |

- If a base price is missing for the outgoing vendor and size, no refill
  combination using it is orderable.
- If either the base price or the surcharge rule is missing for a cross-vendor
  pair, the pair is not priceable and is not orderable.

### Same-Vendor Availability

- Same-vendor refill availability is a claim-time fulfillment constraint.
- Claim-time same-vendor fulfillment depends on the claiming outlet's actual
  filled stock for that vendor and size.
- Expected vendor-depot returns do not make a same-vendor option fulfillable at
  claim time.
- Cylinders in `IN_REFILL` do not make a same-vendor option fulfillable at claim
  time.

### Multi-Line Orders

- A single order may contain multiple refill exchange lines with different
  incoming vendors or cylinder sizes.
- Each refill line is priced independently using its own incoming vendor,
  outgoing vendor, and cylinder size.
- All-or-nothing fulfillment applies across all lines: every refill in the
  order succeeds together or the whole order fails.

### Pre-Order Filtering

- Vendor options shown to a customer are filtered by global vendor support,
  global pair eligibility, and priceability.
- Coarse serviceability by address must pass before checkout: at least one active
  online-fulfillment outlet serves the delivery area.
- Per-outlet SKU/vendor filtering before order placement is not launch behavior.
- Pre-order orderability does not require claim-time outlet stock or
  outlet-vendor availability.
- Cart quote and checkout readiness revalidate orderability and priceability.
- Pre-order filtering is a usability measure, not a substitute for order placement
  revalidation.

## Online Checkout Pricing Basis

- Online COD cart quotes and order placement use global online catalog, price,
  tax, and delivery-fee rules.
- The eventual claiming outlet cannot change the frozen product price,
  delivery fee, tax, discount, or order total.
- Local outlet price-rule configuration does not affect online COD order totals
  unless an approved global online pricing policy explicitly uses that rule as
  an input before order placement.
- Claim eligibility requires the outlet to fulfill the frozen catalog terms,
  vendor terms, quantities, and delivery address scope.

## Bundle Pricing

Bundle pricing applies to accessories purchased in a bundle with a new cylinder.
The discount is applied at component level: each bundle item has a bundled price
and a standalone price.

### Business Rules

- If the same accessory is purchased without a new cylinder, the standalone
  price applies.
- Bundles are not available for retrofit. A customer cannot add accessories to
  an existing order and retroactively receive bundle pricing.
- Customers may add multiple bundles or multiple quantities of the same bundle
  to one cart.
- Bundles can coexist with new cylinders, refill exchanges, and standalone
  accessories under mixed-cart, single-order, all-or-nothing fulfillment rules.

### Launch Accessories

- Generic accessories are supported as distinct stock items tracked per SKU.
- The confirmed launch accessory set is 2m tubing and cooking grill.
- Regulator accessories require explicit launch approval before being treated
  as sellable.

### Catalog Presentation

- Catalog presentation groupings, such as variants for browsing or filtering,
  do not create separate stock, pricing, order, or reporting truth.
- Stock, pricing, order, and reporting facts attach to the concrete sellable
  item the customer buys.

### Inventory Boundary

- Bundle discounts do not change inventory truth.
- Inventory remains accountable for the physical component items, not the
  bundle's pricing state.
- When a bundle reservation is released, component items return to ordinary
  availability without bundle context.
- Any later sale uses the then-applicable standalone or bundle pricing rule.
- A bundle is a standalone sellable catalog offer and customer-facing bundle,
  but stock reservation, pickup, delivery, and inventory accountability remain
  tied to physical component items.

## Outlet Price Guardrails

Outlet Managers may adjust outlet-specific prices within configured limits.

Outlet-specific price rules are local configuration. They do not affect online
COD cart quotes or placed order totals unless an approved global online pricing
policy explicitly uses them before order placement.

### Guardrail Logic

```
within_guardrail :=
  abs(percentage_change) <= configured_percentage_limit
  AND abs(absolute_change) <= configured_absolute_limit
// both conditions must hold
// basis: active approved outlet rule at proposed effective time,
//        or global/default when no outlet rule exists
```

### Outcomes

- Within-guardrail changes take effect at their scheduled effective time without
  additional approval.
- Outside-guardrail changes require Super Admin approval before activation.
- If daily closing is overdue for the outlet, outside-guardrail activation is
  blocked unless a Super Admin urgent override with reason and note is recorded.

### Launch Defaults

| Price type | Guardrail |
| --- | --- |
| Product, refill, accessory prices | Smaller of 10% or UGX 5,000 from the current approved basis |

Delivery-fee guardrails are defined in [delivery.md](delivery.md).

### Always Requires Super Admin Approval

- Global prices.
- Taxes.
- Delivery-cost rules.
- Guardrail rules.
- Missing-basis price changes.

### General Rules

- All price changes require an effective window and audit record.
- `Immediate` means the effective start is now.
- Price changes are future-dated or now-dated; they are not backdated.
- Tax defaults to 0% for all product and outlet combinations unless an active
  tax rule is configured.
- When a non-zero tax applies, it is part of the customer's order total and is
  frozen into the order's price snapshot at placement.
- Multi-country or jurisdiction-specific tax workflows are not launch behavior
  unless explicitly added to launch scope.

### Cart and Placement Effects

Cart quote and checkout readiness revalidate catalog orderability and
priceability under [cart.md](cart.md). Order placement revalidates the same
requirements before creating the immutable order snapshot under
[order.md](order.md).

## Permissions

Trimmed access matrix rows relevant to catalog and pricing. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-03 | P-06 | P-08 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Product catalog browsing | Read | Read | Read | Read | Full |
| Outlet price rules within guardrail | - | - | Scoped | - | Full |
| Outlet price rules above guardrail | - | - | Request | - | Approve / Full |
| Global pricing and catalog | - | - | - | - | Full |
