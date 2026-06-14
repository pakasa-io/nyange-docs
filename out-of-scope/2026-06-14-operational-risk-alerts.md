# Operational Risk Alerts

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business review resolution to remove unresolved risk-alert lifecycle
  from MVP scope
- Owner: Product Manager

## Summary

Operational risk alerts are deferred from the mid-stage MVP. Launch operations
use direct owning-domain workflows, ordinary audit logs, cash variance records,
custody exception records, inventory adjustments, and operational escalation
without a separate risk-alert lifecycle.

## Proposed Scope

If moved back into scope, operational risk alerts may include:

- rolling-window thresholds for repeated custody, cash, delivery, inventory, or
  adjustment exceptions;
- alert records with explicit ownership, lifecycle states, terminality, and
  audit requirements;
- scoped visibility for permissioned Outlet Managers and Super Admins;
- notification rules for configured alert types;
- manual acknowledgement, dismissal, and escalation workflows;
- linkage to owning-domain reviews, personnel review, or support case
  management if that also enters scope;
- retention rules for alert records and derived summaries.

## Why It Is Out of Scope

The current MVP does not need a separate derived alert lifecycle to prove core
order, delivery, inventory, cash, refund, and support fallback workflows. The
previous active business docs named alert states without defining ownership,
allowed transitions, terminality, or acting permissions, which made the behavior
implementation-ambiguous.

## Impact of Deferral

- No launch workflow creates operational risk alert records.
- No launch actor has operational risk alert permissions.
- Delivery assignment does not display risk-alert context.
- Notifications do not include operational risk alert events.
- Repeated exceptions remain reviewable through their owning records, reports,
  audit logs, and ordinary operational escalation.

## Revisit Trigger

Revisit when launch operations need systematic rolling-window detection for
repeated custody, cash, delivery, inventory, or adjustment exceptions beyond
direct owning-domain review and ordinary audit logs.

## Notes

- Support fallback behavior remains in [../business/support.md](../business/support.md).
- Notification channel behavior remains in
  [../business/notifications.md](../business/notifications.md).
- If support case management enters scope, risk alerts can be reconsidered as a
  linked source of case context.
