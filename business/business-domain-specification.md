# Nyange Platform — Domain Specification
## For: Distinguished Principal Engineer

**Version**: 2026-05-24  
**Intent**: Communicate business invariants, domain lifecycles, authorization boundaries, and access patterns. Architecture, technology stack, and implementation patterns are left entirely to the implementing engineer.

---

# 1. Domain in One Page

Nyange distributes gas cylinders (6kg, 12kg) and accessories through company-owned outlets. Customers can order online for delivery from the nearest eligible outlet or buy through walk-in POS at an outlet. Orders are either new cylinder purchases, cylinder refill exchanges, or accessory purchases, and online orders can mix them in one cart.

**Two payment rails:** mobile money (pre-pay, staff-verified against provider reference) and cash on delivery (collected at doorstep by agent).

**Two delivery modes:** express (as soon as possible) and batched (customer selects a delivery window).  Walk-in POS sales have no delivery leg.

**Refill exchange:** customer surrenders an empty cylinder and receives a filled one. The outgoing cylinder defaults to the outlet's configured default vendor when present, otherwise to the global Vengas default. The customer may choose another currently fulfillable outgoing vendor, including same-vendor where outlet policy allows it and the outlet has stock. Pricing varies by incoming vendor, outgoing vendor, and cylinder size.

**Outlet independence:** each outlet is a company-owned branch within the same legal entity, operating as an outlet-level accountability unit — its own inventory, pricing (within guardrails), staff, payment accounts, and cash ledger. Outlet independence is operational and reporting-oriented, not legal or financial separation: outlet cash is company cash, and outlets are not franchisees or marketplace merchants. Outlet stocking is vendor-specific and outlet-owned: outlets may hold filled cylinders from any globally supported vendor, not only Vengas, and each outlet procures stock directly from vendors rather than from a central warehouse or central purchasing pool. An order always belongs to exactly one outlet. Financial performance is tracked per outlet.

---

# 2. Personas

These ten personas are the canonical business archetypes used in this specification. They describe typical responsibilities and access patterns, but they are not identity account categories.

Runtime authorization is determined by explicit permissions and scope assignments. A single human account may hold multiple permission bundles across one or more outlets.

---

## P-01 Customer

An individual who orders gas, refills, and accessories through the platform.

**Context**: Logs in before using cart or checkout at launch, and places orders online from a registered account. Uses the authenticated customer experience to track order status and access customer-visible notices. May have multiple saved addresses. Pays by mobile money or cash. Collects cash refunds in person at the outlet.

**Cannot**: Manage outlet operations, view other customers' data, approve payments, access financial records, or override business rules or policy controls.

---

## P-02 Delivery Agent

A field operative assigned to deliver orders from outlet to customer.

**Context**: Sees only orders assigned to them and customer phone numbers for assigned active deliveries. Performs required field actions during active delivery. Collects cash (COD), records returned cylinders, and enters customer-provided delivery PINs for confirmation. Reports delivery failures. Submits cash handover at end of shift.

**Cannot**: Reassign orders, view other agents' assignments, verify mobile-money references, modify inventory, issue refunds, or access financial records.

---

## P-03 Outlet Cashier

An outlet-level staff member responsible for walk-in POS transactions.

**Context**: Records walk-in sales through the outlet POS workflow. Accepts cash or mobile-money from walk-in customers. May also verify mobile-money payment references for the outlet and assist with delivery PIN fallback when explicitly permissioned. Issues walk-in receipts. Has no delivery responsibilities, and any online-order responsibilities are limited to explicitly permissioned payment-verification and PIN-fallback workflows.

**Cannot**: Manage online orders, approve refunds, adjust inventory, or access another outlet's data.

---

## P-04 Inventory Clerk

An outlet-level staff member responsible for stock management.

**Context**: Manages stock counts, submits inventory adjustment requests, confirms returned-cylinder intake after delivery, initiates outlet-to-outlet transfer requests, and records vendor refill movements. Works within their assigned outlet only.

**Cannot**: Accept/reject orders, verify payments, issue refunds, access financial records, or approve their own adjustment requests.

---

## P-05 Dispatcher

An outlet-level staff member responsible for delivery coordination.

**Context**: Creates and manages delivery batches, assigns delivery agents to orders and runs, tracks delivery progress, and may reassign agents before pickup. Schedules delivery work within the outlet's active delivery windows and policy.

**Cannot**: Verify payments, approve refunds, adjust inventory, access financial records, or modify outlet policies.

---

## P-06 Outlet Manager

The operational owner of a single outlet.

**Context**: Accepts or rejects orders, verifies mobile-money payment references when explicitly permissioned, reconciles daily cash, approves refund liabilities for their outlet according to the active approval policy, submits above-threshold inventory adjustments for approval, oversees staff, manages delivery operations, and views outlet-specific reports. Has elevated authority over P-03, P-04, and P-05 within their outlet.

**Cannot**: Access another outlet's data (unless explicitly granted), change global prices or catalog, approve refunds outside their outlet scope, or override Super Admin controls.

---

## P-07 Customer Support Agent

A support operative who handles complaints, escalations, and exceptions.

**Context**: Opens and manages support cases. Views case-linked, outlet-scoped order, delivery, payment, and inventory details needed for investigation. Requests approved workflows (refunds, inventory corrections, reassignments) on behalf of the resolution — but does not execute those workflows directly. Where explicitly permissioned, performs support-only fallback actions such as delivery PIN fallback and refund collection-code regeneration, unlock, or audited customer-verified reveal. Logs customer communication summaries.

**Cannot**: Directly mutate orders, payments, inventory, or financial records outside explicitly permissioned fallback actions. Cannot approve, pay, void, or write off refunds. Cannot approve their own action requests. Cannot view cases outside their outlet scope unless explicitly granted cross-outlet access.

---

## P-08 Area Manager

A regional oversight role assigned to multiple outlets.

**Context**: Monitors performance and escalates exceptions across assigned outlets. Has read access to outlet operations, inventory, and financial summaries for all assigned outlets. Can escalate exceptions but does not perform direct outlet operations.

**Cannot**: Perform outlet operations (accept orders, verify payments, adjust inventory), view or manage operational risk alerts, access outlets outside their assignment, or override Super Admin controls.

---

## P-09 Finance Officer

A financial operations role with ledger and settlement visibility.

**Context**: Reviews financial ledger entries, reconciles internal settlements between outlets, acknowledges settlement records, and reviews daily closing summaries plus expense reporting and policy outcomes. Works across all outlets.

**Cannot**: Mutate order, delivery, or inventory state. Cannot approve their own submitted records. Cannot manage catalog or pricing.

---

## P-10 Super Admin

The global platform authority. Exactly one or more individuals with this designation; access is tightly controlled.

**Context**: Full operational and configuration authority across all outlets, business domains, and records. Approves above-guardrail price changes, above-threshold inventory adjustments, forced financial closures, break-glass authorization policy changes, and other actions that no other persona can execute. Sensitive Super Admin mutations and overrides require explicit reason codes and are permanently audit-logged.

**Cannot**: Issue unaudited mutations. Super Admin authority does not bypass audit requirements — it extends them.

---

# 3. Access & Authorization Matrix

Rows are capability domains. Columns are personas. Notation:

- **Full** — full read + write within scope
- **Scoped** — access limited to assigned outlet(s)
- **Read** — read-only
- **Own** — access to own records only
- **Request** — can initiate but not approve
- **Approve** — can approve requests from others; cannot approve own
- **Acknowledge** — can record review/acknowledgement of settlement or closing records without voiding them
- **Offset** — can offset internal settlement records within authority
- **Threshold** — authority limited by a configured financial/quantity threshold
- **–** — no access

| Capability Domain | P-01 Customer | P-02 Agent | P-03 Cashier | P-04 Inv Clerk | P-05 Dispatcher | P-06 Outlet Mgr | P-07 Support | P-08 Area Mgr | P-09 Finance | P-10 Super Admin |
|---|---|---|---|---|---|---|---|---|---|---|
| **Own account & address** | Full | – | – | – | – | – | – | – | – | Full |
| **Address coordinate correction** | Own (map pin) | – | – | – | – | – | – | – | – | Full |
| **Product catalog browsing** | Read | – | Read | – | – | Read | Read | Read | – | Full |
| **Cart & order placement** | Full | – | – | – | – | – | – | – | – | Full |
| **Order status tracking** | Own | Own (assigned) | Scoped | – | Scoped | Scoped | Scoped | Read (assigned outlets) | Read | Full |
| **Order acceptance / rejection** | – | – | – | – | – | Scoped (with explicit permission) | – | – | – | Full |
| **Outlet picking** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **Outlet handover confirmation** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **Failed-order return receipt** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **Run-level returned-cylinder receipt** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **POS / walk-in sales** | – | – | Scoped (with explicit permission) | – | – | Scoped (with explicit permission) | – | – | – | Full |
| **Mobile money verification** | – | – | Scoped (with explicit permission) | – | – | Scoped (with explicit permission) | – | – | – | Full |
| **Payment account administration** | – | – | – | – | – | – | – | – | – | Full |
| **Payment reference submission** | Own | – | – | – | – | – | – | – | – | Full |
| **Delivery assignment** | – | – | – | – | Scoped | Scoped | – | – | – | Full |
| **Delivery batch management** | – | – | – | – | Scoped | Scoped | – | – | – | Full |
| **Delivery execution (pickup, PIN, COD)** | – | Own | – | – | – | – | – | – | – | Full |
| **Delivery evidence safety review** | – | – | – | – | – | – | – | – | – | Full |
| **Agent cash handover** | – | Own | – | – | – | Scoped (receive) | – | – | – | Full |
| **Delivery PIN fallback (regenerate)** | – | – | Scoped (with explicit permission) | – | – | Scoped (with explicit permission) | Scoped (with explicit permission) | – | – | Full |
| **Inventory viewing** | – | – | – | Scoped | – | Scoped | Scoped | Read (assigned outlets) | – | Full |
| **Inventory adjustments (submit)** | – | – | – | Scoped (Request) | – | Scoped (Policy-limited post; above = Request) | – | – | – | Full |
| **Inventory adjustments (approve)** | – | – | – | – | – | – | – | – | – | Full |
| **Outlet-to-outlet transfers** | – | – | – | Scoped (Request) | – | Scoped (Request/approve/receive) | – | – | – | Full |
| **Pick reversal confirmation** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **Returned cylinder intake** | – | – | – | Scoped | – | Scoped | – | – | – | Full |
| **Post-delivery return intake** | – | – | – | Scoped (with explicit permission) | – | Scoped (with explicit permission) | – | – | – | Full |
| **Vendor refill batch management** | – | – | – | Scoped | – | Scoped | – | – | – | Full |
| **Outlet configuration & policies** | – | – | – | – | – | Read | – | Read (assigned outlets) | – | Full |
| **Outlet price rules (within guardrail)** | – | – | – | – | – | Scoped | – | – | – | Full |
| **Outlet price rules (above guardrail)** | – | – | – | – | – | Request | – | – | – | Approve / Full |
| **Global pricing & catalog** | – | – | – | – | – | – | – | – | – | Full |
| **Refund initiation** | Own (request) | – | – | – | – | Scoped (Threshold) | Request | – | – | Full |
| **Refund approval** | – | – | – | – | – | Scoped (Threshold) | – | – | – | Full |
| **Refund payout (cash at outlet)** | – | – | Scoped (with explicit permission; payout limits) | – | – | Scoped (within payout limits) | – | – | – | Full |
| **Refund collection code management** | – | – | – | – | – | – | Scoped (with explicit permission) | – | – | Full |
| **Daily cash closing** | – | – | – | – | – | Scoped | – | – | – | Full |
| **Financial ledger (view)** | – | – | – | – | – | Scoped | – | Read (assigned outlets) | Full | Full |
| **Internal settlement management** | – | – | – | – | – | – | – | – | Acknowledge / Offset | Full |
| **Expense submission** | – | – | – | – | – | Scoped | – | – | – | Full |
| **Expense approval** | – | – | – | – | – | Scoped (Threshold) | – | – | – | Full |
| **Support case creation** | – | – | – | – | – | Scoped | Scoped | – | – | Full |
| **Support case management** | – | – | – | – | – | Scoped | Scoped | – | – | Full |
| **Support queue configuration** | – | – | – | – | – | – | – | – | – | Full |
| **Notification template administration** | – | – | – | – | – | – | – | – | – | Full |
| **Support communication logging** | – | – | – | – | – | Scoped | Scoped | – | – | Full |
| **Customer notification requests** | – | – | – | – | – | Scoped (approved transactional only) | Scoped (approved transactional only) | – | – | Full |
| **Support action requests (create/request)** | – | – | – | – | – | Scoped | Scoped | – | – | Full |
| **Support action requests (execute)** | – | – | – | – | – | Scoped | – | – | – | Full |
| **Audit log viewing** | – | – | – | – | – | Scoped | – | Read (assigned outlets) | Read | Full |
| **Operational risk alerts** | – | – | – | – | – | Scoped (with explicit permission) | – | – | – | Full |
| **Low-stock alerts** | – | – | – | – | – | Scoped | – | Read (assigned outlets) | – | Full |
| **Order reassignment (escalated)** | – | – | – | – | – | Scoped (exception) | – | – | – | Full |
| **Forced financial closure** | – | – | – | – | – | – | – | – | – | Full |
| **Cross-outlet reporting** | – | – | – | – | – | – | – | Read (assigned outlets) | Read | Full |
| **User & role management** | – | – | – | – | – | – | – | – | – | Full |
| **Authorization policy management** | – | – | – | – | – | – | – | – | – | Full (with dual approval for sensitive changes) |

**Scope note**: "Scoped" means the persona's assigned outlet(s) only. A persona assigned to Outlet A has no visibility into Outlet B unless explicitly granted additional access by Super Admin. Area Managers have read access to their assigned outlet set, not all outlets. Customer Support Agents are outlet-scoped by default; cross-outlet access requires explicit Super Admin grant.

**Support action request boundary**: Creating or requesting support action requests belongs to scoped support case management. The matrix's "execute" row means the authorized owner of the affected business workflow may accept or complete the requested action; it does not let Customer Support Agents directly approve refunds, pay refunds, post ledger entries, mutate orders, adjust inventory, or complete delivery workflows.

