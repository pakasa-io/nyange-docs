# Identity & Authorization

**Intent**: Define all personas, the complete access matrix, the authentication and authorization model, and
authorization edge cases for the Nyange platform.

**Sources**: §2 Personas, §3 Access Matrix, §4 Auth Model, E-01–E-10

---

## Invariants

**BI-16 — A persona cannot approve their own submission.**
Any action requiring approval or co-approval cannot be approved by the same individual who submitted or requested it,
regardless of their role. This includes expense above threshold, inventory adjustment, refund above threshold, price
change above guardrail, material forced financial closure, and sensitive authorization-policy change.

---

## Personas

These ten personas are the canonical business archetypes used in this specification. They describe typical
responsibilities and access patterns, but they are not identity account categories.

Runtime authorization is determined by explicit permissions and scope assignments. A single human account may hold
multiple permission bundles across one or more outlets.

### P-01 Customer

An individual who orders gas, refills, and accessories through the platform.

**Context**: Logs in before using cart or checkout at launch, and places orders online from a registered account. Uses
the authenticated customer experience to track order status and access customer-visible notices. May have multiple saved
addresses. Pays by mobile money or cash. Collects cash refunds in person at the outlet.

**Cannot**: Manage outlet operations, view other customers' data, approve payments, access financial records, or
override business rules or policy controls.

---

### P-02 Delivery Agent

A field operative assigned to deliver orders from outlet to customer.

**Context**: Sees only orders assigned to them and customer phone numbers for assigned active deliveries. Performs
required field actions during active delivery. Collects cash (COD), records returned cylinders, and enters
customer-provided delivery PINs for confirmation. Reports delivery failures. Submits cash handover at end of shift.

**Cannot**: Reassign orders, view other agents' assignments, verify mobile-money references, modify inventory, issue
refunds, or access financial records.

---

### P-03 Outlet Cashier

An outlet-level staff member responsible for walk-in POS transactions.

**Context**: Records walk-in sales through the outlet POS workflow. Accepts cash or mobile-money from walk-in customers.
May also verify mobile-money payment references for the outlet and assist with delivery PIN fallback when explicitly
permissioned. Issues walk-in receipts. Has no delivery responsibilities, and any online-order responsibilities are
limited to explicitly permissioned payment-verification and PIN-fallback workflows.

**Cannot**: Manage online orders, approve refunds, adjust inventory, or access another outlet's data.

---

### P-04 Inventory Clerk

An outlet-level staff member responsible for stock management.

**Context**: Manages stock counts, submits inventory adjustment requests, confirms returned-cylinder intake after
delivery, initiates outlet-to-outlet transfer requests, and records vendor refill movements. Works within their assigned
outlet only.

**Cannot**: Accept/reject orders, verify payments, issue refunds, access financial records, or approve their own
adjustment requests.

---

### P-05 Dispatcher

An outlet-level staff member responsible for delivery coordination.

**Context**: Creates and manages delivery batches, assigns delivery agents to orders and runs, tracks delivery progress,
and may reassign agents before pickup. Schedules delivery work within the outlet's active delivery windows and policy.

**Cannot**: Verify payments, approve refunds, adjust inventory, access financial records, or modify outlet policies.

---

### P-06 Outlet Manager

The operational owner of a single outlet.

**Context**: Accepts or rejects orders, verifies mobile-money payment references when explicitly permissioned,
reconciles daily cash, approves refund liabilities for their outlet according to the active approval policy, submits
above-threshold inventory adjustments for approval, oversees staff, manages delivery operations, and views
outlet-specific reports. Has elevated authority over P-03, P-04, and P-05 within their outlet.

**Cannot**: Access another outlet's data (unless explicitly granted), change global prices or catalog, approve refunds
outside their outlet scope, or override Super Admin controls.

---

### P-07 Customer Support Agent

A support operative who handles complaints, escalations, and exceptions.

