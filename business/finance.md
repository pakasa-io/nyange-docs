# Finance

**Intent**: Define launch financial operations for daily closing, expense
controls, delivery cost reporting, and forced financial closure.

**Reader task**: Use this document to determine how financial records are
posted, sealed, reviewed, carried forward, or force-closed.

**Sources**: §7.8 Daily Closing, §7.9 Expense Controls, §7.19 Delivery Cost
Reporting, BI-14, F-05

**Related**:
[order.md](order.md) for reassignment and pending-closure behavior;
[payment.md](payment.md) for cash payment facts;
[refund.md](refund.md) for refund liabilities;
[identity-auth.md](identity-auth.md) for the full access matrix;
[../out-of-scope/2026-06-13-mobile-money-payments.md](../out-of-scope/2026-06-13-mobile-money-payments.md)
for deferred mobile-money payment and settlement scope.

## Invariants

**BI-04 — Financial record entries are write-only.**

- Entries in any financial or inventory ledger are never modified or deleted
  after creation.
- Errors are corrected by appending compensating entries with explicit linkage
  to the erroneous entry.
- The audit trail is always complete and continuous.

**BI-05 — Financial closure is terminal.**

- Once an order reaches financial closure, its original sale financial state is
  sealed.
- The receipt issued at closure is immutable.
- Later events do not rewrite the original receipt.
- Post-closure financial activity is allowed only as a separate linked business
  record, such as an approved post-delivery return/refund liability or approved
  adjustment/void record.
- Post-closure activity must not alter the original closure or receipt.

**BI-13 — Mobile-money settlement is not a launch workflow.**

- Launch online orders are COD cash, collected by the Delivery Agent for the
  fulfilling outlet.
- No launch workflow records one outlet as the mobile-money payment receiver and
  another outlet as the fulfillment outlet.
- Post-payment outlet reassignment settlement for mobile-money-paid orders is
  deferred with mobile-money payments.
- Future prepaid or cross-outlet payment workflows require explicit settlement
  rules before they can enter scope.

**BI-14 — Receipt numbers are permanent and sequential.**

- Receipt numbers are outlet-scoped and sequential within the outlet/date
  receipt series.
- Receipt numbers carry the outlet and receipt date in the business number.
- Receipt numbers are issued at financial closure.
- Receipt numbers are distinct from order identifiers.
- A number issued to a valid receipt cannot be reissued or modified.
- Ordinary financial-closure rollback does not create a numbering gap.
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

## Daily Closing

Daily closing is the outlet's end-of-day financial reconciliation.

### Summarized Records

Daily closing summarizes:

- cash count reconciliation;
- posted financial closures and receipts;
- cash and ledger entries;
- outstanding refund liabilities carried forward with Outlet Manager
  acknowledgement.

### Non-Requirements

Daily closing does not require:

- every delivered order to reach `COMPLETED`;
- every refund liability to be resolved.

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
if closing_overdue AND NOT super_admin_urgency_override:
  blocked:  cash_refund_payouts
  blocked:  large_inventory_adjustments
  blocked:  price_changes_outside_guardrail
  blocked:  manual_financial_ledger_adjustments
  blocked:  above_threshold_expense_approval  // >= 100,000 UGX at launch
  allowed:  online_order_placement, pos_walk_in_sales, inventory_reservation
  allowed:  order_acceptance, picking, batching, dispatch, delivery_agent_assignment
  allowed:  cod_collection, stock_intake
  allowed:  within_guardrail_price_changes
