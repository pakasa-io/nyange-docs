# Finance

**Intent**: Define financial operations for cash custody, cash handover,
variance records, daily closing, expense controls, delivery cost reporting, and
receipt issuance.

**Reader task**: Use this document to determine how financial records are
posted, sealed, reviewed, carried forward, or reported.

**Sources**: §7.7 Agent Cash Handling, §7.8 Daily Closing, §7.9 Expense
Controls, §7.19 Delivery Cost Reporting, BI-14

**Related**:
[order.md](order.md) for order placement, terminal states, COD fulfillment, and
cancellation;
[pos.md](pos.md) for walk-in POS sale completion and same-day voids;
[payment.md](payment.md) for cash payment facts;
[delivery.md](delivery.md) for doorstep collection evidence and delivery task
state;
[refund.md](refund.md) for refund liabilities;
[identity-auth.md](identity-auth.md) for personas, authentication, session
assurance, permission-grant facts, and outlet-scope facts.

## Invariants

**BI-04 — Finance ledger entries are write-only.**

- Entries in Finance-owned financial and cash ledgers are never modified or
  deleted after creation.
- Errors are corrected by appending compensating entries with explicit linkage
  to the erroneous entry.
- The audit trail is always complete and continuous.
- Inventory owns inventory ledger entries and their append-only invariant in
  [inventory.md](inventory.md).
- Shared append-only ledger storage or tooling is infrastructure and does not
  transfer logical ledger ownership between modules.

**BI-05 — Financial closure is sale completion.**

- For online COD orders, financial closure means the order reaches
  `DELIVERED`.
- For walk-in POS sales, financial closure means POS Sale reaches `COMPLETED`.
- `DELIVERED` seals the original sale financial state.
- `COMPLETED` seals the original POS sale financial state.
- The receipt issued at `DELIVERED` is immutable.
- The receipt issued at POS `COMPLETED` is immutable.
- Finance participates in Delivery-coordinated completion by issuing the
  immutable receipt number and receipt record in the same atomic commit that
  marks the order `DELIVERED`.
- Finance participates in POS Sale completion by issuing the immutable receipt
  number and receipt record in the same atomic commit that completes the POS
  sale.
- Later events do not rewrite the original receipt.
- Later financial activity is allowed only as a separate linked business record,
  such as an approved refund liability or approved adjustment/void record.
- Later activity must not alter the original delivered online sale, completed
  POS sale, or receipt.

**BI-13 — Cash belongs to the responsible outlet.**

- Launch online orders are COD cash, collected by the Delivery Agent for the
  fulfilling outlet.
- Launch POS sales are immediate cash collected by the selling outlet.

**BI-14 — Receipt numbers are permanent and sequential.**

- Receipt numbers are outlet-scoped and sequential within the outlet/date
  receipt series.
- Receipt numbers carry the outlet and receipt date in the business number.
- Receipt numbers are issued at online order `DELIVERED` or POS sale
  `COMPLETED`.
- Receipt numbers are distinct from order identifiers and POS sale identifiers.
- A number issued to a valid receipt cannot be reissued or modified.
- Receipt rollback means pre-commit technical rollback of the same `DELIVERED`
  or POS `COMPLETED` transaction before a receipt number is issued.
- If the `DELIVERED` or POS `COMPLETED` transaction fails before commit, the
  receipt number is not issued and no numbering gap exists.
- After commit, receipt rollback is not allowed; corrections use separate linked
  adjustment or void records.
- Any skipped number range caused by an intentional exception requires a
  permanent approved exception and audit record explaining the gap.

**BI-15 — Audit records are permanent.**

- Audit log entries are never altered or deleted.
- Data retention policies applied to subject records do not delete audit log
  entries.
- If a subject entity is archived or purged, its audit history remains.
- Sensitive administrative changes must record actor, timestamp, reason where
  required, and before/after business values.
- Audit logs must not store authentication or financial secrets.

**BI-25 — Launch currency is UGX only.**

- All launch prices, cash due, refunds, expenses, receipts, and financial
  reports are denominated in UGX.
- Multi-currency sale, refund, or expense workflows are not supported at
  launch.
- Future currency expansion requires an explicit business policy decision.

## Boundary

- Finance owns cash custody after a COD collection fact is committed, cash
  handover, counted-cash acceptance, outlet cash custody after POS completion,
  cash variance records, Finance-owned financial and cash ledger posting, daily
  closing, receipt records, expense controls, and delivery cost reporting.
- Finance owns the short/over COD collection variance policy and the variance
  records linked from Payment.
- Delivery owns doorstep collection evidence, delivery task state, field facts,
  and handover submission evidence.
- Payment owns expected COD amount, collected cash fact, zero-collection fact,
  POS cash fact, payment outcome, and the link to any Finance-owned cash
  variance record.
