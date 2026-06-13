# Order

**Intent**: Define launch order behavior for placement, lifecycle state,
allocation, reassignment, cancellation, cart behavior, mobile-money expiry,
post-delivery returns, and walk-in POS.

**Reader task**: Use this document to decide whether an order can be placed,
assigned, accepted, changed, cancelled, reassigned, delivered, refunded, or
financially closed.

**Sources**: §6.1 Order Lifecycle, §7.1 Outlet Allocation, §7.6 Cascade &
Reassignment, §7.13 Mobile Money Order Expiry, §7.15 Cart Behaviour, F-01, F-02

**Related**:
[catalog.md](catalog.md) for pricing and catalog rules;
[payment.md](payment.md) for payment lifecycle;
[delivery.md](delivery.md) for delivery lifecycle;
[inventory.md](inventory.md) for reservation lifecycle;
[refund.md](refund.md) for refund liabilities;
[finance.md](finance.md) for post-payment reassignment settlement and forced
closure; [identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-02 — Price snapshot is write-once.**

- When an order is placed, prices in effect at that moment are captured and
  frozen on the order.
- Subsequent pricing-rule changes do not alter historical orders.
- Downstream financial calculations, including receipts, refunds, and
  settlements, derive from the frozen snapshot, not current rules.

**BI-07 — COD orders never require payment verification before fulfillment
begins.**

- For COD orders, outlet acceptance, picking, batching, dispatch, and
  delivery-agent assignment may begin without any payment action.
- Payment is collected at the doorstep by the delivery agent.

**BI-09 — Fulfilled orders have no partial completion state.**

- Either all items in an order are successfully delivered, or the whole order
  fails.
- The only exception is an approved full-order adjustment that resolves the
  delivery issue, such as an Outlet Manager-approved refill-to-new-cylinder
  conversion at the doorstep.
- The exception is still a whole-order mutation, not selective line-level
  completion.

**BI-17 — Outlet policy is a prerequisite for order acceptance, not a post-hoc
check.**

- Before an order is assigned to an outlet, the outlet's policies are evaluated.
- Relevant policies include payment method support, delivery mode support,
  refill vendor acceptance, and same-vendor refill capability.
- An order cannot be assigned to an outlet that does not meet its requirements.

**BI-20 — Order totals are immutable after placement.**

- No business event can alter the total of a placed order except through an
  explicit, audited adjustment workflow that generates its own financial record.
- Covered events include price changes, outlet reassignment, bundle-discount
  updates, and pricing-rule updates.
- Delivery-fee changes and other post-placement deltas are explicit adjustments.
- A customer refund created by a lower final amount remains a separate refund
  liability.
- Refund liabilities do not rewrite the placed order's price snapshot or
  original charge.

**BI-21 — Every cancellation must be fully attributed.**

Every cancellation record must capture:

- actor identity;
- actor role;
- reason code;
- reason note;
- timestamp.

A cancellation missing any required field is a data integrity violation.

**BI-22 — Outlet disable is blocked by active orders.**

- An outlet cannot be set inactive or disabled while it has any non-terminal
  orders.
- Terminal states for this rule are `COMPLETED`, `CANCELLED`, `FAILED` for COD
  orders with no open refund liability, and `REFUNDED`.
- `CANCELLED_PENDING_REFUND` is non-terminal because the financial liability is
  still open.
- Disablement is allowed only after every order is completed, cancelled, failed
  as COD with no open refund liability, refunded, or reassigned.

**BI-23 — Critical mutations are idempotent.**

- Critical one-time state transitions require idempotency keyed by actor,
  endpoint, and client key.
- Covered transitions include order creation, cancellation, payment reference
  submission or reuse, payment verification, refunds, delivery completion,
  inventory reservation/adjustment/transfer, cash reconciliation, receipt
  generation, and comparable mutations.
- Identical replays return the original result.
- Same-key/different-body replays are rejected.
- In-progress duplicates return a retryable response.
- Domain uniqueness rules remain permanent even after operational replay records
  expire.

## Boundary

- Order owns the order request, placement, assignment, cancellation,
  reassignment, customer-visible order state, and order-level immutable price
  snapshot.
- Order does not own payment verification, delivery execution, stock ledger
  movement, refund payout, or financial closure ledger posting.
- Other aggregates may observe or act on order state but must not rewrite the
  placed order request or price snapshot.

## Order Lifecycle

```
DRAFT
  └─► PLACED
        ├─► [Mobile Money]
        │     AWAITING_CUSTOMER_PAYMENT
        │       └─► AWAITING_STAFF_VERIFICATION
        │             └─► STAFF_VERIFIED
        │                   └─► ASSIGNED_TO_OUTLET
        │
        └─► [COD]
              ASSIGNED_TO_OUTLET
                    │
        ┌───────────┘
        ▼
  AWAITING_OUTLET_ACCEPTANCE
        ├─► ACCEPTED_BY_OUTLET
        │     └─► PICKING
        │           └─► READY_FOR_DISPATCH
        │                 └─► OUT_FOR_DELIVERY
        │                       ├─► DELIVERED ──► COMPLETED
        │                       └─► FAILED
        │                             └─► [if mobile-money prepaid] CANCELLED_PENDING_REFUND
        │                                                                  └─► REFUNDED
        │
        └─► [Cascade / Reassignment]
              ├─► AWAITING_CUSTOMER_REASSIGNMENT_ACCEPTANCE
              │     ├─► ACCEPTED_BY_OUTLET
              │     └─► CANCELLED or CANCELLED_PENDING_REFUND or REQUIRES_ADMIN_INTERVENTION
              │
              └─► REQUIRES_ADMIN_INTERVENTION
                    ├─► ACCEPTED_BY_OUTLET
                    └─► CANCELLED or CANCELLED_PENDING_REFUND

CANCELLED
CANCELLED_PENDING_REFUND
  └─► REFUNDED
FAILED
COMPLETED
```

| State | Meaning |
| --- | --- |
| `DRAFT` | Customer-derived order draft before placement. |
| `PLACED` | Order request created with frozen price snapshot. |
| `AWAITING_CUSTOMER_PAYMENT` | Mobile-money order awaits customer reference submission. |
| `AWAITING_STAFF_VERIFICATION` | Reference submitted; outlet verification pending. |
| `STAFF_VERIFIED` | Authorized outlet actor manually confirmed provider reference. |
| `ASSIGNED_TO_OUTLET` | Order has serving outlet assignment. |
| `AWAITING_OUTLET_ACCEPTANCE` | Shared fulfillment entry state for mobile-money and COD paths. |
| `ACCEPTED_BY_OUTLET` | Outlet accepted order under active acceptance policy. |
| `PICKING` | Authorized actor is picking stock. |
| `READY_FOR_DISPATCH` | Picked and ready for delivery assignment/dispatch. |
| `OUT_FOR_DELIVERY` | Delivery-run custody has begun. |
| `DELIVERED` | Customer-facing delivery complete; financial closure may still be pending. |
| `COMPLETED` | Internal financial closure complete; terminal. |
| `FAILED` | Terminal for COD orders; prepaid mobile-money moves to refund path. |
| `CANCELLED` | Terminal order cancelled before payment, or COD cancelled. |
| `CANCELLED_PENDING_REFUND` | Operationally cancelled but refund liability remains open. |
| `REFUNDED` | Terminal after cash refund payout is recorded and posted. |
| `REQUIRES_ADMIN_INTERVENTION` | All candidate outlets exhausted; Super Admin intervention required. |

## State Rules

### Cancellation and Changes

- Customer cancellation is allowed up to and including `PICKING`.
- `READY_FOR_DISPATCH` cancellation is a pre-custody exception for an explicitly
  permissioned in-scope Customer Support Agent or Super Admin.
- `READY_FOR_DISPATCH` cancellation requires override acknowledgement, reason,
  note, audit, and pick-reversal handling when stock has already been picked.
- Once delivery-run custody or `OUT_FOR_DELIVERY` begins, normal cancellation is
  closed.
- After custody begins, failed-delivery, return/refund, forced-closure, or
  audited financial-adjustment workflows carry the outcome.
- Outlet staff without explicit cancellation permission cannot use normal
  cancellation after `PICKING`.
- Those actors use cannot-fulfill, pick-reversal, failed-delivery, Customer
  Support Agent escalation, or Super Admin handling according to fulfillment
  state.
- After placement, customer-requested changes to items, delivery address, or
  original pricing are not supported as order modifications.
- If the customer wants a different order before fulfillment, the existing order
  is cancelled where allowed and the customer places a new order.
- Approved delivery-conversion, mismatch, refund, or reassignment-delta paths
  are exception adjustments, not edits to the original order request.

### Assignment and Payment Gates

- All orders receive outlet assignment and reserve inventory at placement.
- For mobile-money orders, status does not transition to `ASSIGNED_TO_OUTLET`
  until payment reaches `STAFF_VERIFIED` and the payment gate permits
  fulfillment.
- Before mobile-money `ASSIGNED_TO_OUTLET`, the assigned outlet may see the order
  for payment instructions and payment verification only.
- Before mobile-money `ASSIGNED_TO_OUTLET`, the outlet may not accept, pick,
  batch, dispatch, or assign a delivery agent.
- COD orders transition to `ASSIGNED_TO_OUTLET` immediately after placement.
- `STAFF_VERIFIED` and `PAID` are distinct. `STAFF_VERIFIED` means provider
  reference was manually confirmed; `PAID` means payment is posted and
  confirmed.
- An underpayment holds payment at `PARTIALLY_PAID` while COD top-up is arranged.
- Operational progress for mobile-money orders is blocked until
  `STAFF_VERIFIED` and any required payment-gate resolution.
- Required payment-gate resolution includes Outlet Manager approval of COD
  top-up for underpayment.

### Outlet Acceptance

- `AWAITING_OUTLET_ACCEPTANCE` is the shared entry state for all order paths.
- Mobile-money orders enter it after `STAFF_VERIFIED`, `ASSIGNED_TO_OUTLET`, and
  required payment-gate resolution.
- COD orders enter it directly from `ASSIGNED_TO_OUTLET`.
- Outlet acceptance follows the active outlet acceptance policy.
- Launch default requires an actor with explicit order-acceptance permission to
  manually accept or reject the order.
- The system may alternatively be configured to automatically accept eligible
  assigned orders without manual outlet action.
- If the outlet rejects the order or the acceptance window expires, the order
  enters cascade/reassignment before picking begins.
- For orders accepted under no-manual-action policy, failure to start picking
  within the active picking-start timeout enters the same pre-picking
  cascade/reassignment path.
- If an accepted outlet marks an order cannot-fulfill before `PICKING`, the
  order re-enters cascade/reassignment.
- After `PICKING`, cannot-fulfill is a post-picking exception and must use pick
  reversal, cancellation/refund, failed-delivery, Customer Support Agent
  routing, or Super Admin handling according to custody state.

### Delivery and Financial Closure

- `DELIVERED` is customer-facing completion.
- `COMPLETED` is internal financial closure and is not customer-facing.
- Delivery confirmation is issued to the customer immediately at `DELIVERED`.
- The financial receipt with full cost breakdown is generated and issued only at
  `COMPLETED`.
- If financial closure is delayed by unresolved custody or inventory exceptions,
  the customer receives delivery confirmation first and the receipt later.
- Online delivery `COMPLETED` requires delivery-run custody closure, successful
  financial posting, and no unresolved closure blockers.
- Refill orders additionally require returned-cylinder intake confirmation for
  every expected cylinder or approved exception.
- Non-refill orders may move from `DELIVERED` to `COMPLETED` as soon as closure
  facts are present.
- Normal financial closure occurs from recorded business facts.
- Customers and outlet staff do not manually mark online orders `COMPLETED`.
- Super Admin forced closure is a separate audited exception workflow before
  closure effects are posted.

### Pending Closure View

- An order in `DELIVERED` but not `COMPLETED` appears in a derived internal
  pending-closure view when unresolved closure blockers remain.
- The pending-closure view is not a separate order lifecycle state.
- The pending-closure view is not customer-facing.
- Default target resolution is within the same business day.
- A hard operational alert is raised if the order remains unclosed after 24
  hours.
- Super Admin escalation is triggered at 48 hours.
- If blockers remain unresolved after escalation, Super Admin may use forced
  financial closure only through [finance.md](finance.md).
- Each closure blocker identifies blocker type, owning outlet or workflow,
  related business record, SLA deadline, current status, and resolution or
  waiver audit facts.

### Delivery PIN

- The delivery PIN is a six-digit one-time code.
- It is generated when the order reaches `READY_FOR_DISPATCH`.
- It is not generated at placement.
- The customer is notified at `READY_FOR_DISPATCH`.
- The short exposure window reduces PIN leakage risk before the delivery window
  opens.

### Failed and Refunded Outcomes

- For COD orders, `FAILED` is terminal.
- For mobile-money prepaid orders, `CANCELLED_PENDING_REFUND` is required after
  `FAILED`.
- Failed prepaid delivery remains financially open until the full prepaid amount
  is refunded in cash.
- The delivery fee is waived for failed delivery.
- `REFUNDED` is the true terminal for failed prepaid deliveries.
- When a paid mobile-money order is cancelled before dispatch, it moves to
  `CANCELLED_PENDING_REFUND`.
- A refund liability is created immediately for paid mobile-money pre-dispatch
  cancellation.
- The order reaches `REFUNDED` only after cash payout is recorded and posted.
- Unpaid mobile-money orders cancelled before dispatch and COD orders cancelled
  at any point move directly to `CANCELLED`.

## Outlet Allocation

### Geocoding Precondition

- Checkout is blocked if the selected delivery address has no resolved
  coordinates.
- Outlet allocation requires valid latitude/longitude to calculate distances and
  determine eligible outlets.
- If geocoding fails on address save, the address is saved as `UNRESOLVED`.
- Checkout remains blocked until coordinates are resolved or confirmed.
- Customers may correct the map pin.
- Non-customer coordinate correction requires explicit coordinate-correction
  permission and audit logging.

### Eligibility Gate

An order is allocated to exactly one outlet.

```
outlet eligible :=
  address_in_active_service_zone
  AND outlet_active
  AND (express_order  → outlet_currently_operating
       batched_order  → has_valid_future_delivery_window)
  AND supports_delivery_mode
  AND supports_payment_method
  AND sufficient_available_stock
  AND (refill_order → accepts_incoming_vendor)
  AND (same_vendor_requested → same_vendor_policy_enabled AND filled_stock_available)
  AND (capacity_limit == none OR active_online_orders < capacity_limit)
```

### Eligibility Notes

- Geographic eligibility uses confirmed coordinates inside an active radius-ring
  service zone.
- Minimum zone boundary is inclusive.
- Maximum zone boundary is exclusive.
- Polygon zones are out of launch scope.
- The outlet must be active.
- Express orders require the outlet to be currently operating at placement.
- If no eligible outlet operates for express, express is unavailable and the
  customer must choose batched.
- Batched orders require a valid future delivery window.
- Batched orders do not require the outlet to be currently operating at
  placement.
- Inventory is reserved at placement regardless.
- COD and mobile money are enabled by default unless outlet policy overrides.
- An outlet with no active payment method is ineligible for all orders.
- Stock availability is required for all order items.
- For refill orders, stock checks include outgoing filled cylinders and
  accessories only.
- The customer's empty cylinder is an incoming return and does not consume stock
  at placement.
- Expected vendor-depot returns and `IN_REFILL` cylinders do not satisfy
  same-vendor stock requirements.
- Capacity gate is optional.
- If capacity gate is configured and reached, the outlet is excluded.
- Walk-in orders do not count toward active online order capacity.

### Service Zone Templates

| Template | Radius | Eligibility |
| --- | --- | --- |
| CORE | 0 km <= distance < 5 km | Eligible at minimum boundary; not at maximum |
| STANDARD | 5 km <= distance < 10 km | Eligible at minimum boundary; not at maximum |
| EXTENDED | 10 km <= distance < 15 km | Eligible at minimum boundary; not at maximum |

- Each active online-fulfillment outlet must have confirmed outlet
  latitude/longitude before it can be eligible.
- Actual outlet coordinates are outlet master data, not hard-coded in this
  domain specification.

### Priority Ranking

Among eligible outlets:

```
rank outlets by:
  1. distance_to_customer      ASC
  2. delivery_fee              ASC
  3. active_online_order_load  ASC
  4. outlet_priority_score     DESC
```

- `active_online_order_load` counts assignment through delivery-run closure.
- Walk-ins are excluded from active online order load.
- `outlet_priority_score` defaults to 0 when unset.
- Delivery-agent capacity is not an outlet-allocation factor at launch.
- An otherwise eligible outlet is not excluded because no delivery agent is
  immediately available.
- Delivery assignment happens after outlet allocation under delivery lifecycle.
- Customers do not choose their outlet.
- The serving outlet is determined by active allocation policy.

### Split-Order Rules

- Orders are not split across outlets.
- If no single eligible outlet can provide every requested stock item,
  allocation may fall back only to the closest outlet that satisfies every other
  allocation rule and has all core cylinder items.
- This fallback relaxes only accessory stock availability.
- The customer may place the order without unavailable accessories or not place
  the order.
- If an order includes one or more core cylinder items and no single outlet has
  all core cylinder items, the order cannot be placed.
- Accessory-only orders still require one eligible outlet with requested
  accessory stock.
- Refill vendor-policy exhaustion is handled separately from core-stock absence.
- If candidate outlets have required outgoing stock but are excluded by incoming
  vendor acceptance policy, the order follows exhausted-candidate Super Admin
  intervention instead of stock-unavailable checkout failure.

## Cascade and Reassignment

### Unchanged-Terms Cascade

No customer notification is required when:

- the first-choice outlet rejects the order;
- the first-choice outlet marks cannot-fulfill before picking;
- the first-choice outlet acceptance window expires;
- the next eligible outlet can fulfill with unchanged delivery terms.

Unchanged delivery terms mean same fee, same window, and same vendor.

During unchanged-terms cascade, order status remains
`AWAITING_OUTLET_ACCEPTANCE` while outlet assignment is updated internally to
the next candidate.

### Customer Acceptance Required

Customer notification and acceptance are required when reassignment creates:

- higher delivery fee or amount due;
- later or different delivery window;
- different outgoing cylinder vendor;
- refill-to-new-cylinder conversion;
- cancellation/refund-only outcome;
- any other change affecting what the customer pays or receives.

### Changed-Terms Sequence

1. Stock is held at the candidate outlet as `REASSIGNMENT_HOLD`.
2. The candidate outlet provisionally accepts the changed order within a short
   window.
3. Launch provisional acceptance windows are 5 minutes for express and 15
   minutes for batched.
4. Provisional acceptance is not full operational acceptance.
5. Only after provisional outlet acceptance is the customer notified.
6. The customer has a bounded acceptance window.
7. Launch customer acceptance windows are 15 minutes for express and 30 minutes
   for batched.
8. If the customer accepts, the reassignment hold becomes the active reservation
   at the new outlet.
9. If the customer accepts, the order becomes fully assigned and accepted by the
   new outlet.
10. If the customer accepts, the previous outlet's reservation is released.
11. If the customer rejects or does not respond, the new outlet hold is released.
12. The normal outcome after rejection or timeout is cancellation and any
    pre-payment creates a refund liability, unless active reassignment policy
    requires escalation.

### Reassignment Rules

- Delivery-window mismatches between outlets are not silently translated or
  rebooked during cascade.
- If the original customer-visible window cannot be honored by the candidate
  outlet, the customer-acceptance path applies.
- If the candidate outlet does not provisionally accept within timeout, the hold
  is released and the next candidate is tried.
- The customer is not notified about a candidate outlet that has not confirmed
  it can fulfill the changed order.
- Stock hold by itself is not outlet acceptance.
- If no provisional outlet acceptance exists, the order remains in or returns to
  `AWAITING_OUTLET_ACCEPTANCE`.
- For post-payment reassignment, the customer is not notified merely because the
  internal fulfilling outlet changed.
- Customer notification is required only when customer-visible details change.

### Candidate Skip Records

- Every outlet evaluated but filtered out before stock hold or outlet
  confirmation request records a candidate skip reason.
- Candidate skip records support operational and support explanation.
- Candidate skip records are diagnostic, not full reassignment attempts.
- Launch retention is 180 days after record creation.
- After retention, skip records may be archived or summarized.
- Terminal order, financial, settlement, refund, and audit records remain
  durable under their own retention rules.

### Reassignment Attempt Records

- A reassignment attempt begins only when stock is held or outlet confirmation
  is requested.
- Attempt records track candidate hold, provisional outlet acceptance/rejection,
  outlet timeout, customer acceptance/rejection, customer timeout, and hold
  release.
- Once a reassignment attempt reaches a terminal outcome, that outcome is
  immutable except for appended audit or status history.

### Stock Reservation on Reassignment

- For paid or otherwise active reassignment, the candidate outlet hold must be
  secured before the previous outlet reservation is released.
- If the previous reservation cannot release after the new hold succeeds, the
  previous reservation remains unavailable as `PENDING_RELEASE`.
- `PENDING_RELEASE` stock is reported separately from normal reserved stock
  until release outcome is corrected and recorded.

### Delivery Fee Adjustments

- Any delivery-fee difference from reassignment is an explicit price adjustment
  recorded against the order and reassignment record.
- Delivery-fee adjustments comply with BI-20.
- A lower delivery fee does not require customer acceptance.
- For unpaid or COD orders, the lower fee reduces amount due.
- For prepaid mobile-money orders, the lower-fee difference becomes a refund
  liability created at financial closure.
- The customer is notified of a reduction but does not need to act.
- A higher delivery fee requires customer notification and acceptance before
  reassignment activates.
- After customer acceptance, COD orders collect the increase as part of COD due
  at delivery.
- For prepaid mobile-money orders, the accepted increase is collected by the
  delivery agent as a COD delta/top-up at delivery.
- The COD delta/top-up remains distinct from the original prepaid amount.

### Post-Picking Block

- Reassignment is blocked after picking has begun.
- Post-picking order reassignment is an Outlet Manager or Super Admin exception
  workflow.
- Picked stock does not return to available inventory until an authorized
  inventory/custody actor confirms pick reversal or return outcome.

### All Candidates Exhausted

- When all candidate outlets are exhausted, the order enters
  `REQUIRES_ADMIN_INTERVENTION`.
- The order is not immediately cancelled.
- Permissioned Super Admins are notified by push.
- Timeout-driven cancellation/refund applies only if escalation remains
  unresolved within launch defaults.
- Launch unresolved escalation timeout is 30 minutes from escalation for express
  orders.
- Launch unresolved escalation timeout is 2 hours from escalation for batched
  orders.
- During the intervention window, Super Admin may manually assign an outlet or
  cancel the order.
- Manual assignment is an audited exception override of normal eligibility gates.
- Normal eligibility gates include zone, hours, inventory, and outlet policy.
- Manual assignment requires explicit reason.
- Manual assignment does not make eligibility gates optional in ordinary
  allocation.
- For refill vendor-policy exhaustion, intervention may include Super Admin or
  permissioned Customer Support Agent contacting the customer through approved
  support or notification paths to offer cancellation-and-reorder alternatives,
  such as buying a new cylinder.
- Intervention does not permit editing the placed order.
- Exhausted-candidate intervention window may be extended only by Super Admin
  with reason and audit.
- A Customer Support Agent may request an extension through support action
  request, but cannot execute the extension.

## Mobile Money Order Expiry

Unpaid mobile-money orders have a two-stage configurable expiry window. The
clock starts when the order reaches `AWAITING_CUSTOMER_PAYMENT`.

Launch durations are global defaults. Changes or outlet/payment-method
overrides require Product Manager approval plus the operations approval
authority required by active policy.

```
t = 0    → clock starts; order at AWAITING_CUSTOMER_PAYMENT; reservation active
t = 30m  → warning sent; reservation still active
t = 60m  → if no reference submitted  → CANCELLED; reservation released
           if reference submitted      → clock stops permanently
```

- Submitting a reference before cancellation stops the expiry clock.
- Submission moves the order to `AWAITING_STAFF_VERIFICATION`.
- The expiry path is permanently closed once a reference is submitted.
- This is true regardless of verification outcome.
- Payment-verification delay does not trigger stock release under unpaid-expiry
  policy.
- COD orders do not have payment expiry.
- COD orders may time out through outlet acceptance workflow if the outlet does
  not accept within the acceptance window.
- No payment deadline applies to COD orders.

## Cart Behaviour

- Cart creation, cart changes, quotes, and order placement require an
  authenticated customer account at launch.
- Anonymous guest cart and checkout are not launch behavior.
- Out-of-stock cart items are marked unavailable and remain in the cart until
  the customer removes them or stock returns.
- The customer must resolve unavailable items before checkout can proceed.
- Cart and checkout quote prices are estimates until order placement.
- Final pricing is recalculated after outlet allocation.
- Pricing is locked only when the order is placed and the price snapshot is
  created.
- Catalog or pricing changes affecting cart lines require customer review and
  acknowledgement before cart quote or checkout can proceed.
- Catalog or pricing changes include price changes, product disablement, or
  bundle composition changes.
- If a product's price changes while a cart is open, cart prices update to the
  current price and the customer receives an explicit price-change notice before
  checkout.
- Disabled products are not orderable or sellable at any outlet, even if stock
  exists.
- Any cart item referencing a disabled product is marked unavailable.
- Already-placed orders referencing a disabled product are not affected.
- If a bundle's composition changes while a cart is open, any cart line
  containing that bundle is flagged invalid.
- The customer must review and acknowledge the bundle change before cart quote
  or checkout can proceed.
- Active carts are customer-derived commerce state, not durable order, payment,
  ledger, custody, receipt, or audit facts.
- Carts persist across customer sessions and devices.
- A cart with no customer fetch, mutation, or quote activity for 90 days is
  marked `ABANDONED`.
- After an additional 30 days, abandoned cart item detail is no longer retained
  as active cart detail.
- A safe cart summary remains.
- Checked-out carts and placed orders are never part of abandoned-cart cleanup.

## Post-Delivery Return Policy

Launch post-delivery returns require all conditions below.

1. Return window is 7 calendar days from confirmed delivery timestamp.
2. The customer must return the item or cylinder to the owning outlet in person.
3. No reverse-logistics pickup is provided.
4. The customer must be identifiable at return.
5. Anonymous post-delivery returns are not accepted.
6. An authorized outlet return-intake actor must inspect the returned item or
   cylinder.
7. The active return policy's approval role must approve condition before a
   refund liability is created or returned-stock effect is recognized.
8. A condition-rejected return does not create a refund or change inventory
   availability.
9. Post-delivery return refunds are cash-only at the outlet.
10. No mobile-money or alternative refund method applies.

Return eligibility, approved condition, refund amount, drop-off requirement, and
approval role are controlled by active product-type return policy. Later policy
changes must preserve explicit customer identity, approval, and audit
requirements.

- Post-delivery returns are processed against the original order whenever
  possible.
- Unlinked manual refunds are Super Admin-only exceptions.

## Walk-In POS Rules

- Walk-in orders require explicit POS/cashier permission.
- Walk-in orders share the same inventory as online orders.
- Walk-in stock reservation is immediate and final.
- Walk-in stock reservation has no pending state and no delivery delay.
- A walk-in sale reaches financial closure at sale completion.
- The walk-in receipt number and immutable receipt are issued at sale
  completion.
- At launch, the same pricing rules apply to walk-in and online orders.
- Different POS/online pricing requires a later explicit policy decision.
- Operating hours govern online order allocation only.
- Operating hours do not apply to walk-in POS.
- Walk-in refill exchanges use the same incoming/outgoing vendor rules, pricing
  matrix, same-vendor policy checks, and returned-cylinder inspection as online
  refills.
- The delivery and agent custody legs are skipped for walk-in refill exchanges.
- An explicitly permissioned POS/cashier actor handles the exchange directly at
  the counter.
- The same all-or-nothing semantics apply.
- If the returned cylinder is unacceptable, the POS/cashier actor may offer a
  full-order conversion under the same approved adjustment path.
- If the customer refuses conversion, the transaction does not proceed.
- Anonymous walk-in customers are permitted.
- Refunds, returns, and support follow-up require customer identity.
- Anonymous walk-in mobile-money payments still require provider, transaction
  reference, and verified amount capture.
- The same per-provider payment reference uniqueness rules apply.

## F-01: Online Order to Delivery

1. Customer places order from a selected delivery address with confirmed
   coordinates. Coordinates may be provider-resolved or a customer-confirmed
   manual pin.
2. Serving outlet is selected from outlets that serve the address and meet all
   allocation criteria.
3. Stock for all order items is reserved at the selected outlet.
4. For mobile money, customer pays independently, submits reference, authorized
   outlet payment-verification actor verifies, and required payment-gate
   resolution completes before outlet acceptance.
5. For COD, order proceeds directly to outlet acceptance.
6. Outlet accepts.
7. Authorized outlet picking actor picks items.
8. Delivery-agent assignment and delivery-run creation follow active outlet
   delivery policy.
9. Default assignment policy applies unless Dispatcher or Outlet Manager
   manually assigns or overrides before pickup within outlet scope.
10. Outlet handover confirmation and agent receipt confirmation are both
    recorded.
11. Custody transfers to agent.
12. At customer door for refill orders, agent records every expected returned
    cylinder's vendor, size, condition, approved conversion, COD delta or cash
    collection, and delivery exception before PIN confirmation.
13. Customer confirms with PIN only when final delivered order and amount due are
    correct.
14. PIN confirmation atomically commits delivery status, order status, outgoing
    stock commitment, returned-cylinder field recording, cash collection
    recording, and payment status.
15. For refill orders, returned-cylinder intake actor confirms intake for every
    expected returned cylinder or failed intake receives approved exception.
16. For all online deliveries, financial closure waits until delivery-run
    custody is closed, intake/exception outcomes are complete, and no unresolved
    closure blockers remain.
17. Normal financial closure occurs from recorded business facts, generates the
    receipt, and seals the order.

## F-02: Reassignment with Changed Terms

Cross-reference: [finance.md](finance.md) for F-04 when reassignment follows a
mobile-money prepayment.

1. Order is assigned to Outlet A.
2. Outlet A rejects, its acceptance window expires, or it marks cannot-fulfill
   before picking.
3. The next eligible outlet, Outlet B, is selected from the fallback list.
4. Every outlet evaluated but filtered out before a full reassignment attempt is
   recorded with a skip reason, such as zone mismatch, no stock, or policy
   mismatch.
5. Skip records give operations and support visibility into why an outlet was
   bypassed without treating it as a full reassignment attempt.
6. Stock is held at Outlet B under `REASSIGNMENT_HOLD`.
7. The hold is unavailable to other orders but is not yet a final outlet
   assignment.
8. Outlet B provisionally confirms it can fulfill under the new terms within the
   configured window.
9. Customer is notified of the changed terms.
10. Customer has a bounded window to accept.
11. If customer accepts, the reassignment hold becomes the active reservation.
12. If customer accepts, the order becomes fully assigned and accepted by Outlet
    B.
13. Outlet A's reservation is released.
14. If customer rejects or does not respond, Outlet B stock hold is released.
15. The normal outcome is order cancellation and any pre-payment becomes refund
    liability, unless active reassignment policy requires escalation.

## Permissions

Trimmed access matrix rows relevant to orders. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-01 | P-02 | P-03 | P-04 | P-05 | P-06 | P-07 | P-08 | P-10 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Cart & order placement | Full | - | - | - | - | - | - | - | Full |
| Order status tracking | Own | Own assigned | Scoped | - | Scoped | Scoped | Scoped | Read assigned outlets | Full |
| Order acceptance / rejection | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Order reassignment escalated | - | - | - | - | - | Scoped exception | - | - | Full |
