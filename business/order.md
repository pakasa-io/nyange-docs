# Order

**Intent**: Define the order lifecycle, outlet allocation logic, cascade and reassignment rules, cart behaviour, and
mobile money order expiry for the Nyange platform.

**Sources**: §6.1 Order Lifecycle, §7.1 Outlet Allocation, §7.6 Cascade & Reassignment, §7.13 Mobile Money Order Expiry,
§7.15 Cart Behaviour, F-01, F-02

**Related**: [catalog.md](catalog.md) — pricing rules and express delivery fee; [payment.md](payment.md) — payment
lifecycle; [delivery.md](delivery.md) — delivery lifecycle; [inventory.md](inventory.md) — reservation
lifecycle; [finance.md](finance.md) — F-04 post-payment reassignment (settlement), F-05 forced
closure; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-02 — Price snapshot is write-once.**
When an order is placed, the prices in effect at that moment are captured and frozen on the order. Subsequent changes to
pricing rules do not alter the prices on any historical order. All financial calculations downstream (receipts, refunds,
settlements) derive from the frozen snapshot, not from current rules.

**BI-07 — COD orders never require payment verification before fulfillment begins.**
For cash-on-delivery orders, the outlet may begin acceptance, picking, batching, dispatch, and delivery-agent assignment
without any payment action. Payment is collected at the doorstep by the agent.

**BI-09 — Fulfilled orders have no partial completion state.**
Either all items in an order are successfully delivered, or the whole order fails. The only exception is an approved
full-order adjustment that resolves the delivery issue, such as an Outlet Manager-approved refill-to-new-cylinder
conversion at the doorstep; it is still a whole-order mutation, not selective line-level completion.

**BI-17 — Outlet policy is a prerequisite for order acceptance, not a post-hoc check.**
Before an order is assigned to an outlet, the outlet's policies (payment method support, delivery mode support, refill
vendor acceptance, same-vendor refill capability) are evaluated. An order cannot be assigned to an outlet that does not
meet its requirements.

**BI-20 — Order totals are immutable after placement.**
No business event — price change, outlet reassignment, bundle-discount update, or pricing-rule update — can alter the
total of a placed order except through an explicit, audited adjustment workflow that generates its own financial record.
Delivery-fee changes and other post-placement deltas must be represented as explicit adjustments. A customer refund
created by a lower final amount remains a separate refund liability; it does not rewrite the placed order's price
snapshot or original charge.

**BI-21 — Every cancellation must be fully attributed.**
Regardless of who or what initiates a cancellation — customer, outlet staff, policy timeout, or Super Admin — the
cancellation record must capture: the actor identity, the actor's role, a reason code, a reason note, and a timestamp. A
cancellation with any of these fields absent is a data integrity violation.

**BI-22 — Outlet disable is blocked by active orders.**
An outlet cannot be set to inactive or disabled while it has any orders in a non-terminal state. For this rule, terminal
states are `COMPLETED`, `CANCELLED`, `FAILED` for COD orders with no open refund liability, and `REFUNDED`.
`CANCELLED_PENDING_REFUND` is non-terminal — orders in this state also block outlet disable because the financial
liability is still open. Disablement is allowed only after every such order is completed, cancelled, failed as a COD
order with no open refund liability, refunded, or reassigned.

**BI-23 — Critical mutations are idempotent.**
Order creation, cancellation, payment reference submission or reuse, payment verification, refunds, delivery completion,
inventory reservation/adjustment/transfer, cash reconciliation, receipt generation, and comparable one-time state
transitions require idempotency keyed by actor, endpoint, and client key. Identical replays return the original result;
same-key/different-body replays are rejected; in-progress duplicates return a retryable response. Domain uniqueness
rules remain permanent even after operational replay records expire.

---

## State

### Order Lifecycle (§6.1)

