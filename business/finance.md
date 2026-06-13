# Finance

**Intent**: Define the financial operations rules for the Nyange platform, covering daily closing, expense controls, delivery cost reporting, internal settlements, and forced financial closure.

**Sources**: §7.8 Daily Closing, §7.9 Expense Controls, §7.19 Delivery Cost Reporting, BI-13, BI-14, F-04, F-05

**Related**: [order.md](order.md) — cross-reference for F-04 reassignment; [payment.md](payment.md) — cross-reference for F-04 payment receipt attribution; [refund.md](refund.md) — refund liabilities in daily closing; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-04 — Financial record entries are write-only.**
Entries in any financial or inventory ledger are never modified or deleted after creation. Errors are corrected by appending compensating entries with explicit linkage to the erroneous entry. The audit trail is always complete and continuous.

**BI-05 — Financial closure is terminal.**
Once an order reaches financial closure, its original sale financial state is sealed. The receipt issued at closure is immutable and is not rewritten by later events. Post-closure financial activity is allowed only as a separate linked business record, such as an approved post-delivery return/refund liability or an approved adjustment/void record; it must not alter the original closure or receipt.

**BI-13 — Internal settlements are accounting entries only.**
When one outlet fulfills an order that another outlet was paid for, the resulting settlement between those outlets is an internal ledger allocation. It does not represent a cash transfer between outlets. The settlement balance remains open until a Finance Officer or Super Admin acknowledges or offsets it, or a Super Admin approves voiding it, through an audited process. Settlement acknowledgement is a finance review marker, not a customer-payment event or revenue-recognition gate. Settlement offset means the balance is netted against another internal settlement balance; it is not a customer payment or physical cash movement. Outlet performance reports recognize revenue under the final fulfilling outlet at financial closure and show settlement status (`POSTED`, `ACKNOWLEDGED`, `OFFSET`, or `VOIDED`) alongside the revenue view.

**BI-14 — Receipt numbers are permanent and sequential.**
Receipt numbers are outlet-scoped and sequential within the outlet/date receipt series, carry the outlet and receipt date in the business number, and are issued at financial closure. Receipt numbers are distinct from order identifiers; an order identifier is not the financial receipt number. A number issued to a valid receipt cannot be reissued or modified. Ordinary financial-closure rollback does not create a numbering gap. Any skipped number range caused by an intentional exception (for example, voided, superseded, imported, or operator-repaired numbers) requires a permanent approved exception and audit record explaining the gap.

**BI-15 — Audit records are permanent.**
Audit log entries are never altered or deleted, regardless of data retention policies applied to the subject records. If a subject entity is archived or purged, its audit history remains. Sensitive administrative changes must record the actor, timestamp, reason where required, and before/after business values, but must not store secrets or payment credentials.

**BI-25 — Launch currency is UGX only.**
All launch prices, cash due, refunds, settlements, expenses, receipts, and financial reports are denominated in UGX. Multi-currency sale, refund, settlement, or expense workflows are not supported at launch; any future currency expansion requires an explicit business policy decision.

---

## Daily Closing (§7.8)

Daily closing is the outlet's end-of-day financial reconciliation.

**What closing summarizes:**
- Cash count reconciliation.
- Posted financial closures and receipts.
- Cash and ledger entries.
- Outstanding refund liabilities or internal settlement balances carried forward with Outlet Manager acknowledgement.

**Closing does not require:**
- Every delivered order to reach `COMPLETED`.
- Every settlement or liability to be resolved before closing.

**Timing (at launch):**
- Each outlet daily closing for an outlet business day is due by 10:00 outlet local time on the next outlet business day.
- If the outlet is not operating on the next calendar day, the closing is due by 10:00 outlet local time on the next operating day.
- If not posted by that deadline, it is overdue until posted.

**Overdue state:**
- An overdue daily closing is an internal alert condition for permissioned daily-closing actors for that outlet.
- It does not by itself remove normal fulfillment access beyond the restrictions listed below.

```
if closing_overdue AND NOT super_admin_urgency_override:
  blocked:  cash_refund_payouts            // Outlet Manager urgent override allowed for this item
  blocked:  large_inventory_adjustments    // above adjustment approval threshold
  blocked:  price_changes_outside_guardrail
  blocked:  manual_financial_ledger_adjustments
  blocked:  manual_payment_reassignment
  blocked:  above_threshold_expense_approval  // >= 100,000 UGX at launch
  allowed:  online_order_placement, pos_walk_in_sales, inventory_reservation
  allowed:  order_acceptance, picking, batching, dispatch, delivery_agent_assignment
  allowed:  cod_collection, mobile_money_verification, stock_intake
  allowed:  within_guardrail_price_changes
```

