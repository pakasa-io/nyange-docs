# Express Delivery Fee

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business review finding on unspecified express delivery fee behavior
- Owner: Product Manager

## Summary

Express delivery fee behavior is deferred from the mid-stage MVP. Launch
delivery uses the standard global online address-based delivery fee only.

## Proposed Scope

If moved back into scope, express delivery fee behavior may include:

- customer-facing express delivery selection during checkout;
- express eligibility by service area, outlet capacity, operating hours, and
  delivery workload;
- express fee calculation, multiplier rules, guardrails, and approval policy;
- operational priority, dispatch behavior, and delivery promise;
- failed-promise waiver, refund, or adjustment rules;
- reporting for express revenue, delivery cost, and service performance;
- authorization rules for configuring express delivery policy.

## Why It Is Out of Scope

The current MVP does not define a timed or priority delivery promise, outlet
capacity model, dispatch priority rule, or failed-promise outcome. Adding an
express fee multiplier without those rules would create customer-visible pricing
without an implementation-ready fulfillment contract.

## Impact of Deferral

- Customers cannot buy express delivery at launch.
- Online COD orders use standard delivery fee rules.
- Outlet Managers cannot configure express-fee multipliers.
- Delivery assignment and routing remain ordinary single-order delivery flows.

## Revisit Trigger

Revisit when the Product Manager approves a customer-facing priority delivery
offer with service-level promise, eligibility, capacity, fee, refund or waiver,
reporting, and authorization rules.

## Notes

- Related MVP delivery rules remain in [../business/delivery.md](../business/delivery.md).
- Authorization must be updated before any express-fee configuration permission
  enters launch scope.