**Context**: Opens and manages support cases. Views case-linked, outlet-scoped order, delivery, payment, and inventory
details needed for investigation. Requests approved workflows (refunds, inventory corrections, reassignments) on behalf
of the resolution — but does not execute those workflows directly. Where explicitly permissioned, performs support-only
fallback actions such as delivery PIN fallback and refund collection-code regeneration, unlock, or audited
customer-verified reveal. Logs customer communication summaries.

**Cannot**: Directly mutate orders, payments, inventory, or financial records outside explicitly permissioned fallback
actions. Cannot approve, pay, void, or write off refunds. Cannot approve their own action requests. Cannot view cases
outside their outlet scope unless explicitly granted cross-outlet access.

---

### P-08 Area Manager

A regional oversight role assigned to multiple outlets.

**Context**: Monitors performance and escalates exceptions across assigned outlets. Has read access to outlet
operations, inventory, and financial summaries for all assigned outlets. Can escalate exceptions but does not perform
direct outlet operations.

**Cannot**: Perform outlet operations (accept orders, verify payments, adjust inventory), view or manage operational
risk alerts, access outlets outside their assignment, or override Super Admin controls.

---

### P-09 Finance Officer

A financial operations role with ledger and settlement visibility.

**Context**: Reviews financial ledger entries, reconciles internal settlements between outlets, acknowledges settlement
records, and reviews daily closing summaries plus expense reporting and policy outcomes. Works across all outlets.

**Cannot**: Mutate order, delivery, or inventory state. Cannot approve their own submitted records. Cannot manage
catalog or pricing.

---

### P-10 Super Admin

The global platform authority. Exactly one or more individuals with this designation; access is tightly controlled.

**Context**: Full operational and configuration authority across all outlets, business domains, and records. Approves
above-guardrail price changes, above-threshold inventory adjustments, forced financial closures, break-glass
authorization policy changes, and other actions that no other persona can execute. Sensitive Super Admin mutations and
overrides require explicit reason codes and are permanently audit-logged.

**Cannot**: Issue unaudited mutations. Super Admin authority does not bypass audit requirements — it extends them.

---

## Access & Authorization Matrix

Notation:

| Symbol          | Meaning                                                                                 |
|-----------------|-----------------------------------------------------------------------------------------|
| **Full**        | Full read + write within scope                                                          |
| **Scoped**      | Access limited to assigned outlet(s)                                                    |
| **Read**        | Read-only                                                                               |
| **Own**         | Access to own records only                                                              |
| **Request**     | Can initiate but not approve                                                            |
| **Approve**     | Can approve requests from others; cannot approve own                                    |
| **Acknowledge** | Can record review/acknowledgement of settlement or closing records without voiding them |
| **Offset**      | Can offset internal settlement records within authority                                 |
| **Threshold**   | Authority limited by a configured financial/quantity threshold                          |
| **–**           | No access                                                                               |

