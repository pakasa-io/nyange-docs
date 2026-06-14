# Catalog & Pricing

**Intent**: Define launch product pricing behavior for online checkout pricing,
refill pricing, bundle pricing, global online pricing, and launch
commercial-program limits.

**Reader task**: Use this document to determine whether a catalog item or refill
combination is orderable, how it is priced, and who can change prices.

**Sources**: §7.2 Refill Pricing Matrix, §7.3 Bundle Pricing

**Related**:
[cart.md](cart.md) for cart quote readiness, catalog-change handling, and
checkout readiness;
[order.md](order.md) for immutable order price snapshots;
[pos.md](pos.md) for walk-in POS sale line snapshots and saleability checks;
[delivery.md](delivery.md) for delivery fees and failed-delivery fee waivers;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Invariants

**BI-24 — New cylinder purchases are outright sales.**

- A customer who buys a new filled cylinder owns the cylinder and gas outright.
- There is no deposit, cylinder-return obligation, customer credit account, or
  customer-level cylinder ownership tracking at launch.
- New filled cylinder purchases may use any globally supported vendor available
  through the online checkout catalog or walk-in POS catalog.
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
- If every eligible outlet has a current claim-blocking vendor reason for the
  frozen order terms, Order owns the `CLAIM_BLOCKED` and `UNCLAIMABLE`
  lifecycle outcomes.

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
- Claim eligibility requires the outlet to fulfill the frozen catalog terms,
  vendor terms, quantities, and delivery address scope.
- Claim-blocked and unclaimable outcomes do not rewrite catalog rules, price
  rules, or the placed order snapshot.

## POS Pricing Basis

- Walk-in POS uses the same active launch catalog, product, refill, accessory,
  bundle, tax, and priceability rules as online checkout.
- POS does not use delivery-fee rules.
- POS does not use address serviceability or pre-order outlet claiming.
- POS may complete only for stock that is available and saleable at the selling
  outlet at completion time.
- POS refill exchange line pricing uses the recorded incoming vendor, selected
  outgoing vendor, and cylinder size under the active global refill pricing
  matrix.
- Outlet Cashiers cannot override POS line prices, tax, discounts, or totals.
- Outlet-local POS product, refill, accessory, or tax price overrides are not
  launch behavior.

## Price Administration

- Global launch catalog, product, refill, accessory, tax, and delivery-fee rules
  are the launch pricing basis.
- Only Super Admin may create, change, disable, or retire global catalog and
  pricing rules.
- Outlet-local product, refill, accessory, and delivery-fee price configuration
  is not launch behavior.
- Outlet Managers cannot create, change, approve, or apply outlet-local price
  rules or delivery-fee overrides at launch.
- Online COD quotes, order placement, and POS sale completion never consult
  outlet-local price rules at launch.
- Outlet-local pricing guardrails and local delivery-fee overrides are deferred
  in
  [../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md](../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md).
- All price changes require an effective window and audit record.
- `Immediate` means the effective start is now.
- Price changes are future-dated or now-dated; they are not backdated.
- Tax defaults to 0% for all product combinations unless an active tax rule is
  configured.
- When a non-zero tax applies, it is part of the customer's order total and is
  frozen into the order's price snapshot at placement.
- Multi-country or jurisdiction-specific tax workflows are not launch behavior
  unless explicitly added to launch scope.

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

## Cart and Placement Effects

Cart quote and checkout readiness revalidate catalog orderability and
priceability under [cart.md](cart.md). Order placement revalidates the same
requirements before creating the immutable order snapshot under
[order.md](order.md).

Catalog supplies the authoritative version facts used by Cart catalog-change
acknowledgements:

- catalog item version;
- price rule version;
- bundle composition version when the cart line references a bundle;
- effective window for the versioned catalog or price fact.

Any Catalog-owned change that can affect cart orderability, priceability, line
price, or bundle composition creates a new version fact. Cart acknowledgements
must reference the exact version facts reviewed by the customer.

## Permissions

Catalog owns authorization decisions for Catalog-owned catalog and pricing
commands and reads. Related rows are shown here for context; rows enforced by
another aggregate remain with that aggregate. Identity supplies actor, account,
grant, and outlet-scope facts from [identity-auth.md](identity-auth.md).

| Capability | P-01 | P-03 | P-06 | P-08 | P-10 |
| --- | --- | --- | --- | --- | --- |
| Product catalog browsing | Read | Read | Read | Read | Full |
| Global pricing & catalog | - | - | - | - | Full |
