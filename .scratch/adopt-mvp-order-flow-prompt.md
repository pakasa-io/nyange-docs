# Prompt: Rewrite Order Checkout–Fulfillment Flow

## Task

Rewrite `business/order.md` to define the authoritative order lifecycle for
online COD orders. Then audit and update every other file in `business/` for
consistency. Stale states, removed workflows, and out-of-scope capabilities
must be cut from every canonical file — not softened, qualified, or left as
forward references.

`.scratch/mvp-order-checkout-fulfillment-flow.md` is a scratch input. Read it,
extract every rule, decision, and constraint from it, bake them into the
canonical files as first-class permanent content, then delete the scratch file.
No canonical document may reference the scratch file, use the word "MVP", or
carry any qualifier that marks content as temporary, proposed, or provisional.

## Read first

1. Read `.scratch/mvp-order-checkout-fulfillment-flow.md` completely.
2. Read every file in `business/` completely.
3. Read `start-here/doc-style.md` for style and chunk-shape rules.

## Rewrite `business/order.md`

Replace the file entirely. Every section must be grounded in what the scratch
input specifies. Do not carry forward any content without a counterpart there.

### Intent and boundary

- Intent: placement, lifecycle state, outlet claiming, COD fulfillment, and
  cancellation.
- Boundary section: derive directly from the scratch input's "Boundary Rules".
  Written as permanent canonical rules.
- Related links: only aggregates that participate in this flow.

### Invariants

Keep and rewrite as canonical rules exactly these BI entries:

| BI | Rule |
| --- | --- |
| BI-02 | Price snapshot is write-once |
| BI-07 | No pre-payment gate before fulfillment |
| BI-09 | All-or-nothing delivery; no partial completion |
| BI-20 | Order totals immutable after placement |
| BI-21 | Every cancellation fully attributed (update covered states) |
| BI-22 | Outlet disable blocked by active orders (update terminal states) |
| BI-23 | Critical mutations idempotent (update covered transitions) |

Remove BI-17 entirely. BI-01 and BI-06 are owned by `inventory.md`;
cross-reference them only.

### Lifecycle

```text
PENDING
  -> CLAIMED
  -> READY_FOR_PICKUP
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PENDING|CLAIMED|READY_FOR_PICKUP -> CUSTOMER_CANCELLED
CLAIMED|READY_FOR_PICKUP -> PENDING   # outlet claim cancel before pickup
OUT_FOR_DELIVERY -> DELIVERY_FAILED
```

Terminal states: `DELIVERED`, `CUSTOMER_CANCELLED`, `DELIVERY_FAILED`.

Include a state-meaning table for every state.

### State rules

Write precise canonical rules for each phase, sourced directly from the scratch
input.

**Checkout (`POST /orders` → `PENDING`)**

- Authenticated customer account required.
- Delivery address must have resolved coordinates; unresolved address blocks
  checkout.
- Rejected inputs: disabled products, unavailable cart lines, unpriceable
  catalog combinations.
- Server computes all line totals and order totals; client values are ignored.
- Snapshots product, price, delivery-fee, tax, and total fields; snapshot is
  write-once.
- Stores structured delivery address.
- Allocates immutable `ORD-%08d` public order number.
- Grants customer order-scoped read, status, and cancel permissions.
- No stock is reserved at checkout; reservation occurs at claim.
- While `PENDING`, permitted outlets may read the pending pool with
  delivery-area detail redacted.
- Pending-pool reads must not expose: full delivery address, recipient name,
  recipient phone, or delivery instructions.

**Claiming (`PENDING → CLAIMED`)**

- Any active permitted outlet may claim from the pending pool.
- Claim is whole-order only; partial item claims are prohibited.
- Claiming reserves all requested stock atomically.
- At most one active claim may exist per order.
- A claimed order requires an active reservation; claim and reservation are
  inseparable.
- No automatic outlet allocation, ranking, or cascade.

**Ready for pickup (`CLAIMED → READY_FOR_PICKUP`)**

- Claimed outlet marks the order ready.
- Fulfillment creates a `READY_FOR_PICKUP` delivery task.
- Ready-order notification fanout is attempted.

**Out for delivery (`READY_FOR_PICKUP → OUT_FOR_DELIVERY`)**

- A delivery agent is assigned to the task; assignment grants agent order-scoped
  read access to the full delivery address for the duration of the assignment.
- Pickup: keeps reserved stock unavailable, creates full-cylinder custody for
  the agent, moves task to `PICKED_UP`, moves order to `OUT_FOR_DELIVERY`.

**Delivery completion (`OUT_FOR_DELIVERY → DELIVERED`)**

