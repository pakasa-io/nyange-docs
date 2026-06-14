# Competitive Outlet Claiming

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Competitive outlet claiming is not part of the mid-stage MVP. MVP order
placement assigns one fulfilling outlet from the delivery area's configured
service assignment. Only that assigned outlet can accept the order for
fulfillment.

## Proposed Scope

If moved back into scope, competitive outlet claiming would need explicit rules
for:

- pending order pool visibility across multiple eligible outlets;
- outlet claim permissions and service-area eligibility;
- concurrent claim attempts and idempotency;
- full pending-order detail exposure before claim;
- claim cancellation that clears the fulfilling outlet;
- returning an order to a shared pending pool;
- outlet ranking, priority, capacity, and workload inputs;
- outlet-change chains and customer acceptance of outlet changes;
- reporting and cash/custody traceability after a claim changes.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes one
fulfillment assignment model.

- A placed order had no fulfilling outlet while it was `PENDING`.
- Any active online-fulfillment outlet that served the delivery area and held
  claim permission could claim a pending order.
- Pending-pool visibility and claim attempts were limited to active
  online-fulfillment outlets that served the delivery area and held claim
  permission.
- Pending-pool reads exposed full delivery address, resolved coordinates,
  recipient name, recipient phone, delivery instructions, order contents,
  customer-visible totals, and serviceability facts.
- Outlets outside the delivery area could not read full pending-pool detail or
  attempt the claim.
- Claiming created exactly one claimed fulfilling outlet association.
- Claiming reserved all requested stock atomically.
- Outlet claim cancellation cleared the fulfilling outlet, released stock, and
  returned the order to the pending pool.
- Customer-cancelled orders never returned to the pending pool.
- Order and inventory permissions used an `Outlet claiming` capability.
- Automatic outlet allocation, outlet ranking, outlet-change chains, and
  customer acceptance of outlet changes were excluded from the prior launch
  lifecycle.

## Why It Is Out of Scope

Competitive claiming adds cross-outlet visibility, concurrency control,
assignment conflict handling, and customer/custody edge cases before the outlet
network requires that complexity.

## Impact of Deferral

- Each online order has one assigned fulfilling outlet at placement.
- Only the assigned outlet can accept the order for fulfillment.
- Other outlets do not see or compete for pending orders.
- Exceptional reassignment remains a Super Admin operational correction.

## Revisit Trigger

Revisit when measured order volume, service-area overlap, or outlet capacity
pressure makes static service-area assignment too manual or too slow for daily
operations.

## Notes

- MVP order behavior is defined in [../business/order.md](../business/order.md).
- MVP reservation behavior is defined in [../business/inventory.md](../business/inventory.md).
