# Catalog & Pricing

**Intent**: Define launch product pricing behavior for refill pricing, bundle
pricing, outlet price guardrails, and launch commercial-program limits.

**Reader task**: Use this document to determine whether a catalog item or refill
combination is orderable, how it is priced, and who can change prices.

**Sources**: §7.2 Refill Pricing Matrix, §7.3 Bundle Pricing, §7.4 Price
Guardrails

**Related**:
[order.md](order.md) for cart behavior and price-change effects;
[delivery.md](delivery.md) for delivery fees and failed-delivery fee waivers;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-24 — New cylinder purchases are outright sales.**

- A customer who buys a new filled cylinder owns the cylinder and gas outright.
- There is no deposit, cylinder-return obligation, customer credit account, or
  customer-level cylinder ownership tracking at launch.
- New filled cylinder purchases may use any supported vendor available at the
  assigned outlet.
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

### Vendor Defaults

- If the customer does not choose a same-vendor refill, the outgoing vendor uses
  the outlet's configured default when present.
- If no outlet default is configured, the outgoing vendor defaults to global
  Vengas.

### Supported Vendors

- Refill vendor choices are limited to globally configured supported vendors at
  launch.
- Outlets may request vendor additions through Customer Support Agent or Super
  Admin escalation.
- The supported-vendor list remains globally managed.
- `Other` and `Unknown` incoming cylinder vendors are not accepted for checkout,
  refill pricing, or doorstep exchange handling.

### Pair Eligibility

- Incoming-to-outgoing refill pair eligibility is global at launch.
- Outlets may restrict which incoming vendors they accept.
- Outlets do not define outlet-specific incoming-to-outgoing pair overrides.
- A refill pair is orderable for a specific outlet only when the global pair is
  eligible and that outlet accepts the customer's incoming vendor.

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

- Same-vendor refill availability depends on the outlet's actual filled stock
  for that vendor and size.
- Expected vendor-depot returns do not make a same-vendor option orderable.
- Cylinders in `IN_REFILL` do not make a same-vendor option orderable.

### Multi-Line Orders

- A single order may contain multiple refill exchange lines with different
  incoming vendors or cylinder sizes.
- Each refill line is priced independently using its own incoming vendor,
  outgoing vendor, and cylinder size.
- All-or-nothing fulfillment applies across all lines: every refill in the
  order succeeds together or the whole order fails.

### Pre-Checkout Filtering

- Vendor options shown to a customer are filtered to combinations that are
  currently fulfillable for their delivery address and incoming cylinder.
- Options that no eligible outlet can currently fulfill are not shown.
- Checkout revalidates availability and price at order placement.
- The pre-checkout filter is a usability measure, not a substitute for checkout
  revalidation.

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
  but stock reservation, picking, delivery, and inventory accountability remain
  tied to physical component items.

## Outlet Price Guardrails

Outlet Managers may adjust outlet-specific prices within configured limits.

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

### Cart Effects

Catalog or pricing changes affecting cart lines require customer review and
acknowledgement before cart quote or checkout can proceed. See
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
