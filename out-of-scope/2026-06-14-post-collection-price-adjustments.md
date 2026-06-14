# Post-Collection Price Adjustments

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business review finding on unsupported refund-liability sources
- Owner: Product Manager

## Summary

Post-collection price adjustment is not a launch refund-liability source.
Launch refund liabilities may be created only from an authorized and posted
Finance-owned cash over-collection correction.

## Proposed Scope

If moved into scope, post-collection price adjustment would need explicit
business rules for:

- source owner;
- eligible source events;
- adjustment authorization and separation of duty;
- adjustment posting and audit;
- impact on immutable order totals and receipts;
- Refund handoff contract;
- customer notification and collection-code behavior;
- daily closing and reporting effects.

## Why It Is Out of Scope

The current MVP preserves immutable order totals, immutable receipts, and
cash-only COD payment facts. No launch module owns a post-collection price
adjustment workflow or the policy that decides when a delivered sale should
create a customer refund liability.

## Impact of Deferral

- Post-collection price adjustment cannot create a launch refund liability.
- Delivery-time price-delta or refund negotiation remains out of scope.
- Refund liability creation is limited to Finance-owned cash over-collection
  correction.

## Revisit Trigger

Revisit when the Product Manager approves a post-delivery adjustment policy with
source owner, authorization, posting, receipt, reporting, and Refund handoff
rules.

## Notes

- Launch refund behavior remains in [../business/refund.md](../business/refund.md).
- Finance-owned cash over-collection correction remains in
  [../business/finance.md](../business/finance.md).