| Capability Domain                            | P-01 Customer | P-02 Agent     | P-03 Cashier                                     | P-04 Inv Clerk                    | P-05 Dispatcher | P-06 Outlet Mgr                               | P-07 Support                         | P-08 Area Mgr           | P-09 Finance         | P-10 Super Admin                                |
|----------------------------------------------|---------------|----------------|--------------------------------------------------|-----------------------------------|-----------------|-----------------------------------------------|--------------------------------------|-------------------------|----------------------|-------------------------------------------------|
| **Own account & address**                    | Full          | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Address coordinate correction**            | Own (map pin) | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Product catalog browsing**                 | Read          | –              | Read                                             | –                                 | –               | Read                                          | Read                                 | Read                    | –                    | Full                                            |
| **Cart & order placement**                   | Full          | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Order status tracking**                    | Own           | Own (assigned) | Scoped                                           | –                                 | Scoped          | Scoped                                        | Scoped                               | Read (assigned outlets) | Read                 | Full                                            |
| **Order acceptance / rejection**             | –             | –              | –                                                | –                                 | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Outlet picking**                           | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Outlet handover confirmation**             | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Failed-order return receipt**              | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Run-level returned-cylinder receipt**      | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **POS / walk-in sales**                      | –             | –              | Scoped (with explicit permission)                | –                                 | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Mobile money verification**                | –             | –              | Scoped (with explicit permission)                | –                                 | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Payment account administration**           | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Payment reference submission**             | Own           | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Delivery assignment**                      | –             | –              | –                                                | –                                 | Scoped          | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Delivery batch management**                | –             | –              | –                                                | –                                 | Scoped          | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Delivery execution (pickup, PIN, COD)**    | –             | Own            | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Delivery evidence safety review**          | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Agent cash handover**                      | –             | Own            | –                                                | –                                 | –               | Scoped (receive)                              | –                                    | –                       | –                    | Full                                            |
| **Delivery PIN fallback (regenerate)**       | –             | –              | Scoped (with explicit permission)                | –                                 | –               | Scoped (with explicit permission)             | Scoped (with explicit permission)    | –                       | –                    | Full                                            |
| **Inventory viewing**                        | –             | –              | –                                                | Scoped                            | –               | Scoped                                        | Scoped                               | Read (assigned outlets) | –                    | Full                                            |
| **Inventory adjustments (submit)**           | –             | –              | –                                                | Scoped (Request)                  | –               | Scoped (Policy-limited post; above = Request) | –                                    | –                       | –                    | Full                                            |
| **Inventory adjustments (approve)**          | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Outlet-to-outlet transfers**               | –             | –              | –                                                | Scoped (Request)                  | –               | Scoped (Request/approve/receive)              | –                                    | –                       | –                    | Full                                            |
| **Pick reversal confirmation**               | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Returned cylinder intake**                 | –             | –              | –                                                | Scoped                            | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Post-delivery return intake**              | –             | –              | –                                                | Scoped (with explicit permission) | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Vendor refill batch management**           | –             | –              | –                                                | Scoped                            | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Outlet configuration & policies**          | –             | –              | –                                                | –                                 | –               | Read                                          | –                                    | Read (assigned outlets) | –                    | Full                                            |
| **Outlet price rules (within guardrail)**    | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Outlet price rules (above guardrail)**     | –             | –              | –                                                | –                                 | –               | Request                                       | –                                    | –                       | –                    | Approve / Full                                  |
| **Global pricing & catalog**                 | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Refund initiation**                        | Own (request) | –              | –                                                | –                                 | –               | Scoped (Threshold)                            | Request                              | –                       | –                    | Full                                            |
| **Refund approval**                          | –             | –              | –                                                | –                                 | –               | Scoped (Threshold)                            | –                                    | –                       | –                    | Full                                            |
| **Refund payout (cash at outlet)**           | –             | –              | Scoped (with explicit permission; payout limits) | –                                 | –               | Scoped (within payout limits)                 | –                                    | –                       | –                    | Full                                            |
| **Refund collection code management**        | –             | –              | –                                                | –                                 | –               | –                                             | Scoped (with explicit permission)    | –                       | –                    | Full                                            |
| **Daily cash closing**                       | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Financial ledger (view)**                  | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | Read (assigned outlets) | Full                 | Full                                            |
| **Internal settlement management**           | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | Acknowledge / Offset | Full                                            |
| **Expense submission**                       | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Expense approval**                         | –             | –              | –                                                | –                                 | –               | Scoped (Threshold)                            | –                                    | –                       | –                    | Full                                            |
| **Support case creation**                    | –             | –              | –                                                | –                                 | –               | Scoped                                        | Scoped                               | –                       | –                    | Full                                            |
| **Support case management**                  | –             | –              | –                                                | –                                 | –               | Scoped                                        | Scoped                               | –                       | –                    | Full                                            |
| **Support queue configuration**              | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Notification template administration**     | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Support communication logging**            | –             | –              | –                                                | –                                 | –               | Scoped                                        | Scoped                               | –                       | –                    | Full                                            |
| **Customer notification requests**           | –             | –              | –                                                | –                                 | –               | Scoped (approved transactional only)          | Scoped (approved transactional only) | –                       | –                    | Full                                            |
| **Support action requests (create/request)** | –             | –              | –                                                | –                                 | –               | Scoped                                        | Scoped                               | –                       | –                    | Full                                            |
| **Support action requests (execute)**        | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | –                       | –                    | Full                                            |
| **Audit log viewing**                        | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | Read (assigned outlets) | Read                 | Full                                            |
| **Operational risk alerts**                  | –             | –              | –                                                | –                                 | –               | Scoped (with explicit permission)             | –                                    | –                       | –                    | Full                                            |
| **Low-stock alerts**                         | –             | –              | –                                                | –                                 | –               | Scoped                                        | –                                    | Read (assigned outlets) | –                    | Full                                            |
| **Order reassignment (escalated)**           | –             | –              | –                                                | –                                 | –               | Scoped (exception)                            | –                                    | –                       | –                    | Full                                            |
| **Forced financial closure**                 | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Cross-outlet reporting**                   | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | Read (assigned outlets) | Read                 | Full                                            |
| **User & role management**                   | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full                                            |
| **Authorization policy management**          | –             | –              | –                                                | –                                 | –               | –                                             | –                                    | –                       | –                    | Full (with dual approval for sensitive changes) |

