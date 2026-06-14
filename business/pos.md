# POS Sale

**Intent**: Define launch walk-in POS sale behavior for outlet counter sales,
immediate cash collection, stock commitment, returned-cylinder receipt, receipt
issuance, and same-day voids.

**Reader task**: Use this document to decide whether an outlet staff actor can
create, complete, abandon, void, or read a walk-in POS sale.

**Sources**: F-19 resolution; 2026-06-14 walk-in POS MVP grill decisions;
prior mobile-money deferral note that launch payment remains cash-only.

**Related**:
[catalog.md](catalog.md) for POS item priceability and price rules;
[inventory.md](inventory.md) for stock availability, stock commitment, and
returned-cylinder recognition;
[payment.md](payment.md) for immediate POS cash payment facts;
[finance.md](finance.md) for outlet cash custody, ledger posting, daily closing,
and receipt issuance;
[refund.md](refund.md) for refund-liability boundaries;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Invariants

**BI-28 - POS sale is not an online order.**

- Walk-in POS sales are separate from online delivery orders.
- POS sales do not enter cart, order placement, pending pool, outlet claiming,
  delivery assignment, COD expectation, or delivery lifecycle states.
- POS sale public identifiers are distinct from online order identifiers.

**BI-29 - POS completion is atomic.**

- POS completion commits sale state, stock effects, payment fact, Finance cash
  custody basis, ledger basis, and receipt issuance in one transaction.
- If any participant rejects the mutation or the transaction fails before
  commit, no receipt number is issued and the POS sale remains uncompleted or
  abandoned.

**BI-30 - POS cash is immediate outlet cash.**

- Walk-in POS is cash-only at launch.
- Exact cash is required before POS completion.
- POS does not support COD, customer credit, mobile money, partial payment,
  short collection, or over collection at launch.

**BI-31 - POS receipts are immutable.**

- Finance issues the immutable receipt at POS completion.
- The original receipt is never rewritten.
- Corrections use linked void and reversal records.

## Boundary

- POS Sale owns walk-in counter-sale lifecycle state, public POS sale number,
  line snapshot, sale completion, abandonment, same-day void eligibility, and
  POS sale audit fields.
- POS Sale does not own catalog price rules, inventory ledger entries, payment
  facts, Finance cash custody, financial ledger entries, receipt records,
  refund liabilities, customer accounts, or online order state.
- Catalog owns whether POS line items are priceable and saleable.
- Inventory owns stock availability, stock commitment, returned-cylinder
  recognition, and inventory ledger posting.
- Payment owns the immediate `POS_CASH` collected payment fact and any linked
  POS payment void fact.
- Finance owns outlet cash custody, financial and cash ledger posting, daily
  closing, receipt records, and receipt void audit.
- Refund does not own POS same-day voids at launch.
- Identity owns staff actor identity, session assurance, and outlet scope facts.

## Lifecycle

```
DRAFT -> COMPLETED
DRAFT -> ABANDONED
COMPLETED -> VOIDED
```

Terminal states: `ABANDONED`, `VOIDED`. `COMPLETED` is terminal after the
same-day void window closes without a void.

| State | Meaning |
| --- | --- |
| `DRAFT` | Outlet Cashier is building a counter sale that has not committed cash, stock, payment, ledger, or receipt effects. |
| `COMPLETED` | POS sale completion committed all participant effects and Finance issued the immutable receipt; same-day pre-closing void may still be allowed. |
| `ABANDONED` | Incomplete draft was discarded before completion; no cash, stock, payment, ledger, or receipt effect exists. |
| `VOIDED` | Completed sale was voided through the same-outlet, same-business-day, pre-closing void path with linked reversal records. |

## Business Rules

### Sale Creation

- A POS sale is outlet-local.
- An Outlet Cashier may create POS sale drafts only for their assigned outlet.
- A POS sale draft has an internal draft identifier before completion.
- A completed POS sale receives one immutable public `POS-%08d` sale number.
- `POS-%08d` numbers are distinct from `ORD-%08d` online order numbers and from
  Finance receipt numbers.
- POS sale creation does not require a registered customer account.
- A POS sale may be anonymous.
- Optional customer name or phone may be captured only for receipt lookup or
  voluntary customer contact.
- Optional POS customer details do not create an Identity account, customer
  permission, online order read access, or refund collection eligibility.

### Item Scope

- Launch POS may sell:
  - new filled cylinder purchases;
  - refill exchange sales;
  - accessories.
- POS uses the same globally supported launch products, vendors, cylinder
  sizes, accessory SKUs, bundle rules, tax rules, and priceability rules as the
  launch catalog.
- POS does not use delivery fees.
- POS does not use online serviceability by address.
- POS does not create delivery tasks or delivery-agent custody.

### Pricing