```
DRAFT
  └─► PLACED
        ├─► [Mobile Money path]
        │     AWAITING_CUSTOMER_PAYMENT
        │       └─► AWAITING_STAFF_VERIFICATION
        │             └─► STAFF_VERIFIED
        │                   └─► ASSIGNED_TO_OUTLET
        │
        └─► [COD path]
              ASSIGNED_TO_OUTLET
                    │
        ┌───────────┘
        ▼
  AWAITING_OUTLET_ACCEPTANCE   ← both paths enter here (Mobile Money after STAFF_VERIFIED/ASSIGNED_TO_OUTLET and any payment-gate resolution; COD directly)
        ├─► ACCEPTED_BY_OUTLET
        │         └─► PICKING
        │                 └─► READY_FOR_DISPATCH
        │                         └─► OUT_FOR_DELIVERY
        │                                 ├─► DELIVERED ──► COMPLETED
        │                                 └─► FAILED
        │                                       └─► [if mobile-money prepaid] CANCELLED_PENDING_REFUND
        │                                                                            └─► REFUNDED
        │
        └─► [Cascade / Reassignment path]
              ├─► AWAITING_CUSTOMER_REASSIGNMENT_ACCEPTANCE
              │         ├─► (customer accepts) ──► ACCEPTED_BY_OUTLET
              │         └─► (customer rejects / timeout) ──► CANCELLED [or CANCELLED_PENDING_REFUND if prepaid] or REQUIRES_ADMIN_INTERVENTION if policy requires escalation
              │
              └─► [All candidates exhausted]
                    REQUIRES_ADMIN_INTERVENTION
                          ├─► (Super Admin assigns outlet) ──► ACCEPTED_BY_OUTLET
                          └─► (Super Admin cancels or unresolved escalation timer expires without extension) ──► CANCELLED [or CANCELLED_PENDING_REFUND if prepaid]

  CANCELLED                (terminal — order cancelled before any payment, or COD cancelled)
  CANCELLED_PENDING_REFUND (operationally cancelled; refund liability open; awaiting cash payout)
    └─► REFUNDED           (terminal — cash refund payout recorded and posted to outlet ledger)
  FAILED                   (terminal for COD orders; transitions to CANCELLED_PENDING_REFUND for prepaid)
  COMPLETED                (terminal)
```

### State Rules

- Customer can cancel up to and including `PICKING`. `READY_FOR_DISPATCH` cancellation is a pre-custody exception for an
  explicitly permissioned in-scope Customer Support Agent or Super Admin and requires explicit override acknowledgement,
  reason, note, audit, and pick-reversal handling where stock has already been picked. Once delivery-run custody or
  `OUT_FOR_DELIVERY` begins, the normal cancellation route is closed; failed-delivery, return/refund, forced-closure, or
  audited financial-adjustment workflows carry the outcome.
- Outlet staff without explicit cancellation permission cannot use normal cancellation after `PICKING`; they use
  cannot-fulfill, pick-reversal, failed-delivery, or Customer Support Agent escalation workflows according to
  fulfillment state.
- After placement, customer-requested changes to items, delivery address, or original pricing are not supported as order
  modifications. If the customer wants a different order before fulfillment, the existing order is cancelled where
  cancellation is still allowed and the customer places a new order. Approved delivery-conversion, mismatch, refund, or
  reassignment-delta paths are exception adjustments, not edits to the original order request.
- All orders (mobile money and COD) receive an outlet assignment and reserve inventory at placement. For mobile-money
  orders, the status does not transition to `ASSIGNED_TO_OUTLET` until the payment reaches `STAFF_VERIFIED` and the
  payment gate permits fulfillment; before that point, the assigned outlet may see the order for payment-instruction and
  payment-verification purposes only, not for acceptance, picking, batching, dispatch, or delivery-agent assignment. By
  contrast, COD orders transition to `ASSIGNED_TO_OUTLET` immediately after placement.
- Outlet acceptance follows the active outlet acceptance policy. By default at launch, an outlet actor with explicit
  order-acceptance permission must manually accept or reject the order. As an alternative, the system may be configured
  to automatically accept eligible assigned orders without manual outlet action. If the outlet rejects the order or the
  acceptance window expires, the order enters the cascade/reassignment path before picking begins.
