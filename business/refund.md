# Refund

**Intent**: Define the refund liability lifecycle, collection code rules, and payout constraints for all customer refund scenarios on the Nyange platform.

**Source**: §6.6 Refund Lifecycle

**Related**: [order.md](order.md) — cancellation paths that create refund liabilities; [payment.md](payment.md) — overpayment and underpayment; [delivery.md](delivery.md) — failed delivery fee waiver; [finance.md](finance.md) — refund liabilities in daily closing; [identity-auth.md](identity-auth.md) — full access matrix

---

## Invariants

**BI-12 — A refund liability persists until discharged.**
A refund owed to a customer (from overpayment, failed delivery, pre-delivery cancellation of paid orders, reassignment-driven prepaid overage, prepaid doorstep price-recalculation overage, or post-delivery return) remains an open liability on the outlet's financial record until a cash payout event is recorded or a Super Admin-approved void/write-off path resolves it with reason, note, and audit trail. Code expiry, daily closing, and time passing do not discharge the liability, and an open liability does not convert to revenue. Open refund liabilities appear in the owning outlet's daily closing and liability reports until paid, voided, or written off.

**BI-18 — A refund collection code is single-use and perishable.**
The code issued to a customer to collect a cash refund at an outlet has a finite validity window; at launch, that window is 24 hours after issuance. The code is invalidated upon first successful presentation. Failed verification attempts are rate-limited per refund and outlet actor and audit-logged with actor, outlet, refund ID, timestamp, and safe failure reason. After a configured refund-code attempt threshold, the code is temporarily locked and requires regeneration or unlock by a permissioned Customer Support Agent or Super Admin with reason and audit trail. The refund liability remains outstanding while the code is locked, and no payout may be recorded until the code is unlocked or regenerated.

---

## State

```
PENDING_CREATION (liability condition: overpayment, failed delivery, pre-delivery cancellation of paid orders, reassignment-driven prepaid overage, prepaid doorstep price-recalculation overage, post-delivery return)
  └─► LIABILITY_OPEN (refund owed; collection code not yet issued)
        ├─► [Below approval-required threshold; launch default below UGX 50,000]
        │     └─► COLLECTIBLE (collection code issued; customer can present at outlet)
        └─► [At or above approval-required threshold; launch default UGX 50,000 or more]
              PENDING_APPROVAL (awaiting Outlet Manager approval within launch default UGX 500,000 approval threshold, or Super Admin approval above that threshold)
                ├─► COLLECTIBLE (approved; collection code issued)
                └─► LIABILITY_OPEN (approval not granted; liability remains open until later approved and paid, voided, or written off through a Super Admin-approved path)

  COLLECTIBLE
    ├─► PAID (customer presented code; outlet cash dispensed; terminal)
    ├─► CODE_EXPIRED (code expired; liability remains open; new code can be issued)
    │         └─► COLLECTIBLE (new code issued)
    └─► CODE_LOCKED (rate-limit threshold exceeded; payout prohibited until unlock or regeneration)
              └─► COLLECTIBLE (permissioned Customer Support Agent or Super Admin unlocks or regenerates; liability remains open)

  VOIDED (Super Admin-approved void; terminal; requires reason, note, and audit)
  WRITTEN_OFF (Super Admin writes off uncollectable liability; terminal; requires reason, note, and audit)
```

---

## Business Rules

**Cash-only refunds:**
- At launch, all customer refunds are cash-only regardless of original payment method.
- Provider-issued refunds, customer wallet credit, and mobile-money refund workflows are not supported.
- When the original customer payment was mobile money, the cash refund is treated as an outlet cash payout against that mobile-money-paid order for closing and reporting; it does not create a provider refund, customer credit, or internal transfer.

**Pre-dispatch cancellation of paid mobile-money orders:**
- When a paid mobile-money order is cancelled before dispatch, a refund liability is created immediately.
- The order moves to `CANCELLED_PENDING_REFUND` and the refund lifecycle begins from `LIABILITY_OPEN`.
- The order does not reach `REFUNDED` (its true terminal) until the cash payout is recorded and the liability is discharged.