- Requires `acknowledged_cash_collected=true`.
- COD derives from the persisted order total; client cannot override it.
- Records returned-cylinder field facts when applicable.
- Records COD collection fact.
- Commits outgoing stock.
- Completes the claim.
- Marks task and order `DELIVERED`.
- All of the above commit in a single transaction.
- `DELIVERED` is the order terminal state. There is no subsequent
  financial-closure state.

**Alternate paths**

- *Outlet claim cancellation before pickup*: releases reservation, cancels any
  pre-pickup delivery task, cancels the active claim, clears the claimed outlet,
  returns the order to `PENDING`.
- *Customer cancellation before pickup*: terminal. Releases reservation, cancels
  active claim, cancels any pre-pickup delivery task, records required
  cancellation attribution, marks order `CUSTOMER_CANCELLED`. Order never
  returns to pending.
- *Delivery failure after pickup*: returns all picked-up full-cylinder custody
  to outlet stock, records zero-collection payment fact, completes the claim,
  marks task `FAILED`, marks order `DELIVERY_FAILED`.

Do not write rules for: outlet allocation ranking, cascade, reassignment,
admin intervention, batch delivery windows, delivery PIN, `COMPLETED` financial
closure state, post-delivery returns, walk-in POS, or payment expiry.

### Main flow

Numbered operational steps derived from the scratch input's "Main Flow" steps
1–8. Do not label it "MVP" or frame it as simplified.

### Out of scope

`## Out of Scope` section. Restate the list from the scratch input as permanent
canonical scope exclusions. Plain declarative bullets; no framing that suggests
these were ever considered or under discussion.

### Permissions

Remove rows for: outlet acceptance/rejection, reassignment escalation. Ensure
rows exist for: cart and order placement, order status tracking, outlet
claiming, delivery assignment, delivery completion, COD recording, and
cancellation. Map all scope notes to the new state names.

---

## Update `business/finance.md`

Finance has significant exposure to removed concepts. Make every change below.

### Related links

Remove "reassignment and pending-closure behavior" from the `order.md` link
description. Replace with the correct scope of what order.md still governs.

### BI-05 — Financial closure is terminal

`DELIVERED` is the terminal state and the closure event. Rewrite BI-05 so
"financial closure" means reaching `DELIVERED`. The receipt is issued at
`DELIVERED`. Remove any reference to `COMPLETED` as a separate closure state.

### BI-14 — Receipt numbers

"Receipt numbers are issued at financial closure" → issued at `DELIVERED`.
Update that line and any other occurrence.

### Daily closing — Non-Requirements

"every delivered order to reach `COMPLETED`" is no longer a non-requirement to
state because `COMPLETED` does not exist. Remove or rewrite that bullet to
reflect the current terminal state.

### Daily closing — overdue code block

Remove `pos_walk_in_sales` from the `allowed` list (walk-in POS is out of
scope). Remove `order_acceptance`, `picking`, `batching`, `dispatch` from the
`allowed` list (those stages no longer exist in the order lifecycle). Keep only
operations that apply to the new flow.

### Cost Inputs

"fixed at financial closure" → "fixed at `DELIVERED`". Update every occurrence
in this section.

### F-05: Forced Financial Closure

Remove this entire section. It is predicated on `COMPLETED` state, the
pending-closure SLA view, and the escalation chain that no longer exists.
`DELIVERED` is terminal; there is no pending-closure state to escalate.
Remove the cross-reference to F-05 in the Sources line in the file header.

### Authorization Edge Cases

Remove E-06 (it documents forced financial closure audit requirements, which
are removed with F-05).

---

## Update `business/identity-auth.md`

Identity-auth embeds removed capabilities into persona descriptions and the
access matrix. Remove them everywhere.

### P-01 Customer

Remove "walk-in POS transactions" from the persona description. Customers pay
cash at the doorstep for online COD delivery.

### P-02 Delivery Agent

Remove "enters customer-provided delivery PINs" from the persona description.
Delivery PIN is out of scope.

### P-03 Outlet Cashier

Walk-in POS and delivery PIN fallback are both out of scope. If P-03 has no
remaining in-scope responsibilities after removing those, remove the persona
entirely. If other in-scope responsibilities exist, remove all walk-in POS and
delivery PIN references and update the description to reflect only what remains.

### P-05 Dispatcher

Remove "Creates and manages delivery batches" — batch delivery is out of scope.
Update the description to reflect delivery assignment and tracking for
single-order delivery tasks.

### P-06 Outlet Manager

Remove "Accepts or rejects orders" — manual outlet acceptance/rejection is
replaced by outlet claiming. Update to reflect the claiming workflow.

### P-07 / Customer Support Agent (wherever it appears)

Remove "delivery PIN fallback and refund collection-code regeneration" from the
delivery PIN part. Keep refund collection-code regeneration if the refund
workflow remains in scope. Remove any other delivery-PIN references.

