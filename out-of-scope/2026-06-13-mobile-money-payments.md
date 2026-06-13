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

## Captured Prior Business Rules

These details were removed from `business/` so launch business documents can
describe the cash-only MVP without referencing this deferred payment rail.

### Order and Allocation

- Mobile-money orders were treated as prepaid orders.
- Customer-facing payment instructions were shown only after outlet allocation.
- The assigned outlet's configured merchant account was the default payment
  account, with a global company account allowed only as an explicit fallback
  configuration.
- Operational progress was blocked before outlet acceptance until provider
  reference submission, authorized staff verification, and any required payment
  gate resolution completed.
- Outlet acceptance, picking, batching, dispatch, and delivery-agent assignment
  were blocked until the payment gate permitted fulfillment.
- Payment-method support was part of outlet allocation eligibility.
- COD and mobile money were enabled by default unless outlet policy overrode
  them.
- An outlet with no active payment method was ineligible for all orders.

### Payment Lifecycle

The prior payment states were:

```text
PENDING
  +-> AWAITING_CUSTOMER_PAYMENT
  |     +-> AWAITING_STAFF_VERIFICATION
  |           +-> STAFF_VERIFIED
  |           |     +-> PAID
  |           |     +-> PARTIALLY_PAID
  |           |     |     +-> PAID
  |           |     +-> OVERPAID
  |           |           +-> PAID
  |           +-> FAILED
  +-> CANCELLED
```

- `STAFF_VERIFIED` meant an authorized outlet actor manually confirmed the
  provider reference.
- `PAID` meant the payment was posted and confirmed.
- `PARTIALLY_PAID` meant the verified amount was below the order total.
- `OVERPAID` meant the verified amount was above the order total.
- A bad or permanently rejected reference moved the payment to `FAILED`.
- Verification records captured verifier identity, verifier role, timestamp,
  provider, transaction reference, verified amount, and decision.
- Verification relied on provider, transaction reference, verified amount, and
  decision.
- Payer phone was not a required verification input at launch; the verifier
  manually compared provider-statement phone to customer/order phone.
- A phone mismatch was not a clean verification fact and required Outlet Manager
  operational resolution outside the payment-verification workflow.

### Reference Uniqueness and Reuse

- A transaction reference could not be applied to more than one active or
  completed payment per provider.
- The same reference string from different providers was not a duplicate.
- A second attempt to use the same reference at the same provider was rejected
  regardless of customer, outlet, or order.
- Late references after cancellation were flagged as
  `LATE_REFERENCE_AFTER_CANCELLATION`.
- A cancelled order was not reopened by a late reference.
- The same customer could reuse the same late reference on a new order within
  7 days, subject to normal checkout validation.
- Reused references required re-verification against the new order.
- After 7 days, mediation required an explicitly permissioned Outlet Cashier,
  Outlet Manager, or Super Admin.
- Cross-customer reuse required an audited override by an actor with explicit
  cross-customer override authority.

### Expiry

Unpaid mobile-money orders had a two-stage expiry window:

```text
t = 0    -> clock starts; order at AWAITING_CUSTOMER_PAYMENT; reservation active
t = 30m  -> warning sent; reservation still active
t = 60m  -> if no reference submitted -> CANCELLED; reservation released
           if reference submitted     -> clock stops permanently
```

- Submitting a reference before cancellation stopped the expiry clock.
- Submission moved the order to `AWAITING_STAFF_VERIFICATION`.
- Payment-verification delay did not trigger stock release under unpaid-expiry
  policy.

### Amount Handling

- If verified amount equaled order total, payment applied normally.
- If verified amount was less than order total, fulfillment remained blocked
  until an Outlet Manager approved a COD top-up for the full shortfall or a
  Super Admin approved an explicit exception adjustment.
- A verified underpayment could not silently proceed, be waived, or be written
  off.
- If verified amount exceeded order total, the excess became a cash refund
  liability before fulfillment proceeded.
- Excess value did not become customer credit, wallet balance, or a
  provider-issued refund.
- Late-reference reuse on a new order followed the same amount comparison:
  equal amount applied normally, greater new total required approved COD top-up,
  and lower new total created a cash refund liability for the overage.

### Cancellation, Failure, and Refunds

- Paid pre-dispatch cancellation moved the order to
  `CANCELLED_PENDING_REFUND`.
- Failed prepaid delivery moved to `CANCELLED_PENDING_REFUND`.
- Failed prepaid delivery remained financially open until the full prepaid
  amount was refunded in cash.
- `REFUNDED` was the true terminal state after cash payout was recorded and
  posted.
- The outlet that received the original payment was responsible for disbursing
  the refund.
- Cross-outlet refund payout was not supported.
- Provider-issued refunds, customer wallet credit, and mobile-money refund
  workflows were not supported.

### Delivery and Doorstep Adjustments

- Failed prepaid deliveries created a cash refund liability for the full paid
  amount until audited cash refund completion.
- If a failed delivery waived the delivery fee, the waived fee became part of
  the refund liability.
- If doorstep price recalculation lowered the final amount for a prepaid order,
  a cash refund liability was created at financial closure.
- If reassignment increased delivery fee or other final amount due after
  prepayment, the Delivery Agent collected the accepted increase as a COD
  delta/top-up at delivery.

### Finance and Settlement

- When one outlet received prepayment and another outlet fulfilled the order,
  the original payment remained attributed to the paid-to outlet and payment
  account.
- The final fulfilling outlet recognized the sale at financial closure.
- An internal settlement entry represented prepaid value received by the
  paid-to outlet and owed to the fulfilling outlet.
- The settlement was an accounting entry only, not a cash transfer between
  outlets.
- COD deltas, underpayment top-ups, conversion deltas, doorstep
  price-recalculation deltas, and delivery-fee increases collected by the
  fulfilling outlet posted directly to that outlet.
- If a reassigned paid order failed or was cancelled before successful
  completion, no internal outlet settlement was created.
- Receipts for reassigned prepaid orders could show fulfillment outlet,
  payment-receiving outlet, and refund-collection outlet when applicable, but
  did not expose internal settlement status or mechanics.

### Authorization

- Mobile-money verification was a scoped explicit permission for Outlet
  Cashiers and Outlet Managers, and full for Super Admin.
- Payment account administration was Super Admin-only.
- Payment reference submission was available to the owning customer and Super
  Admin.
- A payment-verification actor could not verify a payment for an order assigned
  to a different outlet, even if both outlets were in the same city.
- Creating, changing, deactivating, or setting the default merchant account
  required explicit payment-account administration permission, audit reason, and
  masked ordinary reads.

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