- For orders accepted under the no-manual-action policy, failure to start picking within the active picking-start
  timeout enters the same pre-picking cascade/reassignment path.
- `STAFF_VERIFIED` means an authorized outlet payment-verification actor has manually confirmed the payment reference
  against the provider for the assigned outlet. `PAID` means the payment is posted and confirmed. These are distinct: an
  underpayment holds the payment state at `PARTIALLY_PAID` (not yet `PAID`) while a COD top-up is arranged.
- Operational progress (acceptance, picking, batching, dispatch, and delivery-agent assignment) is blocked for
  mobile-money orders until `STAFF_VERIFIED` and any required payment-gate resolution, such as Outlet Manager approval
  of a COD top-up for an underpayment.
- `AWAITING_OUTLET_ACCEPTANCE` is the shared entry state for all order paths. Mobile-money orders enter after
  `STAFF_VERIFIED`, `ASSIGNED_TO_OUTLET`, and any required payment-gate resolution; COD orders enter directly from
  `ASSIGNED_TO_OUTLET`.
- If an accepted outlet marks an order cannot-fulfill before `PICKING`, the order re-enters the cascade/reassignment
  path under §7.6. After `PICKING`, cannot-fulfill is a post-picking exception and must use pick reversal,
  cancellation/refund, failed-delivery, Customer Support Agent routing, or Super Admin handling according to custody
  state.
- `DELIVERED` is customer-facing completion. `COMPLETED` is internal, signifying financial closure. Customers do not see
  `COMPLETED`. A delivery confirmation is issued to the customer immediately when the order reaches `DELIVERED`. The
  financial receipt (with full cost breakdown) is generated and issued only when the order reaches `COMPLETED`. If
  financial closure is delayed by unresolved custody or inventory exceptions, the customer receives the delivery
  confirmation first and the receipt follows when closure is achieved.
- For online deliveries, `COMPLETED` requires delivery-run custody closure, successful financial posting, and no
  unresolved closure blockers. Refill orders additionally require returned-cylinder intake confirmation for every
  expected cylinder or an approved exception. Non-refill orders may move from `DELIVERED` to `COMPLETED` as soon as
  those closure facts are present.
- Normal financial closure occurs from recorded business facts; customers and outlet staff do not manually mark online
  orders `COMPLETED`. Super Admin forced closure remains a separate audited exception workflow before closure effects
  are posted.
- An order in `DELIVERED` but not yet `COMPLETED` appears in a derived internal pending-closure view when unresolved
  closure blockers remain; this does not add a separate order lifecycle state and is not customer-facing. **SLA**: the
  default target is resolution within the same business day. A hard operational alert is raised if the order remains
  unclosed after 24 hours. Super Admin escalation is triggered at 48 hours. If blockers remain unresolved after
  escalation, Super Admin may use forced financial closure only through the exception path in F-05 (
  see [finance.md](finance.md)).
- Each closure blocker identifies the blocker type, owning outlet or workflow, related business record, SLA deadline,
  current status, and any resolution or waiver audit facts; these blockers explain the derived pending-closure view
  without becoming an order status.
- The delivery PIN is a six-digit one-time code generated when the order reaches `READY_FOR_DISPATCH`, not at placement.
  The customer is notified at that point. The short exposure window reduces the risk of PIN leakage before the delivery
  window opens.
- For COD orders, `FAILED` is a terminal state. For mobile-money prepaid orders, `CANCELLED_PENDING_REFUND` is the
  required next state after `FAILED` — the order is financially open until the full prepaid amount is refunded in cash.
  The delivery fee is waived (§7.12; see [delivery.md](delivery.md)). `REFUNDED` is the true terminal for failed prepaid
  deliveries.
