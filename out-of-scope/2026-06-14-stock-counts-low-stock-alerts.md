# Advanced Stock Counts and Low-Stock Alerts

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Advanced stock-count behavior and low-stock alerting are not part of the
mid-stage MVP. MVP inventory tracks availability, reservations, transfers,
vendor refill movements, and returned-cylinder intake without automated
threshold alerts or count-window variance handling.

## Proposed Scope

If moved back into scope, advanced stock counts and low-stock alerts would need
explicit rules for:

- count-start expected quantity snapshots;
- ledger movement handling during a count window;
- variance calculation at count close;
- whether counts freeze any outlet operations;
- low-stock thresholds by outlet, product, or stock item;
- alert deduplication windows;
- alert visibility by role and outlet scope;
- notification channels for low-stock alerts;
- `IN_REFILL` and incoming-transfer context in alert views;
- reporting that depends on low-stock signals.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes basic
stock availability and manual inventory review.

- Stock counts did not freeze outlet operations.
- When a count began, expected quantities were fixed as the count-start basis.
- Ledger movements during the count window were tracked and used to calculate
  variance at count close.
- Orders could continue to be placed, claimed, and fulfilled while a count was
  in progress.
- Low-stock alerts were based on available stock only.
- Launch low-stock thresholds were available quantity at or below 2 for saleable
  filled cylinders and at or below 1 for saleable accessories.
- Empty cylinders and non-saleable stock alerts were disabled by default with
  threshold 0 unless explicitly configured.
- Thresholds could be overridden per outlet, product, or stock item.
- Repeated alerts for the same outlet and stock item were deduplicated while the
  item remained below threshold during the four-hour launch window.
- Alerts were visible to permissioned Outlet Managers, Area Managers, and Super
  Admins within authorized scope.
- Alerts could show relevant `IN_REFILL` and incoming-transfer context.
- Low-stock alerts used push notifications.

## Why It Is Out of Scope

Manual inventory review is sufficient for MVP operations. Advanced stock counts
and deduplicated alerts add workflow, notification, and reporting complexity
before stockout frequency proves the need.

## Impact of Deferral

- Staff review inventory directly instead of receiving low-stock alerts.
- Stock corrections continue through inventory adjustment rules.
- No count-window variance workflow is required for MVP.
- No low-stock notification permission or channel assignment is required.

## Revisit Trigger

Revisit when manual inventory review causes repeated stockouts, stale inventory
decisions, avoidable claim-blocked orders, or repeated failed claim attempts.

## Notes

- MVP inventory behavior is defined in [../business/inventory.md](../business/inventory.md).
- MVP notification behavior is defined in [../business/notifications.md](../business/notifications.md).
