# Catalog & Pricing

**Intent**: Define the refill pricing matrix, bundle pricing rules, and outlet price guardrails for product prices on the Nyange platform.

**Sources**: §7.2 Refill Pricing Matrix, §7.3 Bundle Pricing, §7.4 Price Guardrails

**Related**: [delivery.md](delivery.md) — express delivery fee and fee waiver rules (§7.5, §7.12); [order.md](order.md) — cart behaviour and price-change effects on open carts (§7.15); [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-26 — Launch commercial programs are limited to explicit pricing rules.**
Customer loyalty points, subscriptions, and advanced-promotion workflows are not supported at launch unless explicitly
added to launch scope. Customer discounts and price reductions come only from active price rules, bundle component
pricing, approved adjustments, or other rules named in this document.

**BI-24 — New cylinder purchases are outright sales.**
A customer who buys a new filled cylinder owns the cylinder and gas outright. There is no deposit, cylinder-return
obligation, customer credit account, or customer-level cylinder ownership tracking at launch. New filled cylinder
purchases may use any supported vendor available at the assigned outlet; a customer who loses or damages their cylinder
must buy a new cylinder rather than treat the loss as a refill.

---

## Refill Pricing Matrix

Refill price is determined by three dimensions:

| Dimension       | Description                                   |
|-----------------|-----------------------------------------------|
| Incoming vendor | The cylinder the customer owns and surrenders |
| Outgoing vendor | The cylinder the customer will receive        |
| Cylinder size   | 6kg or 12kg                                   |

**Vendor defaults:**

- If the customer does not choose a same-vendor refill, the outgoing vendor uses the outlet's configured default where
  present, otherwise the global Vengas default.

**Supported vendors:**

- At launch, refill vendor choices are limited to globally configured supported vendors.
- Outlets may request vendor additions through Customer Support Agent or Super Admin escalation, but the
  supported-vendor list remains globally managed.
- `Other` or `Unknown` incoming cylinder vendors are not accepted for checkout, refill pricing, or doorstep exchange
  handling.

**Pair eligibility:**

- Incoming-to-outgoing refill pair eligibility is global at launch. Outlets may restrict which incoming vendors they
  accept, but they do not define outlet-specific incoming-to-outgoing pair overrides.
- A refill pair is orderable for a specific outlet only when the global pair is eligible and that outlet accepts the
  customer's incoming vendor.

**Pricing requirements by pair type:**

| Pair type                                | Required rules                                                                                                               |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Same-vendor (e.g., Shell in → Shell out) | Base refill price for outgoing vendor and cylinder size                                                                      |
| Cross-vendor (incoming ≠ outgoing)       | Base refill price **and** a positive cross-vendor surcharge rule for the incoming vendor, outgoing vendor, and cylinder size |

- If a base price is missing for the outgoing vendor and size, no refill combination using it is orderable.
- If either the base price or the surcharge rule is missing for a cross-vendor pair, the pair is not priceable and
  therefore not orderable.

**Same-vendor availability constraint:**

- Same-vendor availability is constrained by the outlet's actual filled stock for that vendor and size.
- Expected vendor-depot returns or cylinders in `IN_REFILL` do not make a same-vendor option orderable.

**Multi-line orders:**

- A single order may contain multiple refill exchange lines with different incoming vendors or cylinder sizes (e.g., one
  Shell 6kg refill and one Total 12kg refill in the same cart).
- Each line is priced independently using its own pricing triplet.
- All-or-nothing semantics apply across all lines: every refill in the order succeeds together or the whole order fails.

**Pre-checkout filtering:**

- Vendor options presented to a customer during refill selection are filtered to combinations that are currently
  fulfillable for their delivery address and incoming cylinder.
- Options that no eligible outlet can currently fulfil are not shown.
- Checkout revalidates availability and price at the moment of order placement as a second gate; the pre-checkout filter
  is a usability measure, not a substitute for revalidation.

---

## Bundle Pricing

Accessories purchased in a bundle with a new cylinder receive a discounted price. The discount is applied at the
component level: each bundle item has a bundled price and a standalone price.

**Rules:**

- If the same accessory is purchased without a new cylinder (accessory-only order), the standalone price applies.
- Bundles are not available for retrofit: a customer cannot add accessories to an existing order and retroactively
  receive bundle pricing.
- Customers may add multiple bundles or multiple quantities of the same bundle to one cart.
- Bundles can coexist with new cylinders, refill exchanges, and standalone accessories under the mixed-cart,
  single-order, all-or-nothing fulfillment rules.

**Accessories at launch:**

- At launch, generic accessories are supported as distinct stock items tracked per SKU.
- The confirmed launch accessory set is 2m tubing and cooking grill.
- Regulator accessories require explicit launch approval before being treated as sellable.

**Catalog presentation:**

- Catalog presentation groupings, such as variants used for browsing or filtering, do not create separate stock,
  pricing, order, or reporting truth. Those business facts attach to the concrete sellable item the customer buys.

**Inventory truth:**

- Bundle discounts do not change inventory truth. Inventory remains accountable for the physical component items, not
  the bundle's pricing state.
- When a bundle reservation is released, component items return to ordinary availability without bundle context; any
  later sale uses the then-applicable standalone or bundle pricing rule.
- A bundle is a standalone sellable catalog offer and customer-facing bundle, but stock reservation, picking, delivery,
  and inventory accountability remain tied to its physical component items.

---

## Outlet Price Guardrails

Outlet Managers may adjust outlet-specific prices within bounded limits.

**Within-guardrail conditions:**

```
within_guardrail :=
  abs(percentage_change) <= configured_percentage_limit
  AND abs(absolute_change) <= configured_absolute_limit
// both conditions must hold
// basis: active approved outlet rule at proposed effective time, or global/default when no outlet rule exists
```

**Outcomes:**

- Within-guardrail changes take effect at their scheduled effective time without additional approval.
- Outside-guardrail changes require Super Admin approval before activation.
- If daily closing is overdue for the outlet, outside-guardrail activation is blocked unless a Super Admin urgent
  override with reason and note is recorded.

**Launch defaults:**

| Price type                        | Guardrail                                                    |
|-----------------------------------|--------------------------------------------------------------|
| Product, refill, accessory prices | Smaller of 10% or UGX 5,000 from the current approved basis |

Delivery-fee guardrail: see [delivery.md](delivery.md).

**Always require Super Admin approval regardless of amount:**

- Global prices
- Taxes
- Delivery-cost rules
- Guardrail rules
- Missing-basis price changes

**General rules:**

- All price changes must have an effective window and audit record.
- "Immediate" means the effective start is now. All changes are future-dated or now-dated; no backdating.
- Tax defaults to 0% for all product and outlet combinations unless an active tax rule is configured.
- When a non-zero tax applies, it is part of the customer's order total and is frozen into the order's price snapshot at
  placement.
- Multi-country or jurisdiction-specific tax workflows are not launch behavior unless explicitly added to launch scope.

**Cart effects of pricing changes:** see [order.md §7.15](order.md) — catalog or pricing changes affecting cart lines
require customer review and acknowledgement before cart quote or checkout can proceed.

## Permissions

Trimmed access matrix rows relevant to catalog and pricing. Full matrix: [identity-auth.md](identity-auth.md).

| Capability                            | P-01 | P-03 | P-06    | P-08 | P-10           |
|---------------------------------------|------|------|---------|------|----------------|
| Product catalog browsing              | Read | Read | Read    | Read | Full           |
| Outlet price rules (within guardrail) | –    | –    | Scoped  | –    | Full           |
| Outlet price rules (above guardrail)  | –    | –    | Request | –    | Approve / Full |
| Global pricing & catalog              | –    | –    | –       | –    | Full           |