- When a paid mobile-money order is cancelled before dispatch, the order moves to `CANCELLED_PENDING_REFUND` —
  operationally cancelled but not financially closed. A refund liability is created immediately. The order reaches
  `REFUNDED` (the true terminal) only after the cash payout is recorded and posted. Unpaid mobile-money orders cancelled
  before dispatch (no payment reference submitted) and COD orders cancelled at any point move directly to `CANCELLED`.

---

## Outlet Allocation (§7.1)

**Geocoding pre-condition:** checkout is blocked if the selected delivery address has no resolved coordinates. Outlet
allocation requires a valid latitude/longitude to calculate distances and determine eligible outlets. If geocoding fails
on address save, the address is saved as `UNRESOLVED`, but checkout remains blocked until coordinates are resolved or
confirmed. Customers may correct the map pin. Any non-customer coordinate correction requires explicit
coordinate-correction permission and audit logging.

An order is allocated to one outlet. The outlet must satisfy all of the following before it is eligible:

1. **Geographic**: at launch, the customer's delivery address falls within one of the outlet's active radius-ring
   service zones, based on the confirmed customer and outlet coordinates. A distance at a ring's minimum boundary is
   eligible for that ring; a distance at its maximum boundary is not. Polygon service zones are out of launch scope.
2. **Operational**: the outlet must be active. For express orders, the outlet must also be currently operating at
   placement time — if no eligible outlet is currently operating, express delivery is unavailable at checkout and is not
   offered as a selectable delivery mode; the customer must choose batched delivery or not place the order. For batched
   orders, the outlet need only have a valid future delivery window available — it does not need to be currently
   operating when the order is placed. Inventory is reserved at placement time regardless.
3. **Delivery mode**: the outlet supports the requested delivery mode (express or batched).
4. **Payment method**: COD and mobile money are both enabled by default at launch unless an outlet policy overrides
   them. The outlet must support the customer's selected payment method. An outlet with no active payment method is
   ineligible for all orders and must be disabled from fulfillment until at least one payment method is active.
5. **Stock availability**: the outlet has sufficient available stock for all items in the order, except for the
   accessory-only relaxation described below when a core-cylinder outlet exists. For refill orders, this gate checks
   outgoing filled cylinders and accessories only; the expected customer empty cylinder is an incoming return and does
   not consume stock at placement.
6. **Refill vendor policy**: for refill items, the outlet accepts the customer's incoming cylinder vendor. If the outlet
   has no configured incoming vendor restriction, it accepts cylinders from all supported vendors by default.
7. **Same-vendor policy**: if the customer requests same-vendor outgoing, the outlet must have that policy enabled and
   the corresponding filled inventory.
8. **Capacity gate (optional)**: if the outlet has a configured active-online-order limit and has reached that limit, it
   is excluded from the candidate set entirely — it does not appear in the ranked list. An outlet below its limit is
   ranked by load as normal. Walk-in orders are not subject to the online order capacity gate and do not count toward
   that limit.

**Service zone templates at launch:**

| Template | Radius                   | Eligibility rule                             |
|----------|--------------------------|----------------------------------------------|
| CORE     | 0 km ≤ distance < 5 km   | Eligible at minimum boundary; not at maximum |
| STANDARD | 5 km ≤ distance < 10 km  | Same                                         |
| EXTENDED | 10 km ≤ distance < 15 km | Same                                         |

Each active online-fulfillment outlet must have confirmed outlet latitude/longitude before it can be eligible for
allocation. Actual outlet coordinates are outlet master data, not hard-coded in this domain specification.

**Priority ranking** (among eligible outlets):

1. Distance to customer (closest first).
2. Delivery fee (lowest first).
3. Current active order load (least loaded first) — counted as active online delivery orders only, from assignment
   through delivery-run closure; walk-in sales do not count toward this load.
4. Outlet priority score (higher score preferred; default 0 when no score is set).

Delivery-agent capacity is not an outlet-allocation factor at launch. An otherwise eligible outlet is not excluded
because no delivery agent is immediately available; delivery assignment happens after outlet allocation under the
delivery lifecycle.