**Inventory adjustment approval threshold**: At launch, inventory adjustment authority is policy-driven. Outlet Managers may post without separate Super Admin approval only single-unit damage or quarantine adjustments that do not increase available stock and carry a source reference (for example, a count, delivery, or order). This single-unit rule is the launch default Outlet Manager posting threshold; no higher Outlet Manager quantity threshold exists at launch. Every loss, missing-source adjustment, manual correction, positive available-stock increase, count variance, or absolute delta greater than one unit is `PENDING_APPROVAL` and requires Super Admin approval before the ledger movement is posted. Every adjustment requires reason, note, active policy code, ledger correlation when posted, and audit trail.

**Inventory reconciliation paths**: Inventory reconciliation supports both quick day-to-day adjustments with reason codes and periodic physical counts with variance reports. Both paths create audited inventory adjustments and follow the active approval threshold before stock is recognized as changed.

**Refund payout — Cashier permission**: An Outlet Cashier (P-03) with explicit refund-payout permission may verify the customer's collection code and disburse cash only within the active refund payout limits. Launch default cashier payout limits are UGX 100,000 per refund and UGX 300,000 per outlet business day. Launch default Outlet Manager payout limits are UGX 500,000 per refund and UGX 1,500,000 per outlet business day. A payout above the Outlet Manager limit requires Super Admin release approval before cash is disbursed. Without that explicit per-outlet permission, cashiers cannot handle refund payouts.

**Refund approval thresholds**: Refund liabilities below the approval-required threshold become collectible without separate approval. At launch, the approval-required threshold is UGX 50,000: refund liabilities below UGX 50,000 become collectible without separate approval, and refund liabilities at or above UGX 50,000 require approval before a collection code is issued. The active refund approval policy starts from global defaults and may define outlet/refund-reason overrides. At launch, Outlet Managers may approve refund liabilities from UGX 50,000 through UGX 500,000 within their outlet scope; above UGX 500,000, Super Admin approval is required. Approval makes the liability collectible; payout is the separate cash-disbursement event.

**Refund collection code management**: This permission covers regenerating expired codes, regenerating codes when the customer loses access after verification through a linked support case with reason and audit, unlocking or regenerating rate-limited codes, and audited reveal by a permissioned Customer Support Agent or Super Admin after customer verification. It does not allow approving, paying, voiding, or writing off a refund liability.

**Internal settlement authority**: A Finance Officer (P-09) may acknowledge, query, and offset internal settlements. Voiding a settlement requires Super Admin approval with an explicit reason and audit record. Finance Officer cannot void unilaterally.

**Outlet configuration vs price rules**: "Outlet configuration & policies" covers settings that only Super Admin may change: service zone, vendor acceptance list, delivery mode support, operating-hours policy, payment method support, workload/capacity limits, max batch size, and outlet priority score. "Outlet price rules (within guardrail)" is a distinct sub-domain covering outlet-scoped product/refill/accessory prices, delivery-fee overrides, and express-fee multipliers. Outlet Managers have write access to price rules within configured guardrails; they do not have write access to broader outlet configuration.

**Payment account administration**: Creating, changing, deactivating, or setting the default mobile-money merchant account requires explicit payment-account administration permission. Writes require an audit reason. Ordinary reads expose only masked merchant-account details. Outlet/default constraints remain per outlet/provider, and account changes do not alter already-recorded payment attribution.

**Order reassignment (escalated)**: Outlet Managers may handle post-picking reassignment exceptions only for orders in their outlet scope. Super Admin retains full authority for cross-outlet, exhausted-candidate, and global exception paths.

---

# 4. Authentication & Authorization Model

Authentication establishes the human user's identity, but it does not determine outlet scope or business authority.

At launch:
- All human users except the root/bootstrap Super Admin enter the platform by phone-number signup. Phone control is verified through SMS OTP, the Identity account is created or reused, and the customer account/profile is ensured before customer features are available.
- Customer-only accounts authenticate with SMS OTP. Email OTP is not active at launch, and phone control is verified at registration/login.
- Privileged accounts (staff, delivery, support, finance, managers, and Super Admins) authenticate with username/password plus MFA. When any privileged grant or privileged scope is active or pending activation, customer SMS OTP is no longer an allowed login method for that account. The user must complete the required password and MFA setup or recovery path before authenticating.
- Privileged authentication does not remove customer access. After a privileged account successfully authenticates with username/password plus MFA, the authenticated context may include customer capabilities as well as any privileged capabilities allowed by explicit permissions, outlet scopes, policy, and session assurance.
- Any user may submit a generic unauthenticated recovery request for "I need recovery" using their login alias and safe contact context. That request does not prove identity, disclose account existence or status, authenticate the user, or change credentials. Privileged account recovery, lost phone, password reset, and MFA reset remain Super Admin-mediated only. At launch, Super Admin recovery means recovery of an account with Super Admin authority and requires break-glass dual approval by two distinct active Super Admins: one initiating or performing the recovery and a second non-requesting Super Admin co-approver. The recovered account subject cannot approve the recovery. No single-actor Super Admin recovery path exists at launch. The recovery record must identify the subject account, affected credential or recovery factor, initiating actor, co-approving actor, reason code, note, timestamp, before/after account-status and credential-readiness values, and confirmation that no secrets or payment credentials are stored. Normal checkout and late-payment-reference reuse do not require an extra OTP beyond the authenticated customer session.
- Credential readiness is a business access gate. Accounts requiring password setup, MFA setup, or recovery cannot exercise normal business permissions until the required step is completed; disabled accounts cannot authenticate or access any experience. If account-status or credential-readiness facts conflict, access fails closed until corrected through an audited path.

Authorization is determined by explicit permissions and scope assignments managed by the platform. A single human account may hold multiple permission bundles across one or more outlets. Login itself does not let the user choose a persona or experience as a source of authority. Experiences such as customer, delivery, staff, or admin are derived from those permissions; they are not separate account types. After authentication, the available experiences are limited to the account's permissions and current session assurance, and selecting an experience never grants new permissions, outlet scope, or business authority.

Outlet rosters, work queues, case queues, and other operational projections may organize work or explain visibility, but they are not authorization sources and cannot grant permission or outlet scope beyond the explicit assignments.

Only the latest approved and valid authorization policy may grant business access. Draft, invalid, failed, or unapproved authorization-policy changes cannot expand access; when authoritative permission or scope facts conflict, access fails closed until corrected through an audited approval path.

Authorization denials and sensitive successful grants are audit-relevant and must identify the active authorization policy basis used for the decision. Ordinary successful reads are not individually audit-logged unless this document classifies the read as sensitive.

Launch-scope acceptance, dependency-gate completion, and launch approval are business-governance decisions outside the runtime persona model. The Product Manager is the sole formal approval authority for those decisions unless the Product Manager explicitly adds another approval authority to launch scope. Engineering, QA, support, operations, and platform personas may provide evidence, risk notes, recommendations, and estimates, but those inputs do not create independent launch-blocking authority.

Operational-readiness recommendations do not create separate formal launch blockers unless the Product Manager accepts them into launch scope.

Launch scope remains open until Product Manager launch approval. When the Product Manager accepts a new launch-scope requirement, the acceptance record must identify the affected business workflows, any audit or reporting behavior affected, evidence reviewed, accepted residual risk, and launch timing impact.

Hands-on trials using real business operations are not allowed until the complete launch system is ready for production use. No launch-scope workflow is considered operationally usable in isolation before the complete launch system is production-ready. Scripted non-production sign-off may occur earlier, but it does not authorize real operational use.

---

# 5. Business Invariants

Numbered rules that must hold unconditionally. These are not implementation suggestions — they are correctness requirements. Any system state that violates them is a bug.

---

**BI-01 — Available stock never goes below zero.**  
Stock committed to orders cannot exceed physical available stock. When an order claims stock, the claim is atomic relative to other concurrent claims. A claim that cannot be satisfied is rejected.

**BI-02 — Price snapshot is write-once.**  
When an order is placed, the prices in effect at that moment are captured and frozen on the order. Subsequent changes to pricing rules do not alter the prices on any historical order. All financial calculations downstream (receipts, refunds, settlements) derive from the frozen snapshot, not from current rules.

**BI-03 — Payment references are unique per provider.**  
A mobile-money transaction reference cannot be applied to more than one active or completed payment per provider. The same reference string from different providers is not a duplicate. The second attempt to use the same reference at the same provider is rejected regardless of which customer, outlet, or order is involved, except for the explicitly handled same-customer late-reference reuse workflow in §7.10 where the original cancelled order remains cancelled and the reference is re-verified against the recreated order.

**BI-04 — Financial record entries are write-only.**  
Entries in any financial or inventory ledger are never modified or deleted after creation. Errors are corrected by appending compensating entries with explicit linkage to the erroneous entry. The audit trail is always complete and continuous.

**BI-05 — Financial closure is terminal.**  
Once an order reaches financial closure, its original sale financial state is sealed. The receipt issued at closure is immutable and is not rewritten by later events. Post-closure financial activity is allowed only as a separate linked business record, such as an approved post-delivery return/refund liability or an approved adjustment/void record; it must not alter the original closure or receipt.

**BI-06 — Reserved stock is unavailable.**  
Stock reserved for any active order — whether online or walk-in — cannot be allocated to any other order. This applies equally to every sales channel sharing the same physical inventory.

**BI-07 — COD orders never require payment verification before fulfillment begins.**  
For cash-on-delivery orders, the outlet may begin acceptance, picking, batching, dispatch, and delivery-agent assignment without any payment action. Payment is collected at the doorstep by the agent.

**BI-08 — Delivery confirmation is all-or-nothing.**  
Customer confirmation of delivery (via PIN) is the single commit point for: delivery status, order status, outgoing stock commitment, returned-cylinder field recording, cash collection recording, and payment status. These succeed together or none do.

**BI-09 — Fulfilled orders have no partial completion state.**  
Either all items in an order are successfully delivered, or the whole order fails. The only exception is an approved full-order adjustment that resolves the delivery issue, such as an Outlet Manager-approved refill-to-new-cylinder conversion at the doorstep; it is still a whole-order mutation, not selective line-level completion.

**BI-10 — Incoming and outgoing cylinder sizes match on refill exchange.**  
A refill order exchanges a customer's empty cylinder for a filled cylinder of the same size. A customer who brings a 6kg empty cannot receive a 12kg filled, and vice versa. A size upgrade or downgrade is a new cylinder purchase, not a refill. A returned cylinder whose size differs from the expected size is a size-mismatch exception: the actual size must be recorded, and the original refill may continue only through an approved full-order adjustment that keeps the final incoming and outgoing sizes matched and handles any COD/refund delta before PIN confirmation; otherwise the order fails or converts under the new-cylinder path.

**BI-11 — Outlet stock is isolated.**  
Stock at Outlet A cannot be drawn down by an order assigned to Outlet B. Stock transfers between outlets follow an explicit, audited transfer workflow. No order can implicitly access another outlet's inventory.

**BI-12 — A refund liability persists until discharged.**  
A refund owed to a customer (from overpayment, failed delivery, pre-delivery cancellation of paid orders, reassignment-driven prepaid overage, prepaid doorstep price-recalculation overage, or post-delivery return) remains an open liability on the outlet's financial record until a cash payout event is recorded or a Super Admin-approved void/write-off path resolves it with reason, note, and audit trail. Code expiry, daily closing, and time passing do not discharge the liability, and an open liability does not convert to revenue. Open refund liabilities appear in the owning outlet's daily closing and liability reports until paid, voided, or written off.

**BI-13 — Internal settlements are accounting entries only.**  
When one outlet fulfills an order that another outlet was paid for, the resulting settlement between those outlets is an internal ledger allocation. It does not represent a cash transfer between outlets. The settlement balance remains open until a Finance Officer or Super Admin acknowledges or offsets it, or a Super Admin approves voiding it, through an audited process. Settlement acknowledgement is a finance review marker, not a customer-payment event or revenue-recognition gate. Settlement offset means the balance is netted against another internal settlement balance; it is not a customer payment or physical cash movement. Outlet performance reports recognize revenue under the final fulfilling outlet at financial closure and show settlement status (`POSTED`, `ACKNOWLEDGED`, `OFFSET`, or `VOIDED`) alongside the revenue view.

**BI-14 — Receipt numbers are permanent and sequential.**  
Receipt numbers are outlet-scoped and sequential within the outlet/date receipt series, carry the outlet and receipt date in the business number, and are issued at financial closure. Receipt numbers are distinct from order identifiers; an order identifier is not the financial receipt number. A number issued to a valid receipt cannot be reissued or modified. Ordinary financial-closure rollback does not create a numbering gap. Any skipped number range caused by an intentional exception (for example, voided, superseded, imported, or operator-repaired numbers) requires a permanent approved exception and audit record explaining the gap.

**BI-15 — Audit records are permanent.**  
Audit log entries are never altered or deleted, regardless of data retention policies applied to the subject records. If a subject entity is archived or purged, its audit history remains. Sensitive administrative changes must record the actor, timestamp, reason where required, and before/after business values, but must not store secrets or payment credentials.

**BI-16 — A persona cannot approve their own submission.**  
Any action requiring approval or co-approval cannot be approved by the same individual who submitted or requested it, regardless of their role. This includes expense above threshold, inventory adjustment, refund above threshold, price change above guardrail, material forced financial closure, and sensitive authorization-policy change.

**BI-17 — Outlet policy is a prerequisite for order acceptance, not a post-hoc check.**  
Before an order is assigned to an outlet, the outlet's policies (payment method support, delivery mode support, refill vendor acceptance, same-vendor refill capability) are evaluated. An order cannot be assigned to an outlet that does not meet its requirements.

**BI-18 — A refund collection code is single-use and perishable.**  
The code issued to a customer to collect a cash refund at an outlet has a finite validity window; at launch, that window is 24 hours after issuance. The code is invalidated upon first successful presentation. Failed verification attempts are rate-limited per refund and outlet actor and audit-logged with actor, outlet, refund ID, timestamp, and safe failure reason. After a configured refund-code attempt threshold, the code is temporarily locked and requires regeneration or unlock by a permissioned Customer Support Agent or Super Admin with reason and audit trail. The refund liability remains outstanding while the code is locked, and no payout may be recorded until the code is unlocked or regenerated.

