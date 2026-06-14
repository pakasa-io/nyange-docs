# Finance

**Intent**: Define financial operations for daily closing, expense controls,
delivery cost reporting, receipt issuance, and cash custody reporting.

**Reader task**: Use this document to determine how financial records are
posted, sealed, reviewed, carried forward, or reported.

**Sources**: §7.8 Daily Closing, §7.9 Expense Controls, §7.19 Delivery Cost
Reporting, BI-14

**Related**:
[order.md](order.md) for order placement, terminal states, COD fulfillment, and
cancellation;
[payment.md](payment.md) for cash payment facts;
[refund.md](refund.md) for refund liabilities;
[identity-auth.md](identity-auth.md) for the full access matrix.

## Invariants

**BI-04 — Financial record entries are write-only.**

- Entries in any financial or inventory ledger are never modified or deleted
  after creation.
- Errors are corrected by appending compensating entries with explicit linkage
  to the erroneous entry.
- The audit trail is always complete and continuous.

**BI-05 — Financial closure is `DELIVERED`.**

- For online COD orders, financial closure means the order reaches
  `DELIVERED`.
- `DELIVERED` seals the original sale financial state.
- The receipt issued at `DELIVERED` is immutable.
- Later events do not rewrite the original receipt.
- Later financial activity is allowed only as a separate linked business record,
  such as an approved refund liability or approved adjustment/void record.
- Later activity must not alter the original delivered sale or receipt.

**BI-13 — Prepayment settlement is not a launch workflow.**

- Launch online orders are COD cash, collected by the Delivery Agent for the
  fulfilling outlet.
- No launch workflow records one outlet as the external prepayment receiver and
  another outlet as the fulfillment outlet.
- External prepaid payment workflows require explicit settlement rules before
  they can enter scope.

**BI-14 — Receipt numbers are permanent and sequential.**

- Receipt numbers are outlet-scoped and sequential within the outlet/date
  receipt series.
- Receipt numbers carry the outlet and receipt date in the business number.
- Receipt numbers are issued at `DELIVERED`.
- Receipt numbers are distinct from order identifiers.
- A number issued to a valid receipt cannot be reissued or modified.
- Ordinary receipt rollback does not create a numbering gap.
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
- delivered orders and issued receipts;
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
            price_changes_outside_guardrail
            manual_financial_ledger_adjustments
            above_threshold_expense_approval  // >= 100,000 UGX at launch
  allowed:  online_order_placement, order_claiming, inventory_reservation_for_claim
  allowed:  ready_for_pickup, delivery_agent_assignment, pickup, cod_collection
  allowed:  stock_intake, within_guardrail_price_changes
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

### Deferred Prepayment Reporting

- External payment receipts and merchant-account attribution are not launch
  reporting requirements.
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
  applicable cost for each delivered sale is fixed at `DELIVERED`.
- Later cost changes do not silently restate sealed-sale estimates.

## Internal Settlements

External prepaid settlement is outside launch scope.

No ordinary launch workflow creates a cross-outlet customer-payment settlement
because online delivery payment is COD cash collected for the fulfilling outlet.

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

Trimmed access matrix rows relevant to finance. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-06 | P-08 | P-09 | P-10 |
| --- | --- | --- | --- | --- |
| Daily cash closing | Scoped | - | - | Full |
| Financial ledger view | Scoped | Read assigned outlets | Full | Full |
| Expense submission | Scoped | - | - | Full |
| Expense approval | Scoped threshold | - | - | Full |
| Cross-outlet reporting | - | Read assigned outlets | Read | Full |
| Audit log viewing | Scoped | Read assigned outlets | Read | Full |