Customers do not choose their outlet. The serving outlet is determined by the active allocation policy.

**Split-order rules:**

- Orders are not split across outlets.
- If no single eligible outlet can provide every requested stock item, allocation may fall back only to the closest
  outlet that satisfies every other outlet-allocation eligibility rule and has all of the order's core cylinder items;
  this fallback relaxes only accessory stock availability.
- The customer may place the order without unavailable accessories or not place the order.
- If an order includes one or more core cylinder items and no single outlet has all of them, the order cannot be placed.
- Accessory-only orders still require one eligible outlet with the requested accessory stock.
- Refill vendor-policy exhaustion is handled separately from core-stock absence: if candidate outlets have the required
  outgoing stock but all are excluded by incoming-vendor acceptance policy, the order follows the exhausted-candidate
  Super Admin intervention path in §7.6 rather than being treated as a stock-unavailable checkout failure.

---

## Cascade & Reassignment (§7.6)

### Unchanged-Terms Cascade

No customer notification required when:

- The first-choice outlet rejects the order, marks it cannot-fulfill before picking, or its acceptance window expires.
- The next eligible outlet can fulfill with unchanged delivery terms (same fee, same window, same vendor).

During unchanged-terms cascade, the order's status remains `AWAITING_OUTLET_ACCEPTANCE` while the outlet assignment is
updated internally to the next candidate.

### Customer Notification and Acceptance Required

Required when a reassignment results in:

- A higher delivery fee or amount due.
- A later or different delivery window.
- A different outgoing cylinder vendor.
- A refill-to-new-cylinder conversion.
- A cancellation/refund-only outcome.
- Any other change that affects what the customer is paying or receiving.

**Sequence when customer notification is required:**

1. Stock is held at the candidate outlet (`REASSIGNMENT_HOLD`).
2. The candidate outlet **provisionally accepts** the changed order within a short window (launch defaults: 5 minutes
   for express, 15 minutes for batched). This is not full operational acceptance.
3. Only after provisional outlet acceptance is the customer notified and asked to approve the changed terms. The
   customer has a bounded acceptance window (launch defaults: 15 minutes for express, 30 minutes for batched).
4. If the customer accepts: the reassignment hold becomes the order's active reservation at the new outlet, the order
   becomes fully assigned and accepted by that outlet, and the previous outlet's reservation is released.
5. If the customer rejects or does not respond: the hold at the new outlet is released; the normal outcome is
   cancellation and any pre-payment creates a refund liability, unless the active reassignment policy requires
   escalation instead.

**Other rules:**

- Delivery-window mismatches between outlets are not silently translated or rebooked during cascade. If the original
  customer-visible window cannot be honored by the candidate outlet, the change follows the customer-acceptance path
  above.
- If the candidate outlet does not provisionally accept within its timeout, the hold is released and the next candidate
  outlet is tried — the customer is never notified about that outlet. Customers are notified only once an outlet has
  confirmed it can fulfil the changed order.
- A stock hold by itself is not outlet acceptance. If no provisional outlet acceptance exists, the order remains in or
  returns to `AWAITING_OUTLET_ACCEPTANCE` rather than moving to customer approval.
- For post-payment reassignment, the customer is not notified merely because the internal fulfilling outlet changed.
  Customer notification is required only when customer-visible details change.

**Candidate skip records:**

- Every outlet evaluated but filtered out before a stock hold or outlet confirmation request records a candidate skip
  reason for operational and support explanation.
- Candidate skip records are diagnostic, not full reassignment attempts.
- They have a configured retention window; at launch, the retention window is 180 days after record creation. After
  that, they may be archived or summarized; terminal order, financial, settlement, refund, and audit records remain
  durable under their own retention rules.

**Reassignment attempt records:**

- A reassignment attempt begins only when stock is held or outlet confirmation is requested. It tracks the candidate
  hold, provisional outlet acceptance or rejection, outlet timeout, customer acceptance or rejection, customer timeout,
  and hold release.