**BI-19 — No order can complete financially until all expected returned cylinders are accounted for.**  
For refill orders, financial closure requires every expected returned cylinder to be accounted for through returned-cylinder intake confirmation or an approved exception when the cylinder is missing, rejected, or otherwise fails intake. Delivery can be confirmed by the customer before intake is complete, but financial closure waits.

**BI-20 — Order totals are immutable after placement.**  
No business event — price change, outlet reassignment, bundle-discount update, or pricing-rule update — can alter the total of a placed order except through an explicit, audited adjustment workflow that generates its own financial record. Delivery-fee changes and other post-placement deltas must be represented as explicit adjustments. A customer refund created by a lower final amount remains a separate refund liability; it does not rewrite the placed order's price snapshot or original charge.

**BI-21 — Every cancellation must be fully attributed.**  
Regardless of who or what initiates a cancellation — customer, outlet staff, policy timeout, or Super Admin — the cancellation record must capture: the actor identity, the actor's role, a reason code, a reason note, and a timestamp. A cancellation with any of these fields absent is a data integrity violation.

**BI-22 — Outlet disable is blocked by active orders.**  
An outlet cannot be set to inactive or disabled while it has any orders in a non-terminal state. For this rule, terminal states are `COMPLETED`, `CANCELLED`, `FAILED` for COD orders with no open refund liability, and `REFUNDED`. `CANCELLED_PENDING_REFUND` is non-terminal — orders in this state also block outlet disable because the financial liability is still open. Disablement is allowed only after every such order is completed, cancelled, failed as a COD order with no open refund liability, refunded, or reassigned.

**BI-23 — Critical mutations are idempotent.**
Order creation, cancellation, payment reference submission or reuse, payment verification, refunds, delivery completion, inventory reservation/adjustment/transfer, cash reconciliation, receipt generation, and comparable one-time state transitions require idempotency keyed by actor, endpoint, and client key. Identical replays return the original result; same-key/different-body replays are rejected; in-progress duplicates return a retryable response. Domain uniqueness rules remain permanent even after operational replay records expire.

**BI-24 — New cylinder purchases are outright sales.**
A customer who buys a new filled cylinder owns the cylinder and gas outright. There is no deposit, cylinder-return obligation, customer credit account, or customer-level cylinder ownership tracking at launch. New filled cylinder purchases may use any supported vendor available at the assigned outlet; a customer who loses or damages their cylinder must buy a new cylinder rather than treat the loss as a refill.

**BI-25 — Launch currency is UGX only.**
All launch prices, cash due, refunds, settlements, expenses, receipts, and financial reports are denominated in UGX. Multi-currency sale, refund, settlement, or expense workflows are not supported at launch; any future currency expansion requires an explicit business policy decision.

**BI-26 — Launch commercial programs are limited to explicit pricing rules.**
Customer loyalty points, subscriptions, and advanced-promotion workflows are not supported at launch unless explicitly added to launch scope. Customer discounts and price reductions come only from active price rules, bundle component pricing, approved adjustments, or other rules named in this document.

---

# 6. Domain Lifecycles

## 6.1 Order Lifecycle

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

**State rules:**
- Customer can cancel up to and including `PICKING`. `READY_FOR_DISPATCH` cancellation is a pre-custody exception for an explicitly permissioned in-scope Customer Support Agent or Super Admin and requires explicit override acknowledgement, reason, note, audit, and pick-reversal handling where stock has already been picked. Once delivery-run custody or `OUT_FOR_DELIVERY` begins, the normal cancellation route is closed; failed-delivery, return/refund, forced-closure, or audited financial-adjustment workflows carry the outcome.
- Outlet staff without explicit cancellation permission cannot use normal cancellation after `PICKING`; they use cannot-fulfill, pick-reversal, failed-delivery, or Customer Support Agent escalation workflows according to fulfillment state.
- After placement, customer-requested changes to items, delivery address, or original pricing are not supported as order modifications. If the customer wants a different order before fulfillment, the existing order is cancelled where cancellation is still allowed and the customer places a new order. Approved delivery-conversion, mismatch, refund, or reassignment-delta paths are exception adjustments, not edits to the original order request.
- All orders (mobile money and COD) receive an outlet assignment and reserve inventory at placement. For mobile-money orders, the status does not transition to `ASSIGNED_TO_OUTLET` until the payment reaches `STAFF_VERIFIED` and the payment gate permits fulfillment; before that point, the assigned outlet may see the order for payment-instruction and payment-verification purposes only, not for acceptance, picking, batching, dispatch, or delivery-agent assignment. By contrast, COD orders transition to `ASSIGNED_TO_OUTLET` immediately after placement.
- Outlet acceptance follows the active outlet acceptance policy. By default at launch, an outlet actor with explicit order-acceptance permission must manually accept or reject the order. As an alternative, the system may be configured to automatically accept eligible assigned orders without manual outlet action. If the outlet rejects the order or the acceptance window expires, the order enters the cascade/reassignment path before picking begins.
- For orders accepted under the no-manual-action policy, failure to start picking within the active picking-start timeout enters the same pre-picking cascade/reassignment path.
- `STAFF_VERIFIED` means an authorized outlet payment-verification actor has manually confirmed the payment reference against the provider for the assigned outlet. `PAID` means the payment is posted and confirmed. These are distinct: an underpayment holds the payment state at `PARTIALLY_PAID` (not yet `PAID`) while a COD top-up is arranged.
- Operational progress (acceptance, picking, batching, dispatch, and delivery-agent assignment) is blocked for mobile-money orders until `STAFF_VERIFIED` and any required payment-gate resolution, such as Outlet Manager approval of a COD top-up for an underpayment.
- `AWAITING_OUTLET_ACCEPTANCE` is the shared entry state for all order paths. Mobile-money orders enter after `STAFF_VERIFIED`, `ASSIGNED_TO_OUTLET`, and any required payment-gate resolution; COD orders enter directly from `ASSIGNED_TO_OUTLET`.
- If an accepted outlet marks an order cannot-fulfill before `PICKING`, the order re-enters the cascade/reassignment path under §7.6. After `PICKING`, cannot-fulfill is a post-picking exception and must use pick reversal, cancellation/refund, failed-delivery, Customer Support Agent routing, or Super Admin handling according to custody state.
- `DELIVERED` is customer-facing completion. `COMPLETED` is internal, signifying financial closure. Customers do not see `COMPLETED`. A delivery confirmation is issued to the customer immediately when the order reaches `DELIVERED`. The financial receipt (with full cost breakdown) is generated and issued only when the order reaches `COMPLETED`. If financial closure is delayed by unresolved custody or inventory exceptions, the customer receives the delivery confirmation first and the receipt follows when closure is achieved.
- For online deliveries, `COMPLETED` requires delivery-run custody closure, successful financial posting, and no unresolved closure blockers. Refill orders additionally require returned-cylinder intake confirmation for every expected cylinder or an approved exception. Non-refill orders may move from `DELIVERED` to `COMPLETED` as soon as those closure facts are present.
- Normal financial closure occurs from recorded business facts; customers and outlet staff do not manually mark online orders `COMPLETED`. Super Admin forced closure remains a separate audited exception workflow before closure effects are posted.
- An order in `DELIVERED` but not yet `COMPLETED` appears in a derived internal pending-closure view when unresolved closure blockers remain; this does not add a separate order lifecycle state and is not customer-facing. **SLA**: the default target is resolution within the same business day. A hard operational alert is raised if the order remains unclosed after 24 hours. Super Admin escalation is triggered at 48 hours. If blockers remain unresolved after escalation, Super Admin may use forced financial closure only through the exception path in F-05.
- Each closure blocker identifies the blocker type, owning outlet or workflow, related business record, SLA deadline, current status, and any resolution or waiver audit facts; these blockers explain the derived pending-closure view without becoming an order status.
- The delivery PIN is a six-digit one-time code generated when the order reaches `READY_FOR_DISPATCH`, not at placement. The customer is notified at that point. The short exposure window reduces the risk of PIN leakage before the delivery window opens.
- For COD orders, `FAILED` is a terminal state. For mobile-money prepaid orders, `CANCELLED_PENDING_REFUND` is the required next state after `FAILED` — the order is financially open until the full prepaid amount is refunded in cash. The delivery fee is waived (§7.12). `REFUNDED` is the true terminal for failed prepaid deliveries.
- When a paid mobile-money order is cancelled before dispatch, the order moves to `CANCELLED_PENDING_REFUND` — operationally cancelled but not financially closed. A refund liability is created immediately. The order reaches `REFUNDED` (the true terminal) only after the cash payout is recorded and posted. Unpaid mobile-money orders cancelled before dispatch (no payment reference submitted) and COD orders cancelled at any point move directly to `CANCELLED`.

---

## 6.2 Payment Lifecycle

```
PENDING
  ├─► [Mobile Money]
  │     AWAITING_CUSTOMER_PAYMENT
  │       └─► AWAITING_STAFF_VERIFICATION
  │             └─► STAFF_VERIFIED
  │                   ├─► PAID (verified amount = order total)
  │                   ├─► PARTIALLY_PAID (verified amount < order total; fulfillment blocked until Outlet Manager approves COD top-up)
  │                   │     └─► PAID (approved COD delta is collected at doorstep during delivery confirmation)
  │                   ├─► OVERPAID (verified amount > order total; refund liability opened for excess; fulfillment proceeds after liability handoff)
  │                   │     └─► PAID (order payment gate satisfied; refund lifecycle remains open until payout)
  │             └─► FAILED (bad reference / permanently rejected)
  │
  └─► [COD]
        PENDING ──► COLLECTED (PIN confirmation commits agent-recorded cash at door)

  PAID            (terminal for mobile money)
  COLLECTED       (terminal for COD)
  PARTIALLY_PAID  (underpayment verified; fulfillment blocked until an Outlet Manager approves a COD top-up delta; after approval, fulfillment may proceed and the state resolves to PAID after doorstep collection)
  OVERPAID        (overpayment verified; excess becomes a refund liability before fulfillment proceeds; resolves to PAID for the order payment gate while the refund lifecycle remains open until payout)
  FAILED          (terminal — bad reference or permanently rejected verification)
  CANCELLED       (order cancelled before payment)
  REFUNDED        (terminal — after successful cash refund payout)
```

**Rules:**
- A payment reference is unique per provider. The same reference cannot be applied to two orders at the same provider, but the same reference string from different providers is not a duplicate.
- Customer-facing mobile-money payment instructions are shown only after outlet allocation and use the assigned outlet's configured merchant account by default. The assigned payment account remains part of the payment record; a global company account is allowed only as an explicit fallback configuration.
- A reference submitted after order cancellation is marked `LATE_REFERENCE_AFTER_CANCELLATION`. The same customer may reuse it on a new order within 7 days, including a recreated order with different items, subject to normal checkout validation. After 7 days, mediation by an explicitly permissioned Outlet Cashier or Outlet Manager within scope, or by a Super Admin, is required.
- Payment verification by an authorized outlet payment-verification actor records verifier identity, role, timestamp, provider, transaction reference, verified amount, and decision.
- Payment verification is independent from downstream cash and financial reconciliation. Unresolved agent cash handover, outlet settlement, refund payout, or daily closing does not by itself block recording a valid payment-verification decision.
- For failed COD deliveries, no customer cash is collected, but the order keeps a zero-collection payment fact tied to the failed-delivery reason. That fact is not a payable amount and does not create a refund liability.
- Payer phone entry is not required at launch, and payer phone is not a required verification input matched against the customer/order phone. Payment verification relies on the provider, transaction reference, verified amount, and verification decision, while the verifier manually compares the provider-statement phone to the customer/order phone. A phone mismatch is not a clean verification fact and cannot be cleared by payment verification alone; the Outlet Manager must resolve it operationally outside the payment-verification workflow.
- `STAFF_VERIFIED` is the first operational gate for mobile-money orders. Once an authorized outlet payment-verification actor verifies the provider reference, the payment outcome branches to `PAID`, `PARTIALLY_PAID`, or `OVERPAID` based on the verified amount. Outlet acceptance, picking, batching, dispatch, and delivery-agent assignment remain blocked until `STAFF_VERIFIED`; for `PARTIALLY_PAID`, they remain blocked until the COD top-up delta is Outlet Manager-approved.
- A verified underpayment cannot silently proceed, be waived, or be written off at launch. It clears only through an Outlet Manager-approved COD top-up for the full shortfall or a Super Admin-approved exception adjustment that records an explicit adjustment path; Customer Support Agents may route the review but cannot clear payment directly.
- A verified overpayment clears the order payment gate only after the excess is handed off as a cash refund liability. The excess does not become customer credit, a wallet balance, or a payment-issued refund.
- Payment discrepancy resolution is a durable business workflow for top-up approval, COD short/over collection exceptions, and refund-liability handoff. Customer Support Agents may route review through support, but they cannot mutate payment state; high-risk approvals and overrides require audit.
- When a late reference (from a cancelled order) is reused on a new order within 7 days (§7.10), it enters the payment lifecycle at `AWAITING_STAFF_VERIFICATION` — the reference already exists and requires re-verification against the new order, not a new customer payment. The lifecycle then proceeds normally (AWAITING_STAFF_VERIFICATION → STAFF_VERIFIED, etc.); the `LATE_REFERENCE_AFTER_CANCELLATION` flag on the original order is a historical marker only.

---

## 6.3 Delivery Lifecycle

```
PENDING
  ├─► CANCELLED (terminal before pickup/custody)
  └─► ASSIGNED (delivery agent assigned)
        ├─► CANCELLED (terminal before pickup/custody)
        └─► PICKED_UP (outlet handover and agent receipt confirmed)
              └─► OUT_FOR_DELIVERY
                    └─► ARRIVED (agent at customer location)
                          ├─► DELIVERED (PIN confirmed; delivery leg terminal; order may still await financial closure)
                          └─► FAILED
                                └─► RETURNED_TO_OUTLET (terminal)
```