**Liability and settlement recognition:**
- Overdue daily closing does not block creation or posting of refund liabilities or internal settlement records at financial closure.
- Those records capture business truth and must be posted even when the outlet is behind on closing; the overdue-closing restriction applies to the later cash payout or manual high-risk action.

**Carry-forward:**
- Daily closing must list outstanding customer refund liabilities separately from internal settlement payables and receivables.
- Refund liabilities carry forward until paid, voided, or written off.
- Settlement balances carry forward until a Finance Officer or Super Admin acknowledges or offsets them, or a Super Admin approves voiding them.
- Neither category blocks daily closing.

**Post-payment reassignment reporting:**
- The outlet that received the mobile-money payment lists that receipt, any refund liability it owns, and the settlement payable.
- The final fulfilling outlet's reporting lists the sale, delivery work, direct COD/top-up/delta collections it received, inventory or estimated COGS where configured, and the settlement receivable.

---

## Expense Controls (§7.9)

Outlet expenses are categorized and approval-gated by amount.

```
if expense_amount < 100,000 UGX   → may post without separate review (when active policy allows)
if expense_amount >= 100,000 UGX  → requires approval AND receipt attachment before posting
```

**Rules:**
- Receipt attachments are required above the active threshold and optional below it.
- The active expense approval policy starts from global defaults and may define outlet/category overrides.
- Expense categories are globally configured; outlets may request additions through Customer Support Agent or Super Admin escalation, but they do not manage category definitions directly, and category configuration cannot bypass the active approval policy.
- An Outlet Manager cannot approve their own expense submissions (BI-16).
- Approved expenses post to the outlet's financial ledger immediately.
- Corrections require reversal records, not edits.

**Record fields:**
- Outlet, category, amount, currency, payment method, payee/vendor if known, expense date, recorder, approver where applicable, status, receipt attachment status, and notes.

**Payment method effects:**
- Only cash expenses affect outlet cash-on-hand and daily cash closing.
- All approved expenses affect outlet performance reporting under the same category and threshold policy.

**Reporting:**
- Outlet performance reports may show gross revenue, estimated COGS, gross margin, recorded expenses, and estimated operating margin.
- Margin or profitability values that depend on configured costs or expense controls must be labeled as estimated unless accounting-grade costing and expense controls are approved.
- At launch, daily and weekly sales reports and low-stock reports satisfy reporting needs. Demand forecasting is not supported at launch.

**Cost inputs:**
- Product/outlet cost inputs are optional and manually maintained at launch.
- When configured, they may use outlet-specific values with a global fallback; cost changes are audited and may carry effective dates for historical estimated-margin reporting.
- When configured cost inputs are used for estimated margin reporting, the applicable cost for each completed sale is fixed at financial closure; later cost changes do not silently restate closed-sale estimates.

---

## Internal Settlements (BI-13)