- Once a reassignment attempt reaches a terminal outcome, that outcome is immutable except for appended audit or status
  history.

**Stock reservation on reassignment:**

- For paid or otherwise active reassignment, the candidate outlet hold must be secured before the previous outlet
  reservation is released.
- If the previous reservation cannot release after the new hold succeeds, the previous reservation remains unavailable
  as `PENDING_RELEASE` until the release outcome is corrected and recorded; it is reported separately from normal
  reserved stock.

**Delivery fee adjustments:**

- Any delivery fee difference resulting from reassignment is handled as an explicit price adjustment recorded against
  the order and reassignment record, complying with BI-20 (immutable order totals).
- A cascade that results in a **lower delivery fee** does not require customer acceptance. For unpaid or COD orders, the
  lower fee reduces the amount due. For prepaid mobile-money orders, the difference becomes a refund liability created
  at financial closure. The customer is notified of the reduction but does not need to take any action.
- A reassignment resulting in a **higher delivery fee** requires customer notification and acceptance of the revised
  payment terms before the reassignment activates. After customer acceptance, COD orders collect the fee increase as
  part of the COD amount due at delivery. For prepaid mobile-money orders, the accepted fee increase is collected by the
  delivery agent as a COD delta/top-up at delivery and remains distinct from the original prepaid amount.

**Post-picking block:**

- Reassignment is blocked after picking has begun. Post-picking order reassignment is an Outlet Manager or Super Admin
  exception workflow. Picked stock does not return to available inventory until an authorized inventory/custody actor
  confirms the pick reversal or return outcome.

**All-candidates-exhausted path:**

- When all candidate outlets are exhausted, the order is not immediately cancelled. The order enters
  `REQUIRES_ADMIN_INTERVENTION` for Super Admin intervention, and permissioned Super Admins are notified by push.
- Timeout-driven cancellation/refund-flag outcome applies only if the escalation remains unresolved within (launch
  defaults): Express orders — 30 minutes from escalation; Batched orders — 2 hours from escalation.
- During the intervention window, the Super Admin may manually assign an outlet or cancel the order. A manual assignment
  is an audited exception override of the normal eligibility gates (zone, hours, inventory, outlet policy); it requires
  an explicit reason and does not make those gates optional in the ordinary allocation path.
- For refill vendor-policy exhaustion, intervention may include a Super Admin or permissioned Customer Support Agent
  contacting the customer through approved support or notification paths to offer cancellation-and-reorder alternatives
  such as buying a new cylinder; it does not permit editing the placed order.
- The exhausted-candidate intervention window may be extended only by a Super Admin with reason and audit. A Customer
  Support Agent may request an extension through the support action-request path, but cannot execute the extension.

---

## Mobile Money Order Expiry (§7.13)

Unpaid mobile-money orders have a two-stage configurable expiry window. The clock starts when the order reaches
`AWAITING_CUSTOMER_PAYMENT`.

Launch durations are global defaults; changes or outlet/payment-method overrides require Product Manager approval plus
the operations approval authority required by the active policy.

| Stage                  | Trigger                                                 | Action                                                                                         |
|------------------------|---------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Stage 1 — Warning      | 30 minutes                                              | Reservation-expiry warning notification sent to the customer. Reservation remains active.      |
| Stage 2 — Cancellation | 60 minutes total (30-minute grace period after warning) | Order cancelled and inventory reservation released if no payment reference has been submitted. |

- Submitting a reference at any point before cancellation stops the expiry clock and moves the order to
  `AWAITING_STAFF_VERIFICATION`. The expiry path is permanently closed once a reference is submitted, regardless of the
  verification outcome, and payment-verification delay does not trigger stock release under the unpaid-expiry policy.
- COD orders do not have a payment expiry. They may time out through the outlet acceptance workflow if the outlet does
  not accept within the acceptance window, but no payment deadline applies.

---

## Cart Behaviour (§7.15)