- Order owns order state and the immutable order total that determines expected
  COD.
- POS Sale owns POS sale lifecycle state, same-day void eligibility, and the POS
  sale snapshot that determines expected POS cash.
- Refund owns customer refund liabilities and payout lifecycle.

## Cash Custody And Handover

Finance owns the physical COD cash custody lifecycle after the
Delivery-coordinated completion commit records a COD collection fact.

```
AGENT_CASH_HELD -> HANDOVER_SUBMITTED -> OUTLET_CASH_ACCEPTED
HANDOVER_SUBMITTED -> HANDOVER_VARIANCE_OPEN -> OUTLET_CASH_ACCEPTED
```

| State | Meaning |
| --- | --- |
| `AGENT_CASH_HELD` | Delivery completion committed a COD collection fact; the assigned Delivery Agent physically holds the cash for the fulfilling outlet. |
| `HANDOVER_SUBMITTED` | The Delivery Agent submitted handover evidence and counted cash for Finance-owned acceptance. |
| `HANDOVER_VARIANCE_OPEN` | Counted handover cash does not match the Finance handover basis; Finance owns the discrepancy record and reconciliation path. |
| `OUTLET_CASH_ACCEPTED` | A permitted receiver accepted counted cash into outlet cash custody for daily closing and ledger posting. |

### COD Collection Variance Policy

Agents are expected to collect the exact COD due at the doorstep. Short or over
collection is allowed only through a Finance-owned COD collection variance
record.

```
if abs_variance_per_order <= 10,000 UGX
   AND shift_cumulative_variance <= 50,000 UGX  -> Outlet Manager approval
else                                             -> Super Admin approval
```

- Required approval must be recorded before Delivery can mark the order
  complete.
- An approved short/over collection variance does not change expected COD or the
  frozen order total.
- Payment stores the collected amount and a link to the approved variance; it
  does not own the variance record.
- Over-collection excess is recorded as a Finance-owned cash discrepancy.
- Finance may post a cash over-collection correction source event that creates a
  Refund-owned liability for the excess amount.
- Post-collection price adjustment is not a launch Finance workflow.
- Any future post-collection adjustment workflow must name its source owner,
  authorization rule, posting rule, and Refund handoff contract before it can
  create a refund liability.

### Handover Rules

- The Delivery Agent must submit cash handover evidence live.
- A permitted Outlet Manager or Super Admin may receive handover within scope.
- The receiver records counted cash before accepting custody into outlet cash.
- Handover acceptance transfers cash custody from the Delivery Agent to outlet
  cash.
- If counted handover cash differs from the Finance handover basis, Finance
  records a `HANDOVER_VARIANCE_OPEN` discrepancy.
- Open cash discrepancy or unaccounted item blocks full shift close until
  resolved by an approved Finance-owned variance, correction, or adjustment
  record.

### POS Cash Custody

- POS Sale completion transfers collected cash directly into Finance-owned
  outlet cash custody for the selling outlet.
- POS cash does not enter `AGENT_CASH_HELD`, `HANDOVER_SUBMITTED`, or
  `HANDOVER_VARIANCE_OPEN`.
- POS cash has no delivery-agent handover.
- POS completion records exact cash only.
- POS short or over collection is blocked before completion and does not create
  a COD variance record.
- An allowed POS same-day void appends linked Finance cash custody, ledger, and
  receipt-audit reversal records.
- POS void cash returned to the customer is a same-day void reversal, not a
  Refund-owned payout.

## Daily Closing

Daily closing is the outlet's end-of-day financial reconciliation.

### Summarized Records

Daily closing summarizes:

- cash count reconciliation;
- delivered online orders;
- completed POS sales and POS voids;
- issued receipts;
- cash and ledger entries;
- outstanding refund liabilities carried forward with Outlet Manager
  acknowledgement.

### Non-Requirements

Daily closing does not require every refund liability to be resolved.

### Timing

- Each outlet daily closing for an outlet business day is due by 10:00 outlet
  local time on the next outlet business day.
- If the outlet is not operating on the next calendar day, closing is due by
  10:00 outlet local time on the next operating day.
- If not posted by the deadline, the closing is overdue until posted.

### Overdue Behavior

- An overdue daily closing is an internal alert condition for permissioned
  daily-closing actors for that outlet.
- Overdue closing does not by itself remove normal fulfillment access beyond the
  restrictions listed here.

```
if closing_overdue:
  blocked unless super_admin_cash_refund_payout_urgency_override:
            cash_refund_payouts
  blocked unless owning_policy_action_specific_urgency_override:
            large_inventory_adjustments
            manual_financial_ledger_adjustments
            above_threshold_expense_approval  // >= 100,000 UGX at launch
  allowed:  online_order_placement, order_claiming, inventory_reservation_for_claim
  allowed:  claim_block_marking, claim_block_reopening, unclaimable_closure
  allowed:  ready_for_pickup, delivery_agent_assignment, pickup, cod_collection
  allowed:  pos_sale_completion, same_day_pos_void_before_current_day_closing
  allowed:  stock_intake
```