See also: [F-04: Post-Payment Outlet Reassignment](#f-04-post-payment-outlet-reassignment-settled-by-finance) below.

**Settlement states:** `POSTED`, `ACKNOWLEDGED`, `OFFSET`, `VOIDED`

**Rules:**
- Internal settlements between outlets are accounting entries only; they do not represent cash transfers between outlets.
- A Finance Officer (P-09) may acknowledge, query, and offset internal settlements.
- Voiding a settlement requires Super Admin approval with an explicit reason and audit record.
- Finance Officer cannot void unilaterally.
- Outlet performance reports recognize revenue under the final fulfilling outlet at financial closure and show settlement status alongside the revenue view.

---

## Delivery Cost Reporting (§7.19)

Delivery estimated cost is an outlet performance reporting input, not customer-facing delivery pricing. Customer delivery fees, failed-delivery fee waivers, refunds, and cash due are governed by the order and pricing rules in [catalog.md](catalog.md) and [delivery.md](delivery.md).

**Rules:**
- Delivery cost policy/rule changes are audited and effective-dated.
- At launch, delivery cost reporting defaults to treating each order separately.
- Approved run-level cost allocation may use an even split or delivery-fee/cylinder-count weighting across orders.
- Run-level cost allocation finalizes when the delivery run closes.
- Failed orders in batched runs may carry an internal failed-attempt delivery cost for estimated outlet performance reporting, but customer revenue remains zero, no delivery fee is charged to the customer, and failed-attempt cost is reported separately.
- Closed delivery cost allocations are not recalculated directly; corrections require audited adjustment records.
- Reports using delivery cost must label resulting margin or profitability values as estimated unless accounting-grade costing and expense controls are later approved.

---

## F-04: Post-Payment Outlet Reassignment (Settled by Finance)

Cross-references: [order.md](order.md) (reassignment conditions), [payment.md](payment.md) (payment receipt attribution).

1. Customer pays mobile money to Outlet A's merchant account.
2. After payment verification, Outlet A cannot fulfill and the order is reassigned to Outlet B either with no customer-visible change or after the customer accepts any changed terms.
3. Outlet B fulfills successfully.
4. At financial closure: the original payment remains attributed to Outlet A and its payment account; Outlet B holds the revenue from fulfillment only because it successfully completed the order.
5. An internal settlement entry is created for only the prepaid mobile-money value actually received by Outlet A and owed to Outlet B (BI-13). COD deltas, underpayment top-ups, conversion deltas, doorstep price-recalculation deltas, and delivery-fee increases collected by Outlet B post directly to Outlet B's financial ledger and are not settled from Outlet A.
6. If instead the reassigned paid order fails or is cancelled before successful completion, no internal outlet settlement is created; any refund liability remains with Outlet A as the paid-to outlet.
7. Finance Officer or Super Admin reviews and acknowledges the settlement. No cash transfer between outlets occurs at launch.
8. Any reassignment-driven prepaid overage refund liability belongs to Outlet A (the outlet that received the funds), but it is created only after financial closure determines the final amount due. Before financial closure, the customer may see only a pending refund calculation, not a collectible refund. The customer collects that refund from Outlet A only; cross-outlet refund payout is not supported at launch.
9. For reassigned prepaid orders, the customer receipt reflects the fulfillment outlet, the payment-receiving outlet, and the refund-collection outlet when applicable; it does not expose internal settlement status or mechanics. Permissioned Outlet Managers, Area Managers, Finance Officers, and Super Admins may see internal settlement status within their reporting or financial-view scope.

---

## F-05: Forced Financial Closure (Super Admin Exception)

Cross-reference: [order.md](order.md) — pending-closure SLA and escalation path.

1. An order remains in the pending-closure view beyond the SLA window.
2. Escalation reaches Super Admin.
3. Super Admin assigns a resolution path to every open closure blocker, such as waiving with adjustment, marking lost stock, confirming damage, or accepting a cash variance.
4. Required forced-closure approval is obtained under the active materiality policy:

```
material_forced_closure :=
  combined_financial_or_inventory_impact >= 100,000 UGX
  OR any_saleable_filled_cylinder_missing_or_lost
  OR any_refund_or_write_off_created
  OR closure_resolves_more_than_one_blocker_category
// materiality policy starts from global defaults; may define outlet/category overrides

if material_forced_closure  → dual approval required; requester cannot satisfy second approval
else                        → single Super Admin approval within active policy
```

5. After required approval, any compensating inventory and cash adjustment entries are posted with explicit linkage to the forced closure.
6. Loss or variance responsibility defaults to the outlet or delivery run that held custody when the loss, damage, or cash variance occurred. When custody facts identify that holder, the custody-chain result is the default; Super Admin may override responsibility only when custody facts are unclear or conflicting, and only with reason, note, and audit trail. Staff, agent, and approver identities remain visible in custody and audit reports for operational accountability. Resulting losses affect outlet-level reporting, not staff payroll or personal financial liability.
7. The forced-closure audit trail identifies the order, every resolved blocker, approving actor(s), reason, timestamp, the resolution path for each blocker, and any adjustment facts used to correct the business truth.
8. Order moves to `COMPLETED`. All records of the exception are permanent.

---

## Permissions

Trimmed access matrix rows relevant to finance. Full matrix: [identity-auth.md](identity-auth.md).

| Capability | P-06 | P-08 | P-09 | P-10 |
|---|---|---|---|---|
| Daily cash closing | Scoped | – | – | Full |
| Financial ledger (view) | Scoped | Read (assigned outlets) | Full | Full |
| Internal settlement management | – | – | Acknowledge / Offset | Full |
| Expense submission | Scoped | – | – | Full |
| Expense approval | Scoped (Threshold) | – | – | Full |
| Forced financial closure | – | – | – | Full |
| Cross-outlet reporting | – | Read (assigned outlets) | Read | Full |
| Audit log viewing | Scoped | Read (assigned outlets) | Read | Full |

---

## Authorization Edge Cases

**E-06**: A Super Admin performing a forced financial closure must provide a reason. This action is always audit-logged with before/after values. No exception to audit logging exists, even for Super Admin.