Cross-reference: see [catalog.md](catalog.md) for how catalog or pricing changes affect cart lines.

- Cart creation, cart changes, quotes, and order placement require an authenticated customer account at launch.
  Anonymous guest cart and checkout are not launch behavior.
- Out-of-stock cart items are marked unavailable and remain in the cart until the customer removes them or stock
  returns. The customer must resolve them (remove or wait for stock) before checkout can proceed.
- Cart and checkout quote prices are estimates until order placement. Final pricing is recalculated after outlet
  allocation and is locked only when the order is placed and the price snapshot is created.
- Catalog or pricing changes that affect cart lines — price changes, product disablement, or bundle composition
  changes — require customer review and acknowledgement before cart quote or checkout can proceed.
- If a product's price changes while a cart is open, cart prices update to reflect the current price and the customer
  receives an explicit price-change notice before checkout.
- Disabled products are not orderable or sellable at any outlet, even if stock exists. Any cart item referencing a
  disabled product is marked unavailable. Already-placed orders referencing a disabled product are not affected.
- If a bundle's composition changes while a cart is open (components added, removed, or repriced), any cart line
  containing that bundle is flagged invalid. The customer must review and acknowledge the change before cart quote or
  checkout can proceed.
- Active carts are customer-derived commerce state, not durable order, payment, ledger, custody, receipt, or audit
  facts. They persist across customer sessions and devices, but a cart with no customer fetch, mutation, or quote
  activity for 90 days is marked `ABANDONED`. After an additional 30 days, abandoned cart item detail is no longer
  retained as active cart detail, while a safe cart summary remains. Checked-out carts and placed orders are never part
  of abandoned-cart cleanup.

---

## Post-Delivery Return Policy (§7.14)

Customers may return eligible items or cylinders after delivery subject to all of the following conditions:

1. **Return window**: 7 calendar days from the confirmed delivery timestamp.
2. **Customer drop-off**: the customer is responsible for returning the item or cylinder to the owning outlet in person.
   No reverse-logistics pickup is provided.
3. **Customer identity required**: the customer must be identifiable at the point of return. Anonymous post-delivery
   returns are not accepted.
4. **Outlet inspection and approval**: an authorized outlet return-intake actor must inspect the returned item or
   cylinder, and the active return policy's approval role must approve the condition before a refund liability is
   created or any returned-stock effect is recognized. A return that is rejected on condition does not create a refund
   or change inventory availability.
5. **Cash-only refund**: post-delivery return refunds are paid in cash at the outlet. No mobile-money or alternative
   refund method applies.

Return eligibility, approved condition, refund amount, drop-off requirement, and approval role are controlled by the
active product-type return policy. The launch policy uses the conditions above; later policy changes must preserve
explicit customer identity, approval, and audit requirements.

Post-delivery returns are processed against the original order whenever possible. Unlinked manual refunds are Super
Admin-only exceptions.

---

## Walk-In / POS Rules (§7.11)

- Walk-in orders require explicit POS/cashier permission and share the same inventory as online orders.
- Walk-in stock reservation is immediate and final (no pending state; no delivery delay).
- A walk-in sale reaches financial closure at sale completion and receives its receipt number and immutable receipt
  then.
- At launch, the same pricing rules apply to walk-in and online orders; different POS/online pricing requires a later
  explicit policy decision.
- Operating hours do not apply to walk-in POS; they govern online order allocation only.
- Walk-in refill exchanges use the same incoming/outgoing vendor rules, pricing matrix, same-vendor policy checks, and
  returned-cylinder inspection as online refills. The delivery and agent custody legs are skipped; an explicitly
  permissioned POS/cashier actor handles the exchange directly at the counter. The same all-or-nothing semantics apply:
  if the returned cylinder is unacceptable, that POS/cashier actor may offer a full-order conversion under the same
  approved adjustment path; if the customer refuses conversion, the transaction does not proceed.
