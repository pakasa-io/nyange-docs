# Outlet-Local Pricing and Guardrails

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Outlet-local product, refill, accessory, and delivery-fee price configuration is
not part of the mid-stage MVP. MVP online orders use global online pricing and
address-based delivery-fee rules.

## Proposed Scope

If moved back into scope, outlet-local pricing and guardrails would need explicit
rules for:

- outlet-scoped product, refill, accessory, and delivery-fee rules;
- percentage and absolute guardrail thresholds;
- within-threshold activation without separate approval;
- above-threshold Super Admin approval;
- missing-basis price change handling;
- overdue daily-closing blocks and urgency overrides;
- historical pricing basis and effective windows;
- cart quote and order placement behavior when local prices differ from global
  online prices;
- authorization rows for outlet-local price changes;
- audit records for local pricing changes and approvals.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes one
pricing model: global online pricing.

- Each outlet had local price-rule configuration within guardrails.
- Online COD order totals ignored local outlet price rules unless a future
  global online pricing policy used those rules as an input.
- Outlet Managers could change outlet-scoped product, refill, accessory, and
  delivery-fee prices within configured guardrails.
- Product, refill, and accessory price changes were within guardrail only when
  both the percentage change and absolute change were within the configured
  limits.
- The launch guardrail for product, refill, and accessory prices was the smaller
  of 10% or UGX 5,000 from the current approved basis.
- The launch guardrail for delivery-fee changes was the smaller of 15% or UGX
  2,000 from the current approved basis.
- Outside-guardrail changes required Super Admin approval before activation.
- Outside-guardrail activation was blocked when daily closing was overdue unless
  a Super Admin urgent override with reason and note was recorded.
- Global prices, taxes, delivery-cost rules, guardrail rules, and missing-basis
  price changes always required Super Admin approval.
- Finance overdue-closing policy included separate blocks for
  outside-guardrail price changes while allowing within-guardrail price changes.

## Why It Is Out of Scope

Outlet-local price configuration adds approval workflows, effective-dated local
rules, overdue-closing gates, and authorization complexity while MVP online COD
orders can run on global online pricing.

## Impact of Deferral

- Outlet Managers cannot configure local prices in the MVP.
- Online COD checkout uses global online product, refill, accessory, tax, and
  delivery-fee rules.
- Super Admin manages global catalog and pricing changes.
- No outlet-local price approval workflow is required for MVP operations.

## Revisit Trigger

Revisit when multiple active outlets need independently managed local price
schedules that cannot be handled through global online pricing.

## Notes

- MVP pricing behavior is defined in [../business/catalog.md](../business/catalog.md).
- MVP delivery-fee behavior is defined in [../business/delivery.md](../business/delivery.md).
