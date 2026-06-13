# Mobile-Money Payments

## Status

- Status: Deferred
- Date Added: 2026-06-13
- Source: User scope decision on 2026-06-13; prior business docs that treated
  mobile money as a launch payment method.
- Owner: Product Manager

## Summary

Mobile-money customer payments are not part of the mid-stage MVP. Launch
payment behavior is cash-only: COD for online delivery orders and immediate
cash for walk-in POS sales.

This deferral covers the complete mobile-money payment rail, not only provider
integration work.

## Proposed Scope

If moved back into scope, mobile-money payments would need explicit business
rules for:

- provider and merchant-account configuration per outlet;
- customer-facing payment instructions after outlet allocation;
- customer transaction-reference submission;
- provider-reference uniqueness and reuse limits;
- authorized staff verification of provider references;
- amount matching, underpayment, overpayment, and discrepancy resolution;
- unpaid-order expiry and late-reference handling;
- mobile-money-paid cancellation and failed-delivery refund liabilities;
- post-payment outlet reassignment and any internal settlement rules;
- audit, authorization, support, reporting, and receipt behavior.

## Why It Is Out of Scope

Mobile-money payments add a second launch payment lifecycle before fulfillment
can proceed. That lifecycle requires provider-reference verification, payment
account administration, discrepancy handling, late-reference policy,
prepaid-cancellation refunds, and cross-module settlement decisions.

Deferring it keeps the MVP focused on one coherent payment model: cash at
delivery or cash at walk-in sale completion.

## Impact of Deferral

- Online customer orders cannot be prepaid by mobile money at launch.
- Customers pay cash to the Delivery Agent at the doorstep.
- Walk-in POS customers pay cash at the outlet.
- No launch actor can submit, verify, reuse, or administer mobile-money payment
  references.
- No mobile-money merchant-account setup is required for launch operations.
- Launch refunds remain cash-only outlet payouts.
- Post-payment outlet reassignment settlement for mobile-money-paid orders is
  not a launch workflow.

## Revisit Trigger

Reconsider mobile-money payments when COD-only launch operations create a
measured adoption, cash-risk, reconciliation, or customer-convenience problem
that cannot be solved by cash-process controls within the MVP.

Before re-entry, produce updated order, payment, refund, delivery, finance,
authorization, reporting, and support rules together. Do not reintroduce
mobile-money payment references in one module without the full cross-module
lifecycle.

## Notes

- Launch payment behavior is defined in [../business/payment.md](../business/payment.md).
- Refund behavior remains cash-only in [../business/refund.md](../business/refund.md).
- This item does not defer customer SMS OTP, notification, or phone-number
  authentication behavior.