### Matrix Scope Notes

- **"Scoped"** means the persona's assigned outlet(s) only. A persona assigned to Outlet A has no visibility into Outlet
  B unless explicitly granted additional access by Super Admin. Area Managers have read access to their assigned outlet
  set, not all outlets. Customer Support Agents are outlet-scoped by default; cross-outlet access requires explicit
  Super Admin grant.

- **Support action request boundary**: Creating or requesting support action requests belongs to scoped support case
  management. The matrix's "execute" row means the authorized owner of the affected business workflow may accept or
  complete the requested action; it does not let Customer Support Agents directly approve refunds, pay refunds, post
  ledger entries, mutate orders, adjust inventory, or complete delivery workflows.

- **Inventory adjustment approval threshold**: At launch, inventory adjustment authority is policy-driven. Outlet
  Managers may post without separate Super Admin approval only single-unit damage or quarantine adjustments that do not
  increase available stock and carry a source reference (for example, a count, delivery, or order). This single-unit
  rule is the launch default Outlet Manager posting threshold; no higher Outlet Manager quantity threshold exists at
  launch. Every loss, missing-source adjustment, manual correction, positive available-stock increase, count variance,
  or absolute delta greater than one unit is `PENDING_APPROVAL` and requires Super Admin approval before the ledger
  movement is posted. Every adjustment requires reason, note, active policy code, ledger correlation when posted, and
  audit trail.

- **Inventory reconciliation paths**: Inventory reconciliation supports both quick day-to-day adjustments with reason
  codes and periodic physical counts with variance reports. Both paths create audited inventory adjustments and follow
  the active approval threshold before stock is recognized as changed.

- **Refund payout — Cashier permission**: An Outlet Cashier (P-03) with explicit refund-payout permission may verify the
  customer's collection code and disburse cash only within the active refund payout limits. Launch default cashier
  payout limits are UGX 100,000 per refund and UGX 300,000 per outlet business day. Launch default Outlet Manager payout
  limits are UGX 500,000 per refund and UGX 1,500,000 per outlet business day. A payout above the Outlet Manager limit
  requires Super Admin release approval before cash is disbursed. Without that explicit per-outlet permission, cashiers
  cannot handle refund payouts.

- **Refund approval thresholds**: Refund liabilities below the approval-required threshold become collectible without
  separate approval. At launch, the approval-required threshold is UGX 50,000: refund liabilities below UGX 50,000
  become collectible without separate approval, and refund liabilities at or above UGX 50,000 require approval before a
  collection code is issued. The active refund approval policy starts from global defaults and may define
  outlet/refund-reason overrides. At launch, Outlet Managers may approve refund liabilities from UGX 50,000 through UGX
  500,000 within their outlet scope; above UGX 500,000, Super Admin approval is required. Approval makes the liability
  collectible; payout is the separate cash-disbursement event.

- **Refund collection code management**: This permission covers regenerating expired codes, regenerating codes when the
  customer loses access after verification through a linked support case with reason and audit, unlocking or
  regenerating rate-limited codes, and audited reveal by a permissioned Customer Support Agent or Super Admin after
  customer verification. It does not allow approving, paying, voiding, or writing off a refund liability.

