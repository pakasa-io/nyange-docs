# Refund Code Expiry and Regeneration

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Refund collection-code expiry, regeneration, lockout, cooldown, and Super Admin
reveal workflows are not part of the mid-stage MVP. MVP refund payout uses a
single-use customer-presented collection code from the authenticated customer
experience.

## Proposed Scope

If moved back into scope, refund code lifecycle management would need explicit
rules for:

- collection-code validity windows;
- transition to an expired-code state;
- regeneration from expired-code state;
- regeneration when the customer loses access;
- Super Admin customer verification requirements;
- audited reveal behavior;
- failed-attempt lockout, rate limiting, or cooldown;
- permissions for code lifecycle management;
- notification and sensitive-content rules after regeneration or reveal.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes a
simple one-time collection-code check.

- A refund collection code was single-use and perishable.
- A collection code had a finite validity window.
- The launch validity window was 24 hours after issuance.
- Unused codes moved `COLLECTIBLE -> CODE_EXPIRED`.
- `CODE_EXPIRED` could return to `COLLECTIBLE` when a new code was issued.
- Code expiry did not discharge the refund liability.
- A Super Admin could regenerate an expired code with audit.
- Regeneration when the customer lost access required customer verification
  through an audited exception record with reason and audit.
- Failed verification attempts did not lock the collection code or refund
  record.
- Collection-code lockout, failed-attempt rate limiting, and cooldown workflows
  were explicitly excluded from the prior launch behavior.
- A Super Admin could perform audited code reveal after customer verification.
- The access matrix included `Refund collection code management` as a Super
  Admin permission.

## Why It Is Out of Scope

Expiry, regeneration, lockout, cooldown, and reveal workflows add support and
authorization complexity before refund volume proves the need.

## Impact of Deferral

- Refund collection codes do not expire through a separate lifecycle state.
- Super Admins do not manage or reveal refund collection codes.
- Outlet payout actors verify customer-presented codes during payout.
- Code support exceptions are handled through ordinary customer authentication
  and outlet verification.

## Revisit Trigger

Revisit when expired, lost, or inaccessible codes create repeated payout support
cases that cannot be handled by ordinary customer authentication and outlet
verification.

## Notes

- MVP refund behavior is defined in [../business/refund.md](../business/refund.md).
- MVP sensitive notification behavior is defined in
  [../business/notifications.md](../business/notifications.md).