**Rules:**
- Express deliveries do not use delivery windows. An express order is dispatched as soon as possible within outlet operating hours after the outlet accepts, picking is complete, and an eligible delivery agent is available — there is no scheduled window and no window-selection step for the customer. For batched deliveries, the customer selects a window at checkout from the outlet's active delivery-window policy within the active maximum advance-booking window; if the outlet has no override, the global launch fallback uses 2-hour delivery blocks. Future batched orders are allocated and inventory-reserved at placement, and the order is held until the selected window opens for dispatch.
- Delivery-agent assignment follows the active outlet assignment policy. At launch, an eligible agent must be an active delivery agent with delivery scope for the run outlet and no active picked-up run. Formal shift schedules, live location, and route-distance optimization do not add eligibility requirements, expand scope, or change assignment ranking at launch. The default assignment policy ranks eligible agents by queued assignment load, then least-recent assignment, then a deterministic tie-breaker. A Dispatcher or Outlet Manager may manually assign or override an assignment before pickup within their outlet scope with a recorded reason. Operational risk alerts may be shown to permissioned Outlet Managers or Super Admins during manual assignment, but they do not block assignment or change assignment priority.
- Batched delivery runs group eligible orders for the same outlet, service zone, and selected delivery window. They obey the outlet's active maximum batch size, using the global default when the outlet has no override. Orders beyond that limit overflow into additional runs for the same window. A Dispatcher or Outlet Manager may manually split, merge, or regroup batched runs within outlet scope, with audit, but manual adjustment does not waive pickup, custody, cash, or run-closure rules.
- A delivery agent may have multiple queued assignments but only one active delivery run in progress at a time. Batched delivery is the mechanism for multi-order runs; an express delivery is always a single-order run. The agent works through queued assignments sequentially.
- Delivery-agent field actions must be recorded live at launch: pickup, arrival, cash collection, returned-cylinder recording, PIN confirmation, failed delivery, and handover. If the agent cannot record the action live, the action has not completed for business purposes. Dead-zone handling is an operational field concern and does not create a deferred-completion business path.
- Every delivery run has a custody manifest. A batched run uses one batch-level custody handover with per-order item lines; an express run uses the same custody semantics for a one-order run.
- `CANCELLED` is a delivery-task terminal state only before pickup/custody, when the underlying order or delivery task is cancelled. After pickup, cancellation is no longer the delivery-task outcome; failed-delivery, return-to-outlet, and custody reconciliation rules apply.
- Transition to `PICKED_UP` requires two confirmations: (1) an authorized outlet handover actor records the handover of all outgoing items to the agent, and (2) the agent confirms receipt of those items. Both must be recorded before the delivery run departs. A discrepancy between the handover record and the agent's receipt confirmation is a custody exception requiring Outlet Manager resolution before departure. Missing agent receipt confirmation can be overridden only by an Outlet Manager or Super Admin with reason and audit.
- After `PICKED_UP`, outgoing stock remains in the agent's custody until delivery confirmation, failed-order return receipt at the outlet, or outlet reconciliation covers an approved exception.
- A Dispatcher or Outlet Manager may reassign the delivery agent within outlet scope until `PICKED_UP`, with a recorded reason. After pickup, reassignment requires Outlet Manager or Super Admin exception.
- A delivery run cannot close until all included deliveries have a terminal status or an approved Outlet Manager exception. Failed orders with physical goods must be returned to the outlet and receipt-confirmed by an authorized outlet return-receipt actor before run closure. For runs that include refill deliveries, closure additionally requires an authorized run-level returned-cylinder receipt actor to confirm that all customer-returned cylinders collected during the run have been physically received at the outlet. This cylinder receipt confirmation is a run-level gate, separate from the order-level intake and financial closure workflow.
- For refill deliveries, returned-cylinder collection is captured before PIN confirmation and committed at `DELIVERED`; returned cylinders do not become outlet inventory until outlet intake confirmation (a separate workflow).
- PIN confirmation is the customer's final acceptance of the delivered order and amount due. For refill deliveries, the agent must record every expected returned cylinder's vendor, size, condition, any approved conversion, COD delta or cash collection, and delivery exception before requesting PIN confirmation.
- If a returned cylinder is found damaged or otherwise unacceptable for refill at the doorstep: the agent's recorded condition decision controls the delivery outcome; there is no separate doorstep dispute flow at launch. The agent offers the customer a full-order conversion. If refused, the order status becomes `FAILED`. If accepted and Outlet Manager approval is recorded, the conversion is recorded as an approved full-order adjustment while the original refill line and price snapshot remain immutable, and the COD delta is collected. (See F-03 in §8.)
- If the returned cylinder at the doorstep is an unexpected vendor but the correct size and in acceptable condition: the agent accepts it. The actual incoming vendor becomes the exchange fact for intake and pricing; the refill price is recalculated using the actual same-size combination, and any delta is recorded as an explicit adjustment. If the recalculated price is higher than the original paid or COD amount due, the delta is collected as a COD top-up (Outlet Manager-approved before PIN confirmation); if lower, unpaid/COD orders reduce the amount due, while prepaid mobile-money orders create a cash refund liability at financial closure. The exchange proceeds; this path does not fail the order. (See F-07 in §8.)
- If the returned cylinder is acceptable but a different size from the expected refill size: the agent records the actual size and treats it as a size-mismatch exception. The original refill cannot complete silently; it may continue only through an Outlet Manager-approved full-order adjustment that keeps incoming and outgoing sizes matched and settles any COD/refund delta before PIN confirmation. If no such adjusted exchange is approved and accepted, the delivery fails or follows the new-cylinder conversion path.
- If an outgoing cylinder is found defective at the outlet before the agent departs: agent records "defective product," the unit is quarantined, and the outlet attempts an immediate replacement from available stock. For express orders, an available replacement proceeds immediately; for batched orders, the replacement proceeds only when it does not disrupt the route or delivery window. If replacement is unavailable, or if a batched replacement would be operationally disruptive, the order is moved to the next eligible dispatch opportunity or batch/window. An Outlet Manager or Super Admin may override the reschedule decision. (See F-06 in §8.)
- Customer-returned damaged or unacceptable cylinders and outlet-owned defective outgoing cylinders remain separate exception categories for reporting and review.
- Delivery agents may mark a delivery `FAILED` without Outlet Manager approval. A controlled reason code and audit trail are required; an optional note may be added. Failed delivery cancels the order at launch — the agent cannot reschedule. If the customer still wants the goods, they must place a new order. Agents return any undelivered physical goods to the outlet; goods are not re-dispatched.
- The same launch field-proof policy applies to express and batched deliveries. An authoritative timestamp is always recorded. GPS is required for arrival, failed delivery, doorstep defect, and terminal delivery attempts when the device can provide it; otherwise the agent must record a controlled location-unavailable reason and note, and the action may proceed unless the active manager/outlet override policy requires additional approval. Photos are required for damaged returned cylinders, unexpected-vendor or wrong-size returned cylinders when a physical cylinder is present, refused returned cylinders when a physical cylinder is present, and every defective outgoing cylinder report. A missing returned cylinder requires time/location facts and a reason note, but no photo unless another physical defect or custody exception is present. Failed-delivery reasons involving a physical defect or safety issue require photo evidence when safe to capture; customer-unavailable, refusal, PIN-failure, cash-mismatch, PIN-fallback, handover, and support-review paths require structured facts, reason notes, and audit rather than mandatory photos unless they also involve a physical defect or custody exception.
- Required or voluntarily supplied photos are linked to the relevant delivery exception. Completed delivery evidence linked to delivery proof history is retained for seven years unless a legal hold or approved legal/accounting retention policy changes the window. Pending evidence capture that never produces completed evidence expires after 30 days. Evidence files are unavailable through ordinary Customer Support Agent or outlet-operational access until safety review clears. Evidence that fails safety review is quarantined from those ordinary access paths and may be inspected only through Super Admin-level safety/audit access; the related delivery proof fact remains durable with a review marker for support and audit review.
- **PIN attempt limits**: PIN entry is rate-limited. Launch defaults: 5 invalid attempts within a 15-minute window triggers a 15-minute lockout. After 2 lockouts, or after 10 lifetime invalid attempts against the active PIN, the agent cannot retry without fallback by a permissioned Outlet Cashier, Outlet Manager, Customer Support Agent, or Super Admin. The fallback requires customer verification note, reason code, and Safety/Audit record, regenerates a new PIN (invalidating the old one), notifies the customer via push/email, and may reveal the replacement PIN once only to one of those fallback actors with explicit reveal permission after customer verification. Delivery agents cannot request fallback, perform fallback, or view/reveal PINs.

---

## 6.4 Inventory Reservation Lifecycle

```
(no reservation)
  └─► RESERVED — stock held for an active order
        ├─► COMMITTED — delivery confirmed; stock deducted
        ├─► RELEASED — order cancelled; stock returned to available
        └─► REASSIGNMENT_HOLD — transitional during outlet reassignment
              ├─► RESERVED (promoted on reassignment acceptance)
              └─► RELEASED (if outlet times out, customer rejects, or reassignment otherwise fails)

TRANSFER_HOLD — stock locked for an in-transit outlet-to-outlet transfer
  └─► (released when transfer received)

PICK_REVERSAL_PENDING — stock locked after a post-picking exception; picked goods await
                        authorized inventory/custody confirmation that they are physically back and sellable
  └─► RELEASED (authorized inventory/custody actor confirms physical return and sellable condition; stock returns to available)

PENDING_RELEASE — release entry failed to post; stock remains unavailable until corrected
```

**Rules:**
- Reserved and held stock is not visible as available to any new order or walk-in sale.
- Only available stock can be reserved.
- Reservation age alone does not release stock. Stock release occurs only through an explicit policy-defined lifecycle such as unpaid mobile-money cancellation, failed delivery return, order cancellation, or another audited release workflow.
- A reservation that cannot be committed (delivery failed) must be released, not left open.
- Incoming customer cylinders (from refill delivery) are tracked separately as `PENDING_INTAKE` and do not enter available empty stock until outlet intake is confirmed.
- `PICK_REVERSAL_PENDING` is entered when an order is pulled back after picking has begun (Outlet Manager or Super Admin exception workflow only). The picked stock remains unavailable to all new orders until an authorized inventory/custody actor explicitly confirms it is physically back at the outlet and sellable. If the item is damaged or missing, the normal damaged/lost stock workflow applies instead of release to available. A recovery outlet may proceed on its own reserved stock before the original reversal resolves if the exception is approved, but financial closure of the original order waits for the outcome.

---

## 6.5 Outlet Transfer Lifecycle

```
REQUESTED (by receiving outlet)
  └─► APPROVED (sending Outlet Manager approves; sender holds transfer stock outside sale availability)
        └─► IN_TRANSIT (stock has left sender availability and is not yet received)
              └─► RECEIVED (receiving outlet confirms receipt; stock added to receiver)

  REJECTED   (terminal — sender declines from `REQUESTED`; no inventory impact)
```

**Rules:**
- Only available filled cylinders, available accessories, and confirmed empty cylinders may be transferred. Reserved, damaged, quarantined, `IN_TRANSIT`, and `IN_REFILL` stock is not transferable.
- Super Admin can bypass the transfer approval step only through an audited override with reason, before/after stock impact, and affected outlets.
- Both sides must participate: the receiving outlet requests the transfer, the sending outlet approves and releases stock, and the receiving outlet confirms receipt.
- Every transfer state transition is audit-logged with the actor, outlet scope, timestamp, and reason where applicable.
- Inter-outlet stock transfers do not involve financial settlement. Only inventory movements are tracked. No financial settlement entry is created between outlets for a stock transfer. This distinguishes transfers from post-payment order reassignments, which do create an internal settlement at financial closure.

---

## 6.6 Refund Lifecycle

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

**Rules:**
- At launch, all customer refunds are cash-only regardless of original payment method. Provider-issued refunds, customer wallet credit, and mobile-money refund workflows are not supported.
- When the original customer payment was mobile money, the cash refund is treated as an outlet cash payout against that mobile-money-paid order for closing and reporting; it does not create a provider refund, customer credit, or internal transfer.
- When a paid mobile-money order is cancelled before dispatch, a refund liability is created immediately. The order moves to `CANCELLED_PENDING_REFUND` and the refund lifecycle begins from `LIABILITY_OPEN`. The order does not reach `REFUNDED` (its true terminal) until the cash payout is recorded and the liability is discharged.
- Refund collection codes expire after the active short operational window; at launch, this window is 24 hours after issuance. A permissioned Customer Support Agent or Super Admin can regenerate an expired code with audit, or regenerate a code when the customer loses access only after customer verification through a linked support case with reason and audit. The refund liability itself does not expire and carries forward regardless of code expiry.
- A refund liability does not expire. It remains open until paid, voided, or written off through a Super Admin-approved void/write-off path with reason, note, and audit trail.
- Only the original account holder may collect a refund at launch. Delegated or proxy collection is not a standard supported workflow. Any exceptional override requires a linked support case, Customer Support Agent routing or Super Admin handling under the active exception policy, explicit reason and evidence notes, and a full audit trail. Outlet Managers cannot independently approve proxy collection.
- At payout, the Outlet Manager or explicitly permissioned Outlet Cashier, acting within the active refund payout limits, must verify the one-time collection code and confirm that the collector is the original account holder tied to the order and that the customer/order phone matches before cash is marked paid. Launch default payout limits are UGX 100,000 per refund and UGX 300,000 per outlet business day for explicitly permissioned Outlet Cashiers, and UGX 500,000 per refund and UGX 1,500,000 per outlet business day for Outlet Managers; payout above the Outlet Manager limit requires Super Admin release approval before cash is disbursed. Every payout attempt must capture actor, role, outlet, amount, order/refund ID, reason, approval status, active threshold policy version, and timestamp; paid cash events must also capture the customer/order identity snapshot and attestation note.
- Delivery Agents do not issue cash refunds. When a delivery fails, they record the failure and return physical goods; any refund remains in the outlet cash-refund lifecycle.
- The collection code is revealed only through the authenticated customer experience or audited reveal by a permissioned Customer Support Agent or Super Admin after customer verification. It must not appear in push notifications, email bodies, or SMS. An Outlet Manager or explicitly permissioned Outlet Cashier may verify a customer-presented code, but they cannot reveal the active code to the customer.
- Daily closing may complete with open refund liabilities, but the Outlet Manager must explicitly acknowledge them and carry them forward.
- The outlet that received the original payment is responsible for disbursing the refund. Another outlet cannot pay the refund on the payment-receiving outlet's behalf.
- For post-delivery returns (§7.14), a refund liability is created only after outlet inspection and approval by the active return policy's approval role. Returns rejected on condition do not enter the refund lifecycle and do not create a liability.

