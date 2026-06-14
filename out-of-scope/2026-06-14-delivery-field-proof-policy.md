# Delivery Field Proof Policy

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Configurable delivery field proof policy is not part of the mid-stage MVP. MVP
delivery requires live field recording and authoritative timestamps for delivery
actions, without GPS, photo, note, fallback-code, or evidence-retention policy
rules.

## Proposed Scope

If moved back into scope, delivery field proof policy would need explicit rules
for:

- GPS capture requirements by action or exception;
- photo capture requirements by action or exception;
- required versus optional delivery notes;
- doorstep fallback codes;
- evidence retention duration and access;
- evidence visibility by role and order state;
- behavior when required evidence cannot be captured;
- privacy, safety, and audit constraints;
- interaction with delivery completion, failed delivery, and cash/cylinder
  custody exceptions.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes live
field recording only.

- GPS, photo, or note requirements were controlled by active field-proof policy.
- Field-proof policy could not allow partial delivery completion.
- Doorstep code fallback and evidence-retention detail were listed outside the
  order MVP lifecycle.

## Why It Is Out of Scope

Live field recording and timestamps are enough for the MVP delivery flow. GPS,
photo, notes, fallback codes, and evidence retention add product, privacy, and
operations policy before dispute volume proves the need.

## Impact of Deferral

- Delivery agents still record required field actions live.
- The system records authoritative timestamps.
- GPS, photo, required-note, fallback-code, and evidence-retention behavior are
  not required for MVP delivery.

## Revisit Trigger

Revisit when delivery disputes, safety incidents, or cash/cylinder custody
exceptions require evidence beyond live field recording and timestamps.

## Notes

- MVP delivery behavior is defined in [../business/delivery.md](../business/delivery.md).
- MVP order behavior is defined in [../business/order.md](../business/order.md).