- **Internal settlement authority**: A Finance Officer (P-09) may acknowledge, query, and offset internal settlements.
  Voiding a settlement requires Super Admin approval with an explicit reason and audit record. Finance Officer cannot
  void unilaterally.

- **Outlet configuration vs price rules**: "Outlet configuration & policies" covers settings that only Super Admin may
  change: service zone, vendor acceptance list, delivery mode support, operating-hours policy, payment method support,
  workload/capacity limits, max batch size, and outlet priority score. "Outlet price rules (within guardrail)" is a
  distinct sub-domain covering outlet-scoped product/refill/accessory prices, delivery-fee overrides, and express-fee
  multipliers. Outlet Managers have write access to price rules within configured guardrails; they do not have write
  access to broader outlet configuration.

- **Payment account administration**: Creating, changing, deactivating, or setting the default mobile-money merchant
  account requires explicit payment-account administration permission. Writes require an audit reason. Ordinary reads
  expose only masked merchant-account details. Outlet/default constraints remain per outlet/provider, and account
  changes do not alter already-recorded payment attribution.

- **Order reassignment (escalated)**: Outlet Managers may handle post-picking reassignment exceptions only for orders in
  their outlet scope. Super Admin retains full authority for cross-outlet, exhausted-candidate, and global exception
  paths.

---

## Authentication & Authorization Model (§4)

Authentication establishes the human user's identity, but it does not determine outlet scope or business authority.

**At launch:**

- All human users except the root/bootstrap Super Admin enter the platform by phone-number signup. Phone control is
  verified through SMS OTP, the Identity account is created or reused, and the customer account/profile is ensured
  before customer features are available.
- Customer-only accounts authenticate with SMS OTP. Email OTP is not active at launch, and phone control is verified at
  registration/login.
- Privileged accounts (staff, delivery, support, finance, managers, and Super Admins) authenticate with
  username/password plus MFA. When any privileged grant or privileged scope is active or pending activation, customer
  SMS OTP is no longer an allowed login method for that account. The user must complete the required password and MFA
  setup or recovery path before authenticating.
- Privileged authentication does not remove customer access. After a privileged account successfully authenticates with
  username/password plus MFA, the authenticated context may include customer capabilities as well as any privileged
  capabilities allowed by explicit permissions, outlet scopes, policy, and session assurance.
- Any user may submit a generic unauthenticated recovery request for "I need recovery" using their login alias and safe
  contact context. That request does not prove identity, disclose account existence or status, authenticate the user, or
  change credentials. Privileged account recovery, lost phone, password reset, and MFA reset remain Super Admin-mediated
  only.
- **At launch, Super Admin recovery** means recovery of an account with Super Admin authority and requires break-glass
  dual approval by two distinct active Super Admins: one initiating or performing the recovery and a second
  non-requesting Super Admin co-approver. The recovered account subject cannot approve the recovery. No single-actor
  Super Admin recovery path exists at launch. The recovery record must identify the subject account, affected credential
  or recovery factor, initiating actor, co-approving actor, reason code, note, timestamp, before/after account-status
  and credential-readiness values, and confirmation that no secrets or payment credentials are stored.
- Normal checkout and late-payment-reference reuse do not require an extra OTP beyond the authenticated customer
  session.
- Credential readiness is a business access gate. Accounts requiring password setup, MFA setup, or recovery cannot
  exercise normal business permissions until the required step is completed; disabled accounts cannot authenticate or
  access any experience. If account-status or credential-readiness facts conflict, access fails closed until corrected
  through an audited path.

**Authorization model:**

- Authorization is determined by explicit permissions and scope assignments managed by the platform. A single human
  account may hold multiple permission bundles across one or more outlets.
- Login itself does not let the user choose a persona or experience as a source of authority. Experiences such as
  customer, delivery, staff, or admin are derived from those permissions; they are not separate account types.
- After authentication, the available experiences are limited to the account's permissions and current session
  assurance, and selecting an experience never grants new permissions, outlet scope, or business authority.