---

## 6.7 Support Case Lifecycle

```
OPEN
  ├─► IN_PROGRESS (queue or individual owner investigating)
  │     ├─► WAITING_ON_CUSTOMER (support needs customer response)
  │     │     └─► IN_PROGRESS
  │     ├─► WAITING_ON_INTERNAL_TEAM (awaiting domain workflow outcome — refund, adjustment, etc.)
  │     │     └─► IN_PROGRESS (domain workflow outcome recorded)
  │     └─► READY_FOR_SUPPORT_REVIEW (all blocking action requests terminal; no customer response remains; support reviews outcome)
  │               └─► CLOSED (terminal — manual closure by a permissioned support-case actor, such as a Customer Support Agent, in-scope Outlet Manager, or Super Admin, with resolution code, resolution note, and any required customer communication summary)
  └─► CANCELLED (terminal — opened in error or no longer needed; requires reason)
```

**Rules:**
- Support cases are internal records. Customers receive notifications and outcomes through direct channels (push, email) but have no access to case records, internal notes, owner assignments, SLA timers, linked entity history, or risk context.
- Support escalation is represented through case type, priority, queue/owner coverage, linked records, and notifications; `ESCALATED` is not a support case status at launch.
- Support cases are structured but lightweight at launch. They are used for customer-facing issues, complaints, refund exceptions, reassignment disputes, payment exceptions, delivery exceptions, operational escalations, risk-alert follow-up, and cross-domain follow-up, but they are not required for every exception. Each case tracks type, status, priority, queue, optional individual owner, linked entities, notes, SLA timestamps, assignment history, and status history.
- Support cases link to domain records (orders, payments, deliveries, refunds) but do not own those records.
- A support case may link to multiple domain records when the issue spans related orders, payments, deliveries, refunds, or other scoped entities.
- Support-case notes are append-only; corrections are new notes, not edits to prior notes. Support cases may store concise customer communication summaries, but they do not act as a two-way inbox and do not store raw recordings, full transcripts, chat archives, or standalone evidence attachments at launch. Customer communication summaries link to relevant notification logs or existing domain records where applicable. Evidence is captured as concise notes or links to existing domain records.
- Customer Support Agents can request approved workflows in linked business domains but cannot execute them, except for explicitly permissioned fallback actions listed in the access matrix. The authorized owner of the affected domain workflow executes and records requested workflow changes.
- For refunds, support may create a refund-request action request with linked order, payment, refund context, and notes; that request does not create, approve, pay, void, write off, or ledger-post the refund by itself.
- Support action requests use `REQUESTED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, and `CANCELLED` states. `COMPLETED`, `REJECTED`, and `CANCELLED` are terminal for review-readiness purposes. A case is not ready for support review while a customer response is still required, but non-blocking notes or linked records do not prevent review readiness. A support action request does not alter linked domain records by itself.
- Linked domain workflow completion adds a support-case note and may move the case to `READY_FOR_SUPPORT_REVIEW` only when no blocking support action requests remain, but the case cannot close until a permissioned support actor records the required closure decision. `READY_FOR_SUPPORT_REVIEW`, `OPEN`, `IN_PROGRESS`, `WAITING_ON_CUSTOMER`, and `WAITING_ON_INTERNAL_TEAM` are non-terminal.
- No goodwill credits are supported at launch. Support outcomes use approved refunds or adjustments tied to orders.
- Support-triggered customer communications must use approved transactional notifications and templates with safe structured parameters. Support-case permissions do not grant notification-template administration, and support does not send arbitrary freeform outbound messages from the case workflow.
- Support cases are created manually at launch. Operational alerts, risk alerts, closure blockers, payment exceptions, delivery exceptions, and refund exceptions may supply default case context (case type, linked entities, priority suggestion, summary, and supporting context), but no case exists until a permissioned Customer Support Agent, in-scope Outlet Manager, or Super Admin explicitly creates it.
- Cases enter a centrally configured default queue based on type, priority, and outlet scope. Queue membership is derived from the actor's support permissions and outlet scope. Individual owner assignment is optional; assigning an individual owner does not remove queue coverage. When no individual owner is assigned or the owner is unavailable, the queue remains accountable for coverage.
- Permissioned Customer Support Agents, Outlet Managers, or Super Admins may manually move a case only to another allowed queue and must record a reason. Queue moves and individual owner changes are append-only assignment history events with actor, previous/new ownership, reason, and timestamp. Assignment changes affect support coverage and accountability only; they do not mutate linked orders, payments, refunds, delivery, inventory, or financial ledger records. Individual owner assignment sends a best-effort internal notification to that owner and records the notification timestamp; queue-owned cases rely on queue dashboards and SLA/priority views rather than notifying every queue member.
- Cases are outlet-scoped. A case's outlet scope is determined by its primary outlet and linked domain records. A Customer Support Agent cannot view cases for an outlet they are not assigned to unless granted explicit cross-outlet access.
- Cases linked to multiple outlets are not partially visible at launch. A Customer Support Agent, Outlet Manager, or Super Admin must have permission for all linked outlets, or explicit cross-outlet/global support access, to view or manage the case.
- Launch support routing uses default queues and permissioned manual moves only. Workload balancing, shift scheduling, round-robin assignment, skill-based routing, and complex support rosters do not grant case ownership, visibility, or routing authority at launch.
- Queue definitions are global configuration only. Only Super Admins or actors with explicit global support queue-configuration permission may create, edit, deactivate, reprioritize support queues, or change routing structures. Outlet Managers may work cases within scope and, when explicitly granted assignment permission, move cases between allowed existing queues under the case-assignment rule above. They request new queues, routing-structure changes, or per-outlet queue behavior only through Customer Support Agent or Super Admin escalation; they do not define or manage queue configuration directly, and no per-outlet queue-configuration approval path exists at launch. Queue configuration changes must be audited with actor, before/after values, reason, and timestamp.
- Detailed support-case views are sensitive reads and must be audit-logged with actor, case ID, outlet scope, and timestamp. Aggregate queue counts and list views do not require per-case read audit at launch.
- Support-case SLA timers are advisory only. They can drive notifications and dashboards, and urgent unowned or overdue cases may notify permissioned Customer Support Agents, Outlet Managers, or Super Admins according to priority and scope, but they do not block or advance orders, payments, refunds, deliveries, inventory, daily closing, financial closure, or ledger posting.
- Case records have their own retention window, independent of the subject domain records. At launch, support-derived records are retained for 36 months after the case reaches `CLOSED` or `CANCELLED`. Open support cases are not purged by age; the retention countdown starts only after `CLOSED` or `CANCELLED`. Support retention applies only to support-derived records and must not delete or weaken linked orders, payments, refunds, deliveries, financial ledger records, ledger entries, notification logs, closure blockers, operational risk alerts, or audit logs.

---

## 6.8 Vendor Refill Batch Lifecycle

```
EMPTY (confirmed empty cylinders at outlet)
  └─► IN_REFILL (authorized vendor-refill actor dispatches cylinders to vendor depot; unavailable)
        └─► FILLED (authorized vendor-refill actor confirms returned filled cylinders and condition)
```

**Rules:**
- The launch inventory-state transition is `EMPTY -> IN_REFILL -> FILLED`: confirmed empty cylinders enter `IN_REFILL` when dispatched to the vendor, and they become filled stock only after an authorized vendor-refill actor confirms return intake. The vendor depot work itself is external; the business record tracks audited inventory state transitions and ledger movements, not the vendor's internal process.
- `IN_REFILL` cylinders are excluded from available stock and cannot be sold or transferred.
- The vendor, size, and quantity sent must be recorded at dispatch; depot name, external reference, and notes may be recorded when known.
- Upon return, an authorized vendor-refill actor records received quantity and condition before confirming intake. Any shortage (fewer returned than dispatched) or overage (more returned than dispatched) must be reconciled through the standard inventory adjustment approval and ledger-posting lifecycle, with reason code, audit trail, and `INVENTORY_ADJUSTMENT_LAUNCH_V1` threshold evaluation before any Ledger movement. Overage cylinders remain pending and unavailable for sale, transfer, or reassignment until the inventory adjustment is posted.

---

## 6.9 Refill Exchange Request Lifecycle

Each `REFILL_EXCHANGE` order line creates one exchange request record. This record tracks the operational lifecycle of the physical cylinder exchange independently of the parent order status.

```
PENDING (created at order placement; expected cylinder vendor/size recorded)
  └─► IN_PROGRESS (order out for delivery; agent en route to customer)
        └─► RETURN_RECORDED (agent records returned cylinder vendor, size, condition at doorstep)
              └─► INTAKE_PENDING (delivery confirmed; cylinder awaiting outlet intake confirmation)
                    ├─► COMPLETED (returned-cylinder intake actor confirms intake; cylinder enters outlet inventory)
                    └─► FAILED (intake rejected or cylinder not returned; exception raised)

  CANCELLED  (order cancelled before delivery; no cylinder exchange occurred)
