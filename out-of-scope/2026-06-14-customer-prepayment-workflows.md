# Customer Prepayment Workflows

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Customer prepayment workflows are not part of the mid-stage MVP. MVP online
orders use COD cash collected by the Delivery Agent at delivery.

This item covers generic pre-delivery payment gates, prepaid settlement,
prepaid cancellation and failed-delivery refund rules, customer wallets, payment
references, and provider-mediated refund behavior. Mobile-money-specific details
remain in [2026-06-13-mobile-money-payments.md](2026-06-13-mobile-money-payments.md).

## Proposed Scope

If moved back into scope, customer prepayment workflows would need explicit
rules for:

- when payment is required before fulfillment;
- customer payment instructions;
- provider reference submission and verification;
- merchant-account attribution;
- wallet, card, bank-transfer, or provider-specific payment facts;
- cancellation after customer payment;
- failed-delivery refund liabilities after customer payment;
- provider refunds versus cash refunds;
- customer credit or wallet balances;
- cross-outlet payment settlement;
- receipt, reporting, authorization, and audit behavior.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes COD
cash only.

- Order fulfillment had no pre-payment gate.
- External payment references, provider verification, wallets, cards, mobile
  money, and prepaid settlement gates were explicitly excluded from fulfillment.
- Payment did not own external payment reference submission, provider
  verification, merchant-account configuration, provider refunds, customer
  wallets, or stored external payment credentials.
- Supported payment rules excluded external payment instructions,
  merchant-account selection, transaction-reference fields, provider-statement
  checks, and late-reference reuse.
- Authorization edge-case `E-01` denied attempts to submit, verify, reuse, or
  administer external payment references.
- Finance had a prepayment settlement invariant and separate deferred
  prepayment reporting section.
- Finance stated that no ordinary workflow created cross-outlet customer-payment
  settlement because COD cash belongs to the fulfilling outlet.
- Refund rules explicitly excluded prepaid cancellation refunds, prepaid failed
  delivery refunds, electronic refunds, wallet credit, and provider refund
  workflows.
- Delivery failure rules stated that failed deliveries did not create prepaid
  refund liabilities.

## Why It Is Out of Scope

COD cash is sufficient for the MVP customer journey. Prepayment adds a second
payment lifecycle and cross-module settlement/refund rules before the business
has measured the need.

## Impact of Deferral

- Customers pay cash to the Delivery Agent at delivery.
- Fulfillment can proceed without customer payment action before delivery.
- Refunds remain cash-only outlet payouts.
- Finance does not need prepaid settlement or merchant-account attribution.

## Revisit Trigger

Revisit when COD-only launch operations create a measured adoption, cash-risk,
reconciliation, or customer-convenience problem that requires payment before
delivery.

## Notes

- MVP payment behavior is defined in [../business/payment.md](../business/payment.md).
- Mobile-money-specific deferred rules are captured in
  [2026-06-13-mobile-money-payments.md](2026-06-13-mobile-money-payments.md).