- Outlet rosters, work queues, case queues, and other operational projections may organize work or explain visibility,
  but they are not authorization sources and cannot grant permission or outlet scope beyond the explicit assignments.
- Only the latest approved and valid authorization policy may grant business access. Draft, invalid, failed, or
  unapproved authorization-policy changes cannot expand access; when authoritative permission or scope facts conflict,
  access fails closed until corrected through an audited approval path.
- Authorization denials and sensitive successful grants are audit-relevant and must identify the active authorization
  policy basis used for the decision. Ordinary successful reads are not individually audit-logged unless this document
  classifies the read as sensitive.

**Launch governance:**

- Launch-scope acceptance, dependency-gate completion, and launch approval are business-governance decisions outside the
  runtime persona model. The Product Manager is the sole formal approval authority for those decisions unless the
  Product Manager explicitly adds another approval authority to launch scope.
- Engineering, QA, support, operations, and platform personas may provide evidence, risk notes, recommendations, and
  estimates, but those inputs do not create independent launch-blocking authority.
- Operational-readiness recommendations do not create separate formal launch blockers unless the Product Manager accepts
  them into launch scope.
- Launch scope remains open until Product Manager launch approval. When the Product Manager accepts a new launch-scope
  requirement, the acceptance record must identify the affected business workflows, any audit or reporting behavior
  affected, evidence reviewed, accepted residual risk, and launch timing impact.
- Hands-on trials using real business operations are not allowed until the complete launch system is ready for
  production use. No launch-scope workflow is considered operationally usable in isolation before the complete launch
  system is production-ready. Scripted non-production sign-off may occur earlier, but it does not authorize real
  operational use.

---

## Authorization Edge Cases (E-01–E-10)

**E-01**: A payment-verification actor cannot verify a mobile-money payment for an order assigned to a different outlet,
even if both outlets are in the same city.

**E-02**: A Delivery Agent can see the customer's phone number only while an order is assigned to them and active. They
lose this access when the delivery reaches a terminal state. Phone-number access is scoped to the active assignment and
is audit-sensitive under the active audit policy.

**E-03**: Customer Support Agents cannot view another outlet's cases unless a Super Admin has granted explicit
cross-outlet support access. There is no implicit cross-outlet access based on case type or priority.

**E-04**: An Area Manager can view reports and operational data for their assigned outlets, but cannot view operational
risk alerts or perform any action (accept order, verify payment, approve refund, adjust inventory).

**E-05**: Refund liabilities at or above the approval-required threshold require approval from the Outlet Manager for
that outlet within their active approval threshold, or from a Super Admin above that threshold. At launch, the
approval-required threshold is UGX 50,000 and the Outlet Manager approval threshold is UGX 500,000 within outlet scope.
The approving actor cannot approve a refund on an order where they submitted the refund request themselves.

**E-06**: A Super Admin performing a forced financial closure must provide a reason. This action is always audit-logged
with before/after values. No exception to audit logging exists, even for Super Admin.

**E-07**: An Outlet Manager can approve refunds for their own outlet within their authorized threshold and approval
policy. At launch, Outlet Managers may approve refund liabilities from UGX 50,000 through UGX 500,000 within their
outlet scope; refunds above that threshold require Super Admin approval. Outlet Managers cannot approve refunds at
another outlet, and they cannot approve their own submitted refund request.

**E-08**: A Dispatcher can reassign delivery agents to orders within their outlet until the point of pickup with a
recorded reason. After pickup, only an Outlet Manager or Super Admin can reassign, and the action requires an explicit
reason code.

**E-09**: A single human account may hold both Outlet Cashier and Inventory Clerk permissions at the same outlet only
when the active authorization policy permits that combination. The combination must be explicit, recorded, and subject
to audit on sensitive actions; the actor still cannot approve their own submissions or bypass normal approval
thresholds.

**E-10**: Authorization policy changes that affect financial behavior, reporting, audit semantics, user-visible
functionality, or launch scope require Product Manager approval and co-approval under the active authorization-change
policy; the Super Admin proposing the change cannot satisfy that co-approval alone. Product Manager approval is
business-governance authority, not runtime platform permission.