- Anonymous walk-in customers are permitted. Refunds, returns, and support follow-up require customer identity.
  Anonymous walk-in mobile-money payments still require provider, transaction reference, and verified amount capture,
  and the same per-provider reference uniqueness rules apply.

---

## F-01: Online Order to Delivery

1. Customer places order from a selected delivery address with confirmed coordinates. The coordinates may be
   provider-resolved or a customer-confirmed manual pin.
2. The serving outlet is selected from outlets that serve the address and meet all allocation criteria (§7.1).
3. Stock for all order items is reserved at the selected outlet.
4. For mobile money: customer pays independently, submits reference, an authorized outlet payment-verification actor
   verifies, and any required payment-gate resolution completes before outlet acceptance (§6.2, BI-03).
5. For COD: order proceeds directly to outlet acceptance.
6. Outlet accepts. An authorized outlet picking actor picks items. Delivery-agent assignment and delivery-run creation
   follow the active outlet delivery policy: assignment by the default policy, with Dispatcher or Outlet Manager manual
   assignment or override before pickup within outlet scope.
7. Outlet handover confirmation and agent receipt confirmation are both recorded; custody transfers to agent.
8. At customer door: for refill orders, agent first records every expected returned cylinder's vendor, size, condition,
   any approved conversion, COD delta or cash collection, and delivery exception. Customer confirms with PIN only when
   the final delivered order and amount due are correct (BI-08).
9. PIN confirmation atomically commits delivery status, order status, outgoing stock commitment, returned-cylinder field
   recording, cash collection recording, and payment status (BI-08).
10. For refill orders: a returned-cylinder intake actor confirms intake for every expected returned cylinder, or a
    failed intake receives an approved exception. For all online deliveries, financial closure waits until delivery-run
    custody is closed, those intake/exception outcomes are complete, and no unresolved closure blockers remain.
11. Normal financial closure occurs from recorded business facts, generates the receipt, and seals the order (BI-05).

## F-02: Reassignment with Changed Terms

Cross-reference: see [finance.md](finance.md) for F-04 when the reassignment follows a mobile-money prepayment.

1. Order is assigned to Outlet A. Outlet A rejects, its acceptance window expires, or it marks cannot-fulfill before
   picking.
2. The next eligible outlet (Outlet B) is selected from the fallback list. Every outlet evaluated but filtered out
   before a full reassignment attempt is recorded with a skip reason (e.g., zone mismatch, no stock, policy mismatch).
   This gives operations and support visibility into why an outlet was bypassed without treating it as a full
   reassignment attempt.
3. Stock is held at Outlet B under a `REASSIGNMENT_HOLD` reservation; it is unavailable to other orders but not yet a
   final outlet assignment.
4. Outlet B provisionally confirms it can fulfill under the new terms (within a short window).
5. Customer is notified of the changed terms. Customer has a bounded window to accept.
6. If customer accepts: the reassignment hold becomes the order's active reservation; the order becomes fully assigned
   and accepted by Outlet B. Outlet A's reservation released.
7. If customer rejects or does not respond: stock hold at Outlet B released; the normal outcome is order cancellation
   and any pre-payment becomes refund liability, unless the active reassignment policy requires escalation instead.

---

## Permissions

Trimmed access matrix rows relevant to orders. Full matrix: [identity-auth.md](identity-auth.md).

| Capability                     | P-01 | P-02           | P-03   | P-04 | P-05   | P-06                              | P-07   | P-08                    | P-10 |
|--------------------------------|------|----------------|--------|------|--------|-----------------------------------|--------|-------------------------|------|
| Cart & order placement         | Full | –              | –      | –    | –      | –                                 | –      | –                       | Full |
| Order status tracking          | Own  | Own (assigned) | Scoped | –    | Scoped | Scoped                            | Scoped | Read (assigned outlets) | Full |
| Order acceptance / rejection   | –    | –              | –      | –    | –      | Scoped (with explicit permission) | –      | –                       | Full |
| Order reassignment (escalated) | –    | –              | –      | –    | –      | Scoped (exception)                | –      | –                       | Full |