**Approval thresholds (at launch):**
- Refund liabilities below UGX 50,000 become collectible without separate approval.
- Refund liabilities at or above UGX 50,000 require approval before a collection code is issued.
- Outlet Managers may approve refund liabilities from UGX 50,000 through UGX 500,000 within their outlet scope.
- Above UGX 500,000, Super Admin approval is required.
- The active refund approval policy starts from global defaults and may define outlet/refund-reason overrides.

**Collection code expiry and regeneration:**
- Refund collection codes expire after 24 hours after issuance at launch.
- A permissioned Customer Support Agent or Super Admin can regenerate an expired code with audit.
- Regeneration when the customer loses access requires customer verification through a linked support case with reason and audit.
- The refund liability itself does not expire and carries forward regardless of code expiry.

**Collection by original account holder:**
- Only the original account holder may collect a refund at launch. Delegated or proxy collection is not a standard supported workflow.
- Any exceptional override requires a linked support case, Customer Support Agent routing or Super Admin handling under the active exception policy, explicit reason and evidence notes, and a full audit trail.
- Outlet Managers cannot independently approve proxy collection.

**Payout process:**
- At payout, the Outlet Manager or explicitly permissioned Outlet Cashier, acting within the active refund payout limits, must:
  1. Verify the one-time collection code.
  2. Confirm the collector is the original account holder tied to the order.
  3. Confirm the customer/order phone matches before cash is marked paid.

**Payout limits (at launch):**

| Actor | Per-refund limit | Per-outlet-business-day limit |
|---|---|---|
| Outlet Cashier (with explicit permission) | UGX 100,000 | UGX 300,000 |
| Outlet Manager | UGX 500,000 | UGX 1,500,000 |
| Above Outlet Manager limit | Super Admin release approval required before cash is disbursed | — |

**Payout record requirements:**
- Every payout attempt must capture: actor, role, outlet, amount, order/refund ID, reason, approval status, active threshold policy version, and timestamp.
- Paid cash events must also capture the customer/order identity snapshot and attestation note.

**Delivery agents:**
- Delivery Agents do not issue cash refunds. When a delivery fails, they record the failure and return physical goods; any refund remains in the outlet cash-refund lifecycle.

**Collection code reveal:**
- The collection code is revealed only through the authenticated customer experience or audited reveal by a permissioned Customer Support Agent or Super Admin after customer verification.
- It must not appear in push notifications, email bodies, or SMS.
- An Outlet Manager or explicitly permissioned Outlet Cashier may verify a customer-presented code, but they cannot reveal the active code to the customer.

**Daily closing:**
- Daily closing may complete with open refund liabilities, but the Outlet Manager must explicitly acknowledge them and carry them forward.

**Outlet responsibility:**
- The outlet that received the original payment is responsible for disbursing the refund. Another outlet cannot pay the refund on the payment-receiving outlet's behalf.

**Post-delivery returns:**
- For post-delivery returns (§7.14), a refund liability is created only after outlet inspection and approval by the active return policy's approval role.
- Returns rejected on condition do not enter the refund lifecycle and do not create a liability.

---

## Permissions

Trimmed access matrix rows relevant to refunds. Full matrix: [identity-auth.md](identity-auth.md).

| Capability | P-01 | P-03 | P-06 | P-07 | P-10 |
|---|---|---|---|---|---|
| Refund initiation | Own (request) | – | Scoped (Threshold) | Request | Full |
| Refund approval | – | – | Scoped (Threshold) | – | Full |
| Refund payout (cash at outlet) | – | Scoped (with explicit permission; payout limits) | Scoped (within payout limits) | – | Full |
| Refund collection code management | – | – | – | Scoped (with explicit permission) | Full |

---

## Authorization Edge Cases

**E-05**: Refund liabilities at or above the approval-required threshold require approval from the Outlet Manager for that outlet within their active approval threshold, or from a Super Admin above that threshold. At launch, the approval-required threshold is UGX 50,000 and the Outlet Manager approval threshold is UGX 500,000 within outlet scope. The approving actor cannot approve a refund on an order where they submitted the refund request themselves.

**E-07**: An Outlet Manager can approve refunds for their own outlet within their authorized threshold and approval policy. At launch, Outlet Managers may approve refund liabilities from UGX 50,000 through UGX 500,000 within their outlet scope; refunds above that threshold require Super Admin approval. Outlet Managers cannot approve refunds at another outlet, and they cannot approve their own submitted refund request.