- Only Super Admin urgency override can allow blocked cash refund payouts while
  daily closing is overdue.
- The cash-refund-payout urgency override does not unlock any other blocked
  action.
- Any other overdue-closing override must be explicitly named by the owning
  policy for that action.

### Liability Recognition

- Overdue daily closing does not block creation or posting of refund liabilities
  at `DELIVERED` or from an approved adjustment.
- Those records capture business truth and must be posted even when the outlet
  is behind on closing.
- The overdue-closing restriction applies to later cash payout or manual
  high-risk action.

### Carry-Forward

- Daily closing lists outstanding customer refund liabilities separately.
- Refund liabilities carry forward until paid, voided, or written off.
- Open refund liabilities do not block daily closing.

### COD Reporting

- The fulfilling outlet reports COD collections, delivery work, inventory or
  estimated COGS where configured, refund liabilities it owns, and cash
  reconciliation facts.

## Expense Controls

Outlet expenses are categorized and approval-gated by amount.

```
if expense_amount < 100,000 UGX   → may post without separate review
if expense_amount >= 100,000 UGX  → requires approval AND receipt attachment
```

### Business Rules

- Posting without separate review applies only when the active policy allows it.
- Receipt attachments are required above the active threshold and optional below
  it.
- The active expense approval policy starts from global defaults and may define
  outlet/category overrides.
- Expense categories are globally configured.
- Outlets may request category additions through Super Admin escalation.
- Outlets do not manage category definitions directly.
- Category configuration cannot bypass the active approval policy.
- An Outlet Manager cannot approve their own expense submissions.
- Approved expenses post to the outlet's financial ledger immediately.
- Corrections require reversal records, not edits.

### Required Fields

Expense records capture:

- outlet;
- category;
- amount;
- currency;
- payment method;
- payee/vendor when known;
- expense date;
- recorder;
- approver when applicable;
- status;
- receipt attachment status;
- notes.

### Payment Method Effects

- Only cash expenses affect outlet cash-on-hand and daily cash closing.
- All approved expenses affect outlet performance reporting under the same
  category and threshold policy.

### Reporting

- Outlet performance reports may show gross revenue, estimated COGS, gross
  margin, recorded expenses, and estimated operating margin.
- Margin or profitability values that depend on configured costs or expense
  controls must be labeled estimated unless accounting-grade costing and expense
  controls are approved.
- At launch, daily and weekly sales reports satisfy reporting needs.
- Demand forecasting is not supported at launch.

### Cost Inputs

- Product/outlet cost inputs are optional and manually maintained at launch.
- When configured, cost inputs may use outlet-specific values with a global
  fallback.
- Cost changes are audited and may carry effective dates for historical
  estimated-margin reporting.
- When configured cost inputs are used for estimated margin reporting, the
  applicable cost for each delivered online sale is fixed at `DELIVERED`.
- When configured cost inputs are used for estimated margin reporting, the
  applicable cost for each POS sale is fixed at POS `COMPLETED`.
- Later cost changes do not silently restate sealed-sale estimates.

## Delivery Cost Reporting

Delivery estimated cost is an outlet performance reporting input. It is not
customer-facing delivery pricing.

Customer delivery fees, failed-delivery fee waivers, refunds, and cash due are
governed by [order.md](order.md), [catalog.md](catalog.md), and
[delivery.md](delivery.md).

### Business Rules

- Delivery cost policy/rule changes are audited and effective-dated.
- Delivery cost reporting treats each delivery task as a single-order task.
- Failed delivery tasks may carry an internal failed-attempt delivery cost for
  estimated outlet performance reporting.
- For failed orders, customer revenue remains zero, no delivery fee is charged
  to the customer, and failed-attempt cost is reported separately.
- Closed delivery cost entries are not recalculated directly.
- Corrections require audited adjustment records.
- Reports using delivery cost must label resulting margin or profitability
  values as estimated unless accounting-grade costing and expense controls are
  approved.

## Permissions

Finance owns authorization decisions for Finance-owned commands and reads.
Related rows are shown here for context; rows enforced by another aggregate
remain with that aggregate. Identity supplies actor, account, grant, and
outlet-scope facts from [identity-auth.md](identity-auth.md).

| Capability | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- |
| Daily cash closing | Scoped | - | - | Full |
| Financial ledger view | Scoped | Read assigned outlets | Full | Full |
| Expense submission | Scoped | - | - | Full |
| Expense approval | Scoped threshold | - | - | Full |
| Cross-outlet reporting | - | Read assigned outlets | Read | Full |
| Audit log viewing | Scoped | Read assigned outlets | Read | Full |