```

- Outlet Manager urgent override is allowed for cash refund payouts when policy
  permits it.

### Liability Recognition

- Overdue daily closing does not block creation or posting of refund liabilities
  at financial closure.
- Those records capture business truth and must be posted even when the outlet
  is behind on closing.
- The overdue-closing restriction applies to later cash payout or manual
  high-risk action.

### Carry-Forward

- Daily closing lists outstanding customer refund liabilities separately.
- Refund liabilities carry forward until paid, voided, or written off.
- Open refund liabilities do not block daily closing.

### Deferred Payment Settlement Reporting

- Mobile-money payment receipts, merchant-account attribution, and
  post-payment outlet reassignment settlement are not launch reporting
  requirements.
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
- Outlets may request category additions through Customer Support Agent or Super
  Admin escalation.
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
- At launch, daily and weekly sales reports plus low-stock reports satisfy
  reporting needs.
- Demand forecasting is not supported at launch.

### Cost Inputs

- Product/outlet cost inputs are optional and manually maintained at launch.
- When configured, cost inputs may use outlet-specific values with a global
  fallback.
- Cost changes are audited and may carry effective dates for historical
  estimated-margin reporting.
- When configured cost inputs are used for estimated margin reporting, the
  applicable cost for each completed sale is fixed at financial closure.
- Later cost changes do not silently restate closed-sale estimates.

## Internal Settlements

Internal settlement for mobile-money-paid post-payment reassignment is deferred
from launch scope. See
[../out-of-scope/2026-06-13-mobile-money-payments.md](../out-of-scope/2026-06-13-mobile-money-payments.md).

No ordinary launch workflow creates a cross-outlet customer-payment settlement
because online delivery payment is COD cash collected for the fulfilling
outlet.

## Delivery Cost Reporting

Delivery estimated cost is an outlet performance reporting input. It is not
customer-facing delivery pricing.

Customer delivery fees, failed-delivery fee waivers, refunds, and cash due are
governed by [order.md](order.md), [catalog.md](catalog.md), and
[delivery.md](delivery.md).

### Business Rules

- Delivery cost policy/rule changes are audited and effective-dated.
- At launch, delivery cost reporting defaults to treating each order separately.
- Approved run-level cost allocation may use an even split or
  delivery-fee/cylinder-count weighting across orders.
- Run-level cost allocation finalizes when the delivery run closes.
- Failed orders in batched runs may carry an internal failed-attempt delivery
  cost for estimated outlet performance reporting.
- For failed orders, customer revenue remains zero, no delivery fee is charged
  to the customer, and failed-attempt cost is reported separately.
- Closed delivery cost allocations are not recalculated directly.
- Corrections require audited adjustment records.
- Reports using delivery cost must label resulting margin or profitability
  values as estimated unless accounting-grade costing and expense controls are
  approved.

## F-05: Forced Financial Closure

Cross-reference: [order.md](order.md) for pending-closure SLA and escalation.

1. An order remains in the pending-closure view beyond the SLA window.
2. Escalation reaches Super Admin.
3. Super Admin assigns a resolution path to every open closure blocker, such as
   waiving with adjustment, marking lost stock, confirming damage, or accepting
   a cash variance.
4. Required forced-closure approval is obtained under the active materiality
   policy.

```
material_forced_closure :=
  combined_financial_or_inventory_impact >= 100,000 UGX
  OR any_saleable_filled_cylinder_missing_or_lost
  OR any_refund_or_write_off_created
  OR closure_resolves_more_than_one_blocker_category
// materiality policy starts from global defaults and may define
// outlet/category overrides

if material_forced_closure  → dual approval required
else                        → single Super Admin approval within active policy
```

- The requester cannot satisfy second approval.
- After required approval, compensating inventory and cash adjustment entries are
  posted with explicit linkage to the forced closure.
- Loss or variance responsibility defaults to the outlet or delivery run that
  held custody when the loss, damage, or cash variance occurred.
- When custody facts identify that holder, the custody-chain result is the
  default.
- Super Admin may override responsibility only when custody facts are unclear or
  conflicting, and only with reason, note, and audit trail.
- Staff, agent, and approver identities remain visible in custody and audit
  reports for operational accountability.
- Resulting losses affect outlet-level reporting, not staff payroll or personal
  financial liability.
- The forced-closure audit trail identifies the order, every resolved blocker,
  approving actor(s), reason, timestamp, resolution path for each blocker, and
  any adjustment facts used to correct business truth.
- The order moves to `COMPLETED`.
- All records of the exception are permanent.

## Permissions

Trimmed access matrix rows relevant to finance. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- |
| Daily cash closing | Scoped | - | - | Full |
| Financial ledger view | Scoped | Read assigned outlets | Full | Full |
| Expense submission | Scoped | - | - | Full |
| Expense approval | Scoped threshold | - | - | Full |
| Forced financial closure | - | - | - | Full |
| Cross-outlet reporting | - | Read assigned outlets | Read | Full |
| Audit log viewing | Scoped | Read assigned outlets | Read | Full |

## Authorization Edge Cases

**E-06**: A Super Admin performing a forced financial closure must provide a
reason. This action is always audit-logged with before/after values. No
exception to audit logging exists, even for Super Admin.