- Catalog computes POS line prices and totals from active launch catalog and
  pricing rules.
- Outlet-local product, refill, accessory, or tax price overrides are not launch
  behavior for POS.
- Client- or cashier-submitted line totals, discounts, taxes, and totals are not
  authoritative.
- POS completion snapshots product, price, tax, total, outlet, cashier, payment,
  and receipt references.
- The POS snapshot is write-once after completion.

### Inventory Participation

- POS may commit only stock that is saleable and available at the cashier's
  outlet at completion time.
- POS does not reserve stock before completion.
- POS completion atomically commits outgoing stock with the sale.
- Partial POS completion is prohibited.
- If any line cannot be fulfilled from available outlet stock, completion is
  rejected and no participant effect is committed.
- POS refill exchange completion requires the customer to physically surrender
  the expected empty cylinder at the counter before completion.
- The cashier records returned-cylinder vendor, size, and condition for each
  refill exchange line before completion.
- POS returned cylinders do not enter Delivery-owned field-leg states or
  Inventory `INTAKE_PENDING`.
- Inventory recognizes accepted POS returned cylinders as confirmed empty stock
  in the same POS completion commit.
- Unknown, unsupported, damaged, or mismatched returned cylinders block POS
  refill exchange completion unless an Inventory-owned launch exception rule
  explicitly allows the condition.

### Cash And Payment

- The cashier must collect exact UGX cash before completing the POS sale.
- Payment records one immediate `POS_CASH` collected payment fact at POS
  completion.
- POS does not create a `PENDING_COLLECTION` COD expectation.
- POS does not create a `ZERO_COLLECTION` fact.
- POS short collection, over collection, and post-completion price adjustment
  are not launch behavior.

### Completion Commit

POS Sale completion is the trigger. Participant ownership remains module-scoped:

| Participant | Owned mutation in the completion commit |
| --- | --- |
| POS Sale | Move `DRAFT -> COMPLETED`, assign public POS sale number, and persist the completed sale snapshot. |
| Catalog | Provide authoritative line pricing, tax, and saleability basis. |
| Inventory | Commit outgoing stock and recognize accepted POS returned cylinders for refill exchange lines. |
| Payment | Record the immediate `POS_CASH` collected payment fact. |
| Finance | Record outlet cash custody basis, create ledger basis, and issue the immutable receipt number and receipt record. |

- The same idempotency key and correlation ID apply to every participant
  mutation in the completion commit.
- Identical completion replays return the original completed POS sale and
  receipt.
- Same-key/different-body completion replays are rejected.
- If any participant rejects the mutation or the transaction fails before
  commit, all participant mutations fail together, no receipt number is issued,
  no stock is committed, no payment fact is recorded, and no outlet cash custody
  row is created.

### Abandonment

- A cashier may abandon a `DRAFT` POS sale before completion.
- Abandonment records actor, outlet, timestamp, reason when provided,
  idempotency key, and correlation ID.
- Abandonment has no inventory, payment, Finance ledger, cash custody, receipt,
  or refund effect.

### Same-Day Void

- After receipt issuance, a POS sale may be voided only by an Outlet Manager for
  the same outlet or by Super Admin.
- A POS void is allowed only on the same outlet business day and before that
  outlet's daily closing is posted.
- A POS void requires reason code, reason note, actor identity, role, outlet,
  original POS sale ID, original payment fact, original inventory movements,
  original Finance ledger/cash custody basis, original receipt number, void
  idempotency key, correlation ID, and audit timestamp.
- A POS void appends linked reversal records across POS Sale, Payment,
  Inventory, Finance cash custody, Finance ledger, and receipt audit.
- The original receipt remains immutable and is not rewritten.
- A POS void may return cash to the customer as a same-transaction reversal, but
  it does not create a Refund-owned liability, collection code, or refund payout
  workflow.
- POS voids after daily closing are not launch behavior.
- Customer returns, exchanges, post-sale price adjustments, and POS refund
  liabilities are not launch behavior.

## Permissions

POS Sale owns authorization decisions for POS-owned commands and reads. Related
rows are shown here for context; rows enforced by another aggregate remain with
that aggregate. Identity supplies actor, session, grant, and outlet-scope facts
from [identity-auth.md](identity-auth.md).

| Capability | P-03 | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- | --- |
| POS sale creation and completion | Scoped | - | - | - | Full |
| POS sale read/reporting | Scoped | Scoped | Read assigned outlets | Read | Full |
| POS same-day void | - | Scoped | - | - | Full |

## Non-Goals

- Mobile-money POS payments.
- Customer credit, layaway, deposits, or accounts receivable.
- POS delivery, dispatch, route, or doorstep workflows.
- POS customer returns, exchanges, post-sale price adjustments, or refund
  liabilities.
- Offline POS completion.
- Serial-number-specific cylinder tracking.