### P-10 Super Admin

Remove "forced financial closures" from the persona description or update it to
reflect that forced closure is no longer a defined workflow.

### Access matrix

Remove these capability rows entirely:
- `POS / walk-in sales`
- `Order reassignment escalated`
- `Forced financial closure`

Update these rows to use the new order lifecycle states and capabilities:
- Any row referencing outlet acceptance/rejection → update to outlet claiming
- Any row referencing `PICKING`, `READY_FOR_DISPATCH`, `COMPLETED` → update or
  remove

### Authorization edge cases

Remove any edge case that references forced financial closure, delivery PIN
fallback, walk-in POS, or post-picking reassignment exceptions.

---

## Update `business/delivery.md`

- Replace all old order-state references (`READY_FOR_DISPATCH`, `PICKING`,
  `ASSIGNED_TO_OUTLET`, `AWAITING_OUTLET_ACCEPTANCE`, `COMPLETED`, etc.) with
  counterparts in the new lifecycle wherever delivery references order state.
- Remove any delivery rules predicated on order flows that no longer exist
  (delivery PIN, batch run orchestration tied to picking state, etc.).

---

## Update `business/payment.md`

- COD is collected at doorstep during `OUT_FOR_DELIVERY` and recorded at
  `DELIVERED`. Update any payment rules that reference order state to use the
  new state names.
- Remove references to prepayment, payment-reference expiry, or financial
  closure states (`COMPLETED`).

---

## Update `business/inventory.md`

- Reservation occurs when an outlet claims an order (`PENDING → CLAIMED`), not
  at order placement. Update the reservation lifecycle and every rule that
  states or implies reservation at placement.
- Update all order-state references to the new lifecycle.

---

## Update `business/refund.md`

### Post-Delivery Returns subsection

Post-delivery returns are out of scope. Remove the "Post-Delivery Returns"
subsection (the block beginning "For post-delivery returns, a refund liability
is created only after outlet inspection…").

### Permissions table

If P-03 Outlet Cashier is removed from `identity-auth.md`, remove the P-03
column from the refund permissions table. If P-03 retains in-scope
responsibilities, keep the column.

---

## Update `business/support.md`

Remove "delivery PIN fallback or reveal actions allowed by delivery.md" from
the Launch Support Actions list. Delivery PIN is out of scope. Keep all other
listed fallback actions that remain in scope.

---

## Scan `business/notifications.md` and `business/catalog.md`

Grep both files for: `PLACED`, `ASSIGNED_TO_OUTLET`, `AWAITING`, `PICKING`,
`READY_FOR_DISPATCH`, `COMPLETED`, `cascade`, `reassignment`, `walk.in`, `POS`,
`delivery PIN`, `financial closure`, `post.delivery`. Fix any matches using the
same rules above. If no matches exist, leave the file unchanged.

---

## Cleanup

After all canonical files are written and verified, delete
`.scratch/mvp-order-checkout-fulfillment-flow.md`.

---

## Style

- Doc-style chunk shape from `start-here/doc-style.md`.
- Preserve all BI-xx, F-xx, E-xx IDs that remain in scope; remove those that
  do not.
- Reproduce ASCII diagrams exactly.
- Precise, concise, direct, neutral. No narrative filler. Bullets and tables
  over prose.

---

## Verification checklist

Fail any item and fix before reporting done.

- [ ] `business/order.md` lifecycle diagram matches the state machine above
      character-for-character.
- [ ] BI-17 does not appear in any canonical file.
- [ ] F-05 does not appear in any canonical file.
- [ ] E-06 does not appear in any canonical file.
- [ ] `COMPLETED` does not appear as an order state in any canonical file.
- [ ] `READY_FOR_DISPATCH`, `PICKING`, `ASSIGNED_TO_OUTLET`,
      `AWAITING_OUTLET_ACCEPTANCE`, `ACCEPTED_BY_OUTLET`,
      `REQUIRES_ADMIN_INTERVENTION` do not appear in any canonical file.
- [ ] "walk-in POS", "walk-in sales", "POS" do not appear in any canonical
      file.
- [ ] "delivery PIN" does not appear in any canonical file.
- [ ] "cascade", "reassignment" (in the outlet-allocation sense), and
      "post-delivery return" do not appear in any canonical file.
- [ ] "financial closure" appears only in the rewritten BI-05 where it now
      means `DELIVERED`, nowhere as a reference to a separate `COMPLETED` state.
- [ ] `inventory.md` reservation trigger is outlet claim, not order placement.
- [ ] No canonical file contains the word "MVP", "scratch", "proposal",
      "simplified", or "temporary".
- [ ] No canonical file references `.scratch/` in any link or note.
- [ ] `.scratch/mvp-order-checkout-fulfillment-flow.md` has been deleted.