```

**Rules:**
- One exchange request exists per refill order line. An order with three refill lines has three exchange requests.
- Each expected returned cylinder requires its own outlet intake confirmation before that cylinder becomes outlet empty inventory.
- `RETURN_RECORDED` happens at the doorstep before PIN confirmation. The agent's recorded condition is the field fact of record (F-03 in §8).
- `INTAKE_PENDING` persists after the customer confirms delivery (PIN). The order can be `DELIVERED` while exchange requests are still `INTAKE_PENDING`.
- Financial closure for a refill order is blocked until all exchange requests for that order reach `COMPLETED` or `FAILED` with an approved exception (BI-19).
- A `FAILED` intake (wrong cylinder returned, damaged beyond acceptance, or cylinder not returned) creates an inventory exception and may supply default support-case context, but no support case exists until a permissioned Customer Support Agent, in-scope Outlet Manager, or Super Admin creates it. The failed intake does not by itself undo the completed customer delivery.
- During intake, a returned-cylinder intake actor (Inventory Clerk, Outlet Manager, or Super Admin within scope) may correct an agent recording mismatch in returned-cylinder vendor or size (e.g., the agent recorded Shell 6kg but the actual cylinder received is Total 6kg). The correction requires a reason code, before/after values, and an audit trail; material differences require Outlet Manager or Super Admin approval. A correction cannot create an unapproved size-mismatch adjustment after delivery; if the physical cylinder size does not match the accepted delivery facts, intake follows the `FAILED` intake and approved-exception path.

---

# 7. Business Rules

## 7.1 Outlet Allocation

**Geocoding pre-condition:** checkout is blocked if the selected delivery address has no resolved coordinates. Outlet allocation requires a valid latitude/longitude to calculate distances and determine eligible outlets. If geocoding fails on address save, the address is saved as `UNRESOLVED`, but checkout remains blocked until coordinates are resolved or confirmed. Customers may correct the map pin. Any non-customer coordinate correction requires explicit coordinate-correction permission and audit logging.

An order is allocated to one outlet. The outlet must satisfy all of the following before it is eligible:

1. **Geographic**: at launch, the customer's delivery address falls within one of the outlet's active radius-ring service zones, based on the confirmed customer and outlet coordinates. A distance at a ring's minimum boundary is eligible for that ring; a distance at its maximum boundary is not. Polygon service zones are out of launch scope.
2. **Operational**: the outlet must be active. For express orders, the outlet must also be currently operating at placement time — if no eligible outlet is currently operating, express delivery is unavailable at checkout and is not offered as a selectable delivery mode; the customer must choose batched delivery or not place the order. For batched orders, the outlet need only have a valid future delivery window available — it does not need to be currently operating when the order is placed. Inventory is reserved at placement time regardless.
3. **Delivery mode**: the outlet supports the requested delivery mode (express or batched).
4. **Payment method**: COD and mobile money are both enabled by default at launch unless an outlet policy overrides them. The outlet must support the customer's selected payment method. An outlet with no active payment method is ineligible for all orders and must be disabled from fulfillment until at least one payment method is active.
5. **Stock availability**: the outlet has sufficient available stock for all items in the order, except for the accessory-only relaxation described below when a core-cylinder outlet exists. For refill orders, this gate checks outgoing filled cylinders and accessories only; the expected customer empty cylinder is an incoming return and does not consume stock at placement.
6. **Refill vendor policy**: for refill items, the outlet accepts the customer's incoming cylinder vendor. If the outlet has no configured incoming vendor restriction, it accepts cylinders from all supported vendors by default.
7. **Same-vendor policy**: if the customer requests same-vendor outgoing, the outlet must have that policy enabled and the corresponding filled inventory.
8. **Capacity gate (optional)**: if the outlet has a configured active-online-order limit and has reached that limit, it is excluded from the candidate set entirely — it does not appear in the ranked list. An outlet below its limit is ranked by load as normal. Walk-in orders are not subject to the online order capacity gate and do not count toward that limit.

At launch, the global service-zone template set used by active outlet service zones has three active radius-ring templates: `CORE` covers `0 km <= distance < 5 km`, `STANDARD` covers `5 km <= distance < 10 km`, and `EXTENDED` covers `10 km <= distance < 15 km`. Each active online-fulfillment outlet must have confirmed outlet latitude/longitude before it can be eligible for allocation. Actual outlet coordinates are outlet master data, not hard-coded in this domain specification.

Among all eligible outlets, priority ranking uses:
- Distance to customer (closest first)
- Delivery fee (lowest first)
- Current active order load (least loaded first) — counted as active online delivery orders only, from assignment through delivery-run closure; walk-in sales do not count toward this load
- Outlet priority score (higher score preferred; default 0 when no score is set)

Delivery-agent capacity is not an outlet-allocation factor at launch. An otherwise eligible outlet is not excluded because no delivery agent is immediately available; delivery assignment happens after outlet allocation under the delivery lifecycle.

Customers do not choose their outlet. The serving outlet is determined by the active allocation policy.

Orders are not split across outlets. If no single eligible outlet can provide every requested stock item, allocation may fall back only to the closest outlet that satisfies every other outlet-allocation eligibility rule and has all of the order's core cylinder items; this fallback relaxes only accessory stock availability. The customer may place the order without unavailable accessories or not place the order; if an order includes one or more core cylinder items and no single outlet has all of them, the order cannot be placed. Accessory-only orders still require one eligible outlet with the requested accessory stock. Refill vendor-policy exhaustion is handled separately from core-stock absence: if candidate outlets have the required outgoing stock but all are excluded by incoming-vendor acceptance policy, the order follows the exhausted-candidate Super Admin intervention path in §7.6 rather than being treated as a stock-unavailable checkout failure.

## 7.2 Refill Pricing Matrix

Refill price is determined by the combination of:
- Incoming vendor (the cylinder the customer owns)
- Outgoing vendor (the cylinder the customer will receive)
- Cylinder size

For refill selection, same-vendor refill means the customer chooses an outgoing vendor that matches the incoming vendor. If no same-vendor choice is made, the outgoing vendor uses the outlet's configured default where present, otherwise the global Vengas default.

At launch, refill vendor choices are limited to globally configured supported vendors. Outlets may request vendor additions through Customer Support Agent or Super Admin escalation, but the supported-vendor list remains globally managed. `Other` or `Unknown` incoming cylinder vendors are not accepted for checkout, refill pricing, or doorstep exchange handling.

Incoming-to-outgoing refill pair eligibility is global at launch. Outlets may restrict which incoming vendors they accept, but they do not define outlet-specific incoming-to-outgoing pair overrides. A refill pair is orderable for a specific outlet only when the global pair is eligible and that outlet accepts the customer's incoming vendor.

A base refill price must exist for the outgoing vendor and cylinder size. If no base price exists for that outgoing vendor and size, no refill combination using it is orderable.

Cross-vendor refill pairs (incoming vendor ≠ outgoing vendor) require **both** the base refill price and a positive cross-vendor surcharge rule for the incoming vendor, outgoing vendor, and cylinder size. If either is missing, the pair is not priceable and therefore not orderable.

Same-vendor refills (e.g., Shell in → Shell out) require a base price rule. They do not require a surcharge rule. Same-vendor availability is constrained by the outlet's actual filled stock for that vendor and size; expected vendor-depot returns or cylinders in `IN_REFILL` do not make a same-vendor option orderable.

A single order may contain multiple refill exchange lines with different incoming vendors or cylinder sizes (e.g., one Shell 6kg refill and one Total 12kg refill in the same cart). Each line is priced independently using its own pricing triplet. All-or-nothing semantics apply across all lines: every refill in the order succeeds together or the whole order fails.

**Pre-checkout filtering**: the vendor options presented to a customer during refill selection are filtered to combinations that are currently fulfillable for their delivery address and incoming cylinder. Options that no eligible outlet can currently fulfil are not shown. Checkout revalidates availability and price at the moment of order placement as a second gate; the pre-checkout filter is a usability measure, not a substitute for revalidation.

## 7.3 Bundle Pricing

Accessories purchased in a bundle with a new cylinder receive a discounted price. The discount is applied at the component level: each bundle item has a bundled price and a standalone price.

If the same accessory is purchased without a new cylinder (accessory-only order), the standalone price applies. Bundles are not available for retrofit: a customer cannot add accessories to an existing order and retroactively receive bundle pricing.

Customers may add multiple bundles or multiple quantities of the same bundle to one cart. Bundles can coexist with new cylinders, refill exchanges, and standalone accessories under the mixed-cart, single-order, all-or-nothing fulfillment rules.

At launch, generic accessories are supported as distinct stock items tracked per SKU. The confirmed launch accessory set is 2m tubing and cooking grill; regulator accessories require explicit launch approval before being treated as sellable.

Catalog presentation groupings, such as variants used for browsing or filtering, do not create separate stock, pricing, order, or reporting truth. Those business facts attach to the concrete sellable item the customer buys.

Bundle discounts do not change inventory truth. Inventory remains accountable for the physical component items, not the bundle's pricing state. When a bundle reservation is released, those component items return to ordinary availability without bundle context; any later sale uses the then-applicable standalone or bundle pricing rule.

A bundle is a standalone sellable catalog offer and customer-facing bundle, but stock reservation, picking, delivery, and inventory accountability remain tied to its physical component items. The bundle label explains the commercial discount; it does not create separate physical stock.

## 7.4 Outlet Price Guardrails

Outlet Managers may adjust outlet-specific prices within bounded limits. A price change is within guardrail if **both** of these hold:
- The percentage change from the current approved basis is within the configured percentage limit
- The absolute change from the current approved basis is within the configured absolute limit

The current approved basis is the active approved outlet rule for the same price identity at the proposed effective start time, or the active global/default rule when no outlet rule exists.

If either limit is exceeded, the change is outside-guardrail and requires Super Admin approval before it activates. Within-guardrail changes take effect at their scheduled effective time without additional approval. If daily closing is overdue for the outlet, outside-guardrail activation is blocked unless a Super Admin urgent override with reason and note is recorded.

**Launch defaults:** product, refill, and accessory price changes — smaller of 10% or UGX 5,000 from the current approved basis; delivery-fee changes — smaller of 15% or UGX 2,000 from the current approved basis. Global prices, taxes, delivery-cost rules, guardrail rules, and missing-basis price changes always require Super Admin approval regardless of amount.

All price changes must have an effective window and audit record. "Immediate" means the effective start is now. All changes are future-dated or now-dated; no backdating.

Tax defaults to 0% for all product and outlet combinations unless an active tax rule is configured. When a non-zero tax applies, it is part of the customer's order total and is frozen into the order's price snapshot at placement. Multi-country or jurisdiction-specific tax workflows are not launch behavior unless explicitly added to launch scope.

## 7.5 Express Delivery Fee

The base delivery fee is determined by the assigned outlet's active service zone for the customer address. Each active outlet service zone maps to a global zone template; the template's default fee applies unless the outlet has an active service-zone fee override. Express delivery fee uses a global default multiplier of 1.5 against that service-zone base fee. This multiplier is configurable per outlet and per service zone. Outlet Managers may adjust the multiplier within the configured guardrail. The express fee premium is presented to the customer at checkout and is reflected in the order total. For cascaded orders where the reassigned outlet has a different service zone or base fee, the express fee is recalculated using the new outlet's rate.

At launch, the global service-zone template default base delivery fees are `CORE` UGX 3,000, `STANDARD` UGX 5,000, and `EXTENDED` UGX 8,000, using the radius-ring boundaries defined in §7.1 unless an outlet has an active service-zone fee override.

The delivery fee remains a separate customer-visible charge component in the order total; it is not folded into product, refill, or accessory prices. Delivery-fee adjustments are explicit commercial adjustments, and any actual money movement follows the payment, refund, and financial-ledger rules rather than rewriting the placed order's original charge.

## 7.6 Cascade and Reassignment Conditions

**Unchanged-terms cascade** (no customer notification required) occurs when the first-choice outlet rejects the order, marks it cannot-fulfill before picking, or its acceptance window expires, and the next eligible outlet can fulfill with unchanged delivery terms (same fee, same window, same vendor). During unchanged-terms cascade, the order's status remains `AWAITING_OUTLET_ACCEPTANCE` while the outlet assignment is updated internally to the next candidate; no customer notification or acceptance step is required.

**Customer notification and acceptance required** when a reassignment results in a higher delivery fee or amount due, a later or different delivery window, a different outgoing cylinder vendor, a refill-to-new-cylinder conversion, a cancellation/refund-only outcome, or any other change that affects what the customer is paying or receiving. When any such change is present, the following sequence applies before the customer is involved:

1. Stock is held at the candidate outlet (`REASSIGNMENT_HOLD`).
2. The candidate outlet **provisionally accepts** the changed order within a short window (launch defaults: 5 minutes for express, 15 minutes for batched). This is not full operational acceptance.
3. Only after provisional outlet acceptance is the customer notified and asked to approve the changed terms. The customer has a bounded acceptance window (launch defaults: 15 minutes for express, 30 minutes for batched).
4. If the customer accepts: the reassignment hold becomes the order's active reservation at the new outlet, the order becomes fully assigned and accepted by that outlet, and the previous outlet's reservation is released.
5. If the customer rejects or does not respond: the hold at the new outlet is released; the normal outcome is cancellation and any pre-payment creates a refund liability, unless the active reassignment policy requires escalation instead.

Delivery-window mismatches between outlets are not silently translated or rebooked during cascade. If the original customer-visible window cannot be honored by the candidate outlet, the change follows the customer-acceptance path above; if no candidate can resolve the window through that path, the all-candidates-exhausted escalation below applies. The reassignment does not remain an unchanged-terms cascade.

If the candidate outlet does not provisionally accept within its timeout, the hold is released and the next candidate outlet is tried — the customer is never notified about that outlet. Customers are notified only once an outlet has confirmed it can fulfil the changed order.

A stock hold by itself is not outlet acceptance. If no provisional outlet acceptance exists, the order remains in or returns to `AWAITING_OUTLET_ACCEPTANCE` rather than moving to customer approval.

For post-payment reassignment, the customer is not notified merely because the internal fulfilling outlet changed. Customer notification is required only when customer-visible details change, such as delivery fee, cash due, delivery window, outgoing vendor, cancellation/refund status, or refund collection outlet.

Every outlet evaluated but filtered out before a stock hold or outlet confirmation request records a candidate skip reason for operational and support explanation. Candidate skip records are diagnostic, not full reassignment attempts. They have a configured retention window; at launch, the retention window is 180 days after record creation. After that, they may be archived or summarized; terminal order, financial, settlement, refund, and audit records remain durable under their own retention rules.

A reassignment attempt begins only when stock is held or outlet confirmation is requested. It tracks the candidate hold, provisional outlet acceptance or rejection, outlet timeout, customer acceptance or rejection, customer timeout, and hold release. Once a reassignment attempt reaches a terminal outcome, that outcome is immutable except for appended audit or status history, and the attempt remains retained with the order because it affects customer promises, reservations, settlement reasoning, and support disputes.

For paid or otherwise active reassignment, the candidate outlet hold must be secured before the previous outlet reservation is released. If the previous reservation cannot release after the new hold succeeds, the previous reservation remains unavailable as `PENDING_RELEASE` until the release outcome is corrected and recorded; it is reported separately from normal reserved stock.

Any delivery fee difference resulting from reassignment is handled as an explicit price adjustment recorded against the order and reassignment record, complying with BI-20 (immutable order totals).

**A cascade that results in a lower delivery fee does not require customer acceptance.** For unpaid or COD orders, the lower fee reduces the amount due. For prepaid mobile-money orders, the difference becomes a refund liability created at financial closure. The customer is notified of the reduction but does not need to take any action.

A reassignment resulting in a **higher delivery fee** requires customer notification and acceptance of the revised payment terms before the reassignment activates. After customer acceptance, COD orders collect the fee increase as part of the COD amount due at delivery. For prepaid mobile-money orders, the accepted fee increase is collected by the delivery agent as a COD delta/top-up at delivery and remains distinct from the original prepaid amount. If customer acceptance is not obtained, the rejection/timeout outcome in this section applies.

**Reassignment is blocked** after picking has begun. Post-picking order reassignment is an Outlet Manager or Super Admin exception workflow. Picked stock does not return to available inventory until an authorized inventory/custody actor confirms the pick reversal or return outcome.

**When all candidate outlets are exhausted** (no eligible outlet accepts or provisionally accepts within its window), the order is not immediately cancelled. The order enters `REQUIRES_ADMIN_INTERVENTION` for Super Admin intervention, and permissioned Super Admins are notified by push. The timeout-driven cancellation/refund-flag outcome applies only if the escalation remains unresolved within the following windows (launch defaults):
- Express orders: 30 minutes from escalation
- Batched orders: 2 hours from escalation

During the intervention window, the Super Admin may manually assign an outlet or cancel the order. A manual assignment in this state is an audited exception override of the normal eligibility gates such as zone, hours, inventory, or outlet policy; it requires an explicit reason and does not make those gates optional in the ordinary allocation path. For refill vendor-policy exhaustion, intervention may include a Super Admin or permissioned Customer Support Agent contacting the customer through approved support or notification paths to offer cancellation-and-reorder alternatives, such as buying a new cylinder; it does not permit editing the placed order. If the timeout-driven cancellation/refund-flag outcome applies, the inventory reservation is released and any pre-payment becomes a refund liability.

The exhausted-candidate intervention window may be extended only by a Super Admin with reason and audit. A Customer Support Agent may request an extension through the support action-request path, but cannot execute the extension.

## 7.7 Delivery Agent Cash Handling

Agents collect exact cash due (COD amount plus any approved underpayment top-up, delivery-fee delta, conversion delta, or doorstep price-recalculation delta). If the collected amount differs from expected:
- Short collection: requires Outlet Manager approval, or Super Admin approval where the active threshold policy requires it, before delivery can be marked complete. At launch, Outlet Managers may approve short collections up to UGX 10,000 per order and UGX 50,000 per agent shift; above either limit requires Super Admin approval.
- Over collection: the excess is recorded as a cash discrepancy and requires Outlet Manager approval, or Super Admin approval where the active threshold policy requires it, before delivery can be marked complete. At launch, Outlet Managers may approve over-collection discrepancies up to UGX 10,000 per order and UGX 50,000 per agent shift; above either limit requires Super Admin approval. The approved discrepancy is reconciled at close.

Agents record cash collected. They cannot modify the expected amount. The expected amount is determined by the order total and any approved underpayment top-up, delivery-fee delta, conversion delta, or doorstep price-recalculation delta.

A delivery run may contain orders with different payment collection needs only when the run manifest makes cash due explicit per order.

Delivery agents are responsible for one combined cash-due amount (order COD total plus any approved top-up or delta). The business record preserves each component separately — base COD amount, delivery fee delta, underpayment top-up, conversion delta, doorstep price-recalculation delta — for reconciliation and audit; the agent neither sees nor enters component breakdowns. Customer receipts show the delta/payment breakdown after financial closure.

Agent cash reconciliation is per shift and rolls up into outlet daily closing. An agent may end active field work with open custody only when the Outlet Manager acknowledges the pending custody and carries it into reconciliation. The agent shift cannot fully close until two reconciliations are complete: (1) **cash reconciliation** — all cash collected matches expected or has an Outlet Manager-approved variance record for each order, or Super Admin approval where the active threshold policy requires it; (2) **custody reconciliation** — all outgoing items assigned to the agent are accounted for (delivered, returned to outlet, or covered by an approved exception). An open cash discrepancy or an unaccounted item blocks full shift close. The Outlet Manager must resolve or explicitly approve each open item before the shift can be fully closed. At launch, custody discrepancies within the Outlet Manager threshold are limited to one accessory item or one non-saleable/damaged cylinder discrepancy when documented custody facts identify the item and impact; missing saleable filled cylinders always require Super Admin approval. Discrepancies above threshold require Super Admin approval with reason, notes, stock/cash impact, and audit before the shift can fully close.

## 7.8 Daily Closing Restrictions

Daily closing is the outlet's end-of-day financial reconciliation. It summarizes cash count reconciliation, posted financial closures and receipts, cash and ledger entries, and any outstanding refund liabilities or internal settlement balances carried forward with Outlet Manager acknowledgement. It does not require every delivered order to reach `COMPLETED` or every settlement/liability to be resolved before closing. Overdue status follows the active daily-closing policy. At launch, each outlet daily closing for an outlet business day is due by 10:00 outlet local time on the next outlet business day. If the outlet is not operating on the next calendar day, the closing is due by 10:00 outlet local time on the next operating day. If it is not posted by that deadline, it is overdue until posted.

An overdue daily closing is an internal alert condition for permissioned daily-closing actors for that outlet; it does not by itself remove normal fulfillment access beyond the restrictions listed below.

When a daily closing is overdue for an outlet (marked overdue under the active daily-closing policy), the following are restricted for that outlet (unless a Super Admin urgency override with reason and audit trail is recorded). For cash refund payouts, an Outlet Manager within outlet scope may also use an urgent override under the active refund policy; the override must record a reason and audit trail:
- Cash refund payouts
- Large inventory adjustments (above the inventory adjustment approval threshold)
- Price changes that are outside the outlet guardrail
- Manual financial ledger adjustments
- Manual payment reassignment
- Above-threshold expense approval and posting (UGX 100,000 or more at launch)

Normal operations continue during overdue closing: online order placement, POS/walk-in sales, inventory reservation, acceptance, picking, dispatch, COD collection, mobile-money verification, stock intake, and within-guardrail price changes.

Overdue daily closing does not block creation or posting of refund liabilities or internal settlement records at financial closure. Those records capture business truth and must be posted even when the outlet is behind on closing; the overdue-closing restriction applies to the later cash payout or manual high-risk action, not to recognition of the liability or settlement.

Daily closing must list outstanding customer refund liabilities separately from internal settlement payables and receivables. Refund liabilities carry forward until paid, voided, or written off; settlement balances carry forward until a Finance Officer or Super Admin acknowledges or offsets them, or a Super Admin approves voiding them. Neither category blocks daily closing.

For post-payment reassignment, the outlet that received the mobile-money payment lists that receipt, any refund liability it owns, and the settlement payable. The final fulfilling outlet's reporting lists the sale, delivery work, direct COD/top-up/delta collections it received, inventory or estimated COGS where configured, and the settlement receivable.

## 7.9 Expense Controls

Outlet expenses are categorized and approval-gated by amount. Receipt attachments are required above the active threshold and optional below it. At launch, the expense approval and receipt-attachment threshold is UGX 100,000: expenses below UGX 100,000 may be approved without separate review when the active expense approval policy allows it, while expenses at or above UGX 100,000 require approval and a receipt attachment before posting. The active expense approval policy starts from global defaults and may define outlet/category overrides. Expense categories are globally configured; outlets may request additions through Customer Support Agent or Super Admin escalation, but they do not manage category definitions directly, and category configuration cannot bypass the active approval policy. An Outlet Manager cannot approve their own expense submissions (BI-16). Approved expenses post to the outlet's financial ledger immediately. Corrections require reversal records, not edits.

Every expense record captures outlet, category, amount, currency, payment method, payee/vendor if known, expense date, recorder, approver where applicable, status, receipt attachment status, and notes.

Expense payment method can be cash, mobile money, bank, or other. Only cash expenses affect outlet cash-on-hand and daily cash closing; all approved expenses affect outlet performance reporting under the same category and threshold policy.

Outlet performance reports may show gross revenue, estimated COGS, gross margin, recorded expenses, and estimated operating margin. Margin or profitability values that depend on configured costs or expense controls must be labeled as estimated unless accounting-grade costing and expense controls are approved.

At launch, daily and weekly sales reports and low-stock reports satisfy reporting needs. Demand forecasting is not supported at launch.

Product/outlet cost inputs are optional and manually maintained at launch. When configured, they may use outlet-specific values with a global fallback; cost changes are audited and may carry effective dates for historical estimated-margin reporting. When configured cost inputs are used for estimated margin reporting, the applicable cost for each completed sale is fixed at financial closure; later cost changes do not silently restate closed-sale estimates.

## 7.10 Late Payment References

If a customer submits a mobile-money reference after their order has been cancelled:
- The reference is flagged; the cancelled order is not reopened. Payment history links the late reference to the cancelled order and, if reused, to the recreated order.
- The same customer may reuse the same reference on a new order within 7 days, including a recreated order with different items, subject to normal checkout validation.
- The recreated order runs normal outlet allocation from scratch; the cancelled order remains cancelled.
- On the new order: if the new total matches the paid amount, it applies normally; if the new total is higher, the shortfall requires an approved COD top-up before fulfillment proceeds; if the new total is lower, a cash refund liability is created for the overage.
- After 7 days, mediation by an explicitly permissioned Outlet Cashier or Outlet Manager within scope, or by a Super Admin, is required. Cross-customer reuse of a reference requires an audited override by one of those actors with explicit cross-customer override authority; ordinary self-service reuse remains same-customer only.

Late references and overpayments do not create customer wallet or credit balances at launch. They remain specific payment exceptions or refund liabilities.

## 7.11 Walk-In / POS Rules

Walk-in orders require explicit POS/cashier permission and share the same inventory as online orders. Walk-in stock reservation is immediate and final (no pending state; no delivery delay). A walk-in sale reaches financial closure at sale completion and receives its receipt number and immutable receipt then. At launch, the same pricing rules apply to walk-in and online orders; different POS/online pricing requires a later explicit policy decision. Operating hours do not apply to walk-in POS; they govern online order allocation only.

Walk-in refill exchanges use the same incoming/outgoing vendor rules, pricing matrix, same-vendor policy checks, and returned-cylinder inspection as online refills. The delivery and agent custody legs are skipped; an explicitly permissioned POS/cashier actor handles the exchange directly at the counter. The same all-or-nothing semantics apply: if the returned cylinder is unacceptable, that POS/cashier actor may offer a full-order conversion under the same approved adjustment path; if the customer refuses conversion, the transaction does not proceed.

Anonymous walk-in customers are permitted. Refunds, returns, and support follow-up require customer identity. Anonymous walk-in mobile-money payments still require provider, transaction reference, and verified amount capture, and the same per-provider reference uniqueness rules apply.

---

## 7.12 Failed Delivery Fee Waiver

No delivery fee is charged on any failed delivery, regardless of the failure reason. This applies to all failure scenarios including customer unavailable, customer refused delivery, PIN lockout, safety issue, damaged returned cylinder where the customer refuses conversion, and any other terminal failure outcome. For COD orders, the waived fee means the delivery fee is not collected from the customer by the agent. For prepaid mobile-money orders, the waived delivery fee becomes part of the refund liability created when the order transitions to `CANCELLED_PENDING_REFUND`.

Failed COD deliveries keep a zero-collection payment fact tied to the failed delivery reason. Failed prepaid mobile-money deliveries create a cash refund liability for the full paid amount until audited cash refund completion.

---

## 7.13 Mobile Money Order Expiry

Unpaid mobile-money orders have a two-stage configurable expiry window. The clock starts when the order reaches `AWAITING_CUSTOMER_PAYMENT`. Launch durations are global defaults; changes or outlet/payment-method overrides require Product Manager approval plus the operations approval authority required by the active policy. Launch defaults:

- **Stage 1 — Warning (at 30 minutes):** a reservation-expiry warning notification is sent to the customer. The reservation remains active.
- **Stage 2 — Payment-expiry cancellation (at 60 minutes total):** if no payment reference has been submitted within the 30-minute grace period following the warning, the order is cancelled and the inventory reservation is released. The total window from placement is 60 minutes.

Submitting a reference at any point before cancellation stops the expiry clock and moves the order to `AWAITING_STAFF_VERIFICATION`. The expiry path is permanently closed once a reference is submitted, regardless of the verification outcome, and payment-verification delay does not trigger stock release under the unpaid-expiry policy.

COD orders do not have a payment expiry. They may time out through the outlet acceptance workflow if the outlet does not accept within the acceptance window, but no payment deadline applies.

---

## 7.14 Post-Delivery Return Policy

Customers may return eligible items or cylinders after delivery subject to all of the following conditions:

1. **Return window**: 7 calendar days from the confirmed delivery timestamp.
2. **Customer drop-off**: the customer is responsible for returning the item or cylinder to the owning outlet in person. No reverse-logistics pickup is provided.
3. **Customer identity required**: the customer must be identifiable at the point of return. Anonymous post-delivery returns are not accepted.
4. **Outlet inspection and approval**: an authorized outlet return-intake actor must inspect the returned item or cylinder, and the active return policy's approval role must approve the condition before a refund liability is created or any returned-stock effect is recognized. A return that is rejected on condition does not create a refund or change inventory availability.
5. **Cash-only refund**: post-delivery return refunds are paid in cash at the outlet. No mobile-money or alternative refund method applies.

Return eligibility, approved condition, refund amount, drop-off requirement, and approval role are controlled by the active product-type return policy. The launch policy uses the conditions above; later policy changes must preserve explicit customer identity, approval, and audit requirements.

Post-delivery returns are processed against the original order whenever possible. Unlinked manual refunds are Super Admin-only exceptions.

---

## 7.15 Cart Behaviour

- Cart creation, cart changes, quotes, and order placement require an authenticated customer account at launch. Anonymous guest cart and checkout are not launch behavior.
- Out-of-stock cart items are marked unavailable and remain in the cart until the customer removes them or stock returns. The customer must resolve them (remove or wait for stock) before checkout can proceed.
- Cart and checkout quote prices are estimates until order placement. Final pricing is recalculated after outlet allocation and is locked only when the order is placed and the price snapshot is created.
- Catalog or pricing changes that affect cart lines — price changes, product disablement, or bundle composition changes — require customer review and acknowledgement before cart quote or checkout can proceed.
- If a product's price changes while a cart is open, cart prices update to reflect the current price and the customer receives an explicit price-change notice before checkout.
- Disabled products are not orderable or sellable at any outlet, even if stock exists. Any cart item referencing a disabled product is marked unavailable. Already-placed orders referencing a disabled product are not affected.
- If a bundle's composition changes while a cart is open (components added, removed, or repriced), any cart line containing that bundle is flagged invalid. The customer must review and acknowledge the change before cart quote or checkout can proceed.
- Active carts are customer-derived commerce state, not durable order, payment, ledger, custody, receipt, or audit facts. They persist across customer sessions and devices, but a cart with no customer fetch, mutation, or quote activity for 90 days is marked `ABANDONED`. After an additional 30 days, abandoned cart item detail is no longer retained as active cart detail, while a safe cart summary remains. Checked-out carts and placed orders are never part of abandoned-cart cleanup.

---

## 7.16 Operational Risk Alerts

Repeated custody/cash exceptions by delivery agents and staff are evaluated against rolling-window thresholds, including repeated short collections, missing returned cylinders, cash discrepancies, custody losses, forced-closure adjustments, and unresolved custody exceptions. When a threshold is exceeded, an operational risk alert is raised; thresholds start from global defaults and may define outlet and role overrides.

Risk alerts are **informational only**. They do not:
- Suspend or block the flagged user by themselves
- Block delivery assignment or alter assignment ranking
- Deduct from pay or assign personal financial liability
- Create a support case without a manual Outlet Manager or Super Admin decision

Alerts are visible only to permissioned Outlet Managers and Super Admins within their authorized scope. Alert notifications, when configured, use push and are limited to those same permissioned actors; they are not customer-facing and are not sent to the flagged staff member or agent. The flagged staff member or agent does not see the alert. Any action in response to an alert — such as opening a support/escalation case — is a deliberate, manual decision by an Outlet Manager for that outlet or a Super Admin. The risk-alert lifecycle states are `OPEN`, `ACKNOWLEDGED`, `DISMISSED`, and `CASE_OPENED`; acknowledging or dismissing an alert requires a note/reason and audit. When an Outlet Manager for that outlet or a Super Admin chooses to open a support case from a risk alert, the risk alert may supply default case context (subject user, alert type, trigger details); no case exists until that actor confirms creation. Once the manual case is created, the case is permanently linked to the alert and the alert transitions to `CASE_OPENED`. Detailed risk-alert views are sensitive reads and are audit-logged with actor, outlet scope, alert ID, and timestamp; aggregate dashboard counts and list views do not require per-alert read audit at launch. Active alerts remain in review until they are dismissed, escalated, or reviewed with the required note/reason and audit. Eligible dismissed derived alerts are summarized after the active staff-risk retention window, which is 24 months at launch; `CASE_OPENED` alerts follow the linked support-case closure and retention path first, then use the same risk-alert retention window. Risk-alert retention must not delete the underlying custody events, delivery runs, cash variances, forced closures, ledger adjustments, or audit logs. If work is manually assigned despite an active relevant risk alert, the assigning Outlet Manager or Super Admin must record a reason, which is retained in audit.

## 7.17 Stock Count Behaviour

Launch inventory accountability is aggregate by outlet, vendor, cylinder size, filled status, condition, and item/SKU as applicable. Individual cylinder serial-number tracking is not required at launch and is not a prerequisite for stock counts, reservations, transfers, custody reconciliation, or vendor refill movements.

Cylinder fill lifecycle (`FILLED`, `EMPTY`, `IN_REFILL`) is independent from availability and condition status such as available, reserved, damaged, quarantined, in transit, lost, sold, or returned. Stock is sellable, reservable, or transferable only when both its fill lifecycle and its availability/condition status permit that business action.

Stock counts do not freeze outlet operations. When a count begins, expected quantities are fixed as the count-start basis. Ledger movements (reservations, releases, sales, intake) that occur during the count window are tracked and used to calculate variance at count close. Orders may continue to be placed, accepted, and fulfilled while a count is in progress.

## 7.18 Low-Stock Alerts

Low-stock alerts are based on available stock only. Launch defaults: saleable filled-cylinder stock alerts when available quantity is at or below 2, saleable accessory alerts when available quantity is at or below 1, and empty-cylinder or non-saleable stock alerts disabled by default with threshold 0 unless explicitly configured. These defaults may be overridden per outlet, product, or stock item.

Alerts are scoped to the outlet and stock item. Repeated alerts for the same outlet/stock item are deduplicated while the item remains below threshold during the four-hour launch threshold window. Alerts are visible to permissioned Outlet Managers, Area Managers, and Super Admins within their authorized scope. They may show relevant `IN_REFILL` and incoming-transfer context, but they do not reserve stock, block orders, alter allocation, or forecast demand.

## 7.19 Delivery Cost Reporting

Delivery estimated cost is an outlet performance reporting input, not customer-facing delivery pricing. Customer delivery fees, failed-delivery fee waivers, refunds, and cash due are governed by the order and pricing rules elsewhere in this document. Delivery cost policy/rule changes are audited and effective-dated.

At launch, delivery cost reporting defaults to treating each order separately. Approved run-level cost allocation may use an even split or delivery-fee/cylinder-count weighting across orders. Run-level cost allocation finalizes when the delivery run closes. Failed orders in batched runs may carry an internal failed-attempt delivery cost for estimated outlet performance reporting, but customer revenue remains zero, no delivery fee is charged to the customer, and failed-attempt cost is reported separately. Closed delivery cost allocations are not recalculated directly; corrections require audited adjustment records. Reports using delivery cost must label resulting margin or profitability values as estimated unless accounting-grade costing and expense controls are later approved.

## 7.20 Notification Channel Boundaries

General business notifications use push and email at launch. SMS is reserved for customer authentication OTP only; WhatsApp and customer-configurable notification preferences are not launch behavior. Notification types are still classified as transactional or non-transactional for future policy use.

Launch notification channels are assigned by event type rather than customer preference: order confirmation, delivery PIN, payment confirmation, failed delivery, and reservation-expiry warning use push and email; dispatch notifications, exhausted-candidate Super Admin intervention alerts, outlet low-stock alerts, and configured operational risk alerts use push only.

Support-requested customer notifications use the same launch notification boundaries: they must be approved transactional notification templates or events with safe structured parameters, and support-authored freeform message bodies are not supported.

Notifications are best-effort and must not block or reverse committed order, payment, inventory, delivery, refund, support, or financial-ledger activity. Failed or unsent notifications are surfaced for operational review where relevant.

Refund collection codes are not sent in push, email, or SMS message bodies. Notifications may tell the customer that a refund is collectible, identify the collection outlet, and direct them to the authenticated customer experience or to a permissioned Customer Support Agent or Super Admin for audited reveal after customer verification. Delivery PIN notifications use push and email at launch and remain governed by the PIN exposure and fallback rules in §6.3.

---

# 8. Key Business Flows (Business-Rule Level)

## F-01: Online Order to Delivery

1. Customer places order from a selected delivery address with confirmed coordinates. The coordinates may be provider-resolved or a customer-confirmed manual pin.
2. The serving outlet is selected from outlets that serve the address and meet all allocation criteria (§7.1).
3. Stock for all order items is reserved at the selected outlet.
4. For mobile money: customer pays independently, submits reference, an authorized outlet payment-verification actor verifies, and any required payment-gate resolution completes before outlet acceptance (§6.2, BI-03).
5. For COD: order proceeds directly to outlet acceptance.
6. Outlet accepts. An authorized outlet picking actor picks items. Delivery-agent assignment and delivery-run creation follow the active outlet delivery policy: assignment by the default policy, with Dispatcher or Outlet Manager manual assignment or override before pickup within outlet scope.
7. Outlet handover confirmation and agent receipt confirmation are both recorded; custody transfers to agent.
8. At customer door: for refill orders, agent first records every expected returned cylinder's vendor, size, condition, any approved conversion, COD delta or cash collection, and delivery exception. Customer confirms with PIN only when the final delivered order and amount due are correct (BI-08).
9. PIN confirmation atomically commits delivery status, order status, outgoing stock commitment, returned-cylinder field recording, cash collection recording, and payment status (BI-08).
10. For refill orders: a returned-cylinder intake actor confirms intake for every expected returned cylinder, or a failed intake receives an approved exception. For all online deliveries, financial closure waits until delivery-run custody is closed, those intake/exception outcomes are complete, and no unresolved closure blockers remain.
11. Normal financial closure occurs from recorded business facts, generates the receipt, and seals the order (BI-05).

## F-02: Reassignment with Changed Terms

1. Order is assigned to Outlet A. Outlet A rejects, its acceptance window expires, or it marks cannot-fulfill before picking.
2. The next eligible outlet (Outlet B) is selected from the fallback list. Every outlet evaluated but filtered out before a full reassignment attempt is recorded with a skip reason (e.g., zone mismatch, no stock, policy mismatch). This gives operations and support visibility into why an outlet was bypassed without treating it as a full reassignment attempt.
3. Stock is held at Outlet B under a `REASSIGNMENT_HOLD` reservation; it is unavailable to other orders but not yet a final outlet assignment.
4. Outlet B provisionally confirms it can fulfill under the new terms (within a short window).
5. Customer is notified of the changed terms. Customer has a bounded window to accept.
6. If customer accepts: the reassignment hold becomes the order's active reservation; the order becomes fully assigned and accepted by Outlet B. Outlet A's reservation released.
7. If customer rejects or does not respond: stock hold at Outlet B released; the normal outcome is order cancellation and any pre-payment becomes refund liability, unless the active reassignment policy requires escalation instead.

## F-03: Damaged or Unacceptable Returned Cylinder at Doorstep

1. Agent delivers filled cylinder, attempts to collect empty cylinder.
2. Returned cylinder is damaged or otherwise unacceptable for refill (rusted, valve broken, etc.).
3. Agent records condition with required photo evidence. The unacceptable cylinder is refused and does not enter outlet inventory.
4. Agent offers full-order conversion to new cylinder under the approved full-order adjustment path. Customer must pay the price delta by COD.
5. If accepted and Outlet Manager approval is recorded: conversion is recorded as an explicit adjustment while the original refill order line and price snapshot remain immutable; agent collects the COD delta; PIN confirmed; delivery marked delivered.
6. If refused: delivery fails; undelivered outgoing goods are returned to the outlet and receipt-confirmed by an authorized outlet return-receipt actor before run closure; the full order fails under all-or-nothing rule (BI-09). No delivery fee charged (§7.12). Mobile-money prepayment becomes refund liability.

## F-04: Post-Payment Outlet Reassignment (Settled by Finance)

1. Customer pays mobile money to Outlet A's merchant account.
2. After payment verification, Outlet A cannot fulfill and the order is reassigned to Outlet B either with no customer-visible change or after the customer accepts any changed terms.
3. Outlet B fulfills successfully.
4. At financial closure: the original payment remains attributed to Outlet A and its payment account; Outlet B holds the revenue from fulfillment only because it successfully completed the order.
5. An internal settlement entry is created for only the prepaid mobile-money value actually received by Outlet A and owed to Outlet B (BI-13). COD deltas, underpayment top-ups, conversion deltas, doorstep price-recalculation deltas, and delivery-fee increases collected by Outlet B post directly to Outlet B's financial ledger and are not settled from Outlet A.
6. If instead the reassigned paid order fails or is cancelled before successful completion, no internal outlet settlement is created; any refund liability remains with Outlet A as the paid-to outlet.
7. Finance Officer or Super Admin reviews and acknowledges the settlement. No cash transfer between outlets occurs at launch.
8. Any reassignment-driven prepaid overage refund liability belongs to Outlet A (the outlet that received the funds), but it is created only after financial closure determines the final amount due. Before financial closure, the customer may see only a pending refund calculation, not a collectible refund. The customer collects that refund from Outlet A only; cross-outlet refund payout is not supported at launch.
9. For reassigned prepaid orders, the customer receipt reflects the fulfillment outlet, the payment-receiving outlet, and the refund-collection outlet when applicable; it does not expose internal settlement status or mechanics. Permissioned Outlet Managers, Area Managers, Finance Officers, and Super Admins may see internal settlement status within their reporting or financial-view scope.

## F-05: Forced Financial Closure (Super Admin Exception)

1. An order remains in the pending-closure view beyond the SLA window.
2. Escalation reaches Super Admin.
3. Super Admin assigns a resolution path to every open closure blocker, such as waiving with adjustment, marking lost stock, confirming damage, or accepting a cash variance.
4. Required forced-closure approval is obtained under the active materiality policy: material forced closures require dual approval, and the requester cannot satisfy the second approval; low-risk forced closures may use single Super Admin approval within that policy. At launch, a forced closure is material if its combined financial or inventory impact is UGX 100,000 or more, any saleable filled cylinder is missing or lost, any refund or write-off is created, or the closure resolves more than one blocker category. Low-risk forced closures below those thresholds may use single Super Admin approval within the active policy. The materiality policy starts from global defaults and may define outlet/category overrides.
5. After required approval, any compensating inventory and cash adjustment entries are posted with explicit linkage to the forced closure.
6. Loss or variance responsibility defaults to the outlet or delivery run that held custody when the loss, damage, or cash variance occurred. When custody facts identify that holder, the custody-chain result is the default; Super Admin may override responsibility only when custody facts are unclear or conflicting, and only with reason, note, and audit trail. Staff, agent, and approver identities remain visible in custody and audit reports for operational accountability. Resulting losses affect outlet-level reporting, not staff payroll or personal financial liability.
7. The forced-closure audit trail identifies the order, every resolved blocker, approving actor(s), reason, timestamp, the resolution path for each blocker, and any adjustment facts used to correct the business truth.
8. Order moves to `COMPLETED`. All records of the exception are permanent.

---

## F-06: Defective Outgoing Cylinder at Pickup

1. Delivery agent arrives at outlet to pick up order.
2. Agent inspects outgoing cylinder and finds it defective (dented, rusted, valve broken, expired).
3. Agent records "defective product" with reason and required photo evidence. The defective unit is quarantined in outlet stock.
4. Outlet attempts immediate replacement from available stock of the same vendor and size.
5. If replacement is available for an express order, or available for a batched order without disrupting the delivery route or window: agent takes replacement, delivery proceeds normally.
6. If replacement is unavailable, or if a batched replacement would disrupt the delivery route/window: the order is moved to the next eligible dispatch opportunity or batch/window. Outlet Manager or Super Admin may override the reschedule decision.
7. The original defective cylinder remains quarantined pending outlet inspection and stock adjustment.

---

## F-07: Unexpected Same-Size Vendor Returned Cylinder at Doorstep

1. Delivery agent arrives at customer's address for a refill exchange.
2. Customer presents a returned cylinder. Agent inspects and identifies it as an unexpected vendor, with the correct size and acceptable condition.
3. Agent records the actual incoming vendor with required photo evidence and confirms the size matches the outgoing cylinder.
4. The refill price is recalculated using the actual same-size combination (actual incoming vendor, outgoing vendor, cylinder size).
5. If recalculated price is higher than the original paid or COD amount due: a COD top-up for the delta is required. Outlet Manager approval is required before PIN confirmation and delivery completion.
6. If recalculated price is lower than the original paid or COD amount due: unpaid/COD orders reduce the amount due; prepaid mobile-money orders create a cash refund liability at financial closure. The exchange proceeds immediately without waiting for refund resolution.
7. If recalculated price equals the original paid or COD amount due: exchange proceeds normally with no delta.
8. The actual incoming vendor becomes the exchange fact for intake and pricing. The original order line and price snapshot remain immutable; any delta is recorded as an explicit adjustment.

---

# 9. Authorization Edge Cases

These cases are explicitly declared to prevent ambiguous business interpretation.

**E-01**: A payment-verification actor cannot verify a mobile-money payment for an order assigned to a different outlet, even if both outlets are in the same city.

**E-02**: A Delivery Agent can see the customer's phone number only while an order is assigned to them and active. They lose this access when the delivery reaches a terminal state. Phone-number access is scoped to the active assignment and is audit-sensitive under the active audit policy.

**E-03**: Customer Support Agents cannot view another outlet's cases unless a Super Admin has granted explicit cross-outlet support access. There is no implicit cross-outlet access based on case type or priority.

**E-04**: An Area Manager can view reports and operational data for their assigned outlets, but cannot view operational risk alerts or perform any action (accept order, verify payment, approve refund, adjust inventory).

**E-05**: Refund liabilities at or above the approval-required threshold require approval from the Outlet Manager for that outlet within their active approval threshold, or from a Super Admin above that threshold. At launch, the approval-required threshold is UGX 50,000 and the Outlet Manager approval threshold is UGX 500,000 within outlet scope. The approving actor cannot approve a refund on an order where they submitted the refund request themselves.

**E-06**: A Super Admin performing a forced financial closure must provide a reason. This action is always audit-logged with before/after values. No exception to audit logging exists, even for Super Admin.

**E-07**: An Outlet Manager can approve refunds for their own outlet within their authorized threshold and approval policy. At launch, Outlet Managers may approve refund liabilities from UGX 50,000 through UGX 500,000 within their outlet scope; refunds above that threshold require Super Admin approval. Outlet Managers cannot approve refunds at another outlet, and they cannot approve their own submitted refund request.

**E-08**: A Dispatcher can reassign delivery agents to orders within their outlet until the point of pickup with a recorded reason. After pickup, only an Outlet Manager or Super Admin can reassign, and the action requires an explicit reason code.

**E-09**: A single human account may hold both Outlet Cashier and Inventory Clerk permissions at the same outlet only when the active authorization policy permits that combination. The combination must be explicit, recorded, and subject to audit on sensitive actions; the actor still cannot approve their own submissions or bypass normal approval thresholds.

**E-10**: Authorization policy changes that affect financial behavior, reporting, audit semantics, user-visible functionality, or launch scope require Product Manager approval and co-approval under the active authorization-change policy; the Super Admin proposing the change cannot satisfy that co-approval alone. Product Manager approval is business-governance authority, not runtime platform permission.
