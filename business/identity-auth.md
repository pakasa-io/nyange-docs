# Identity & Authorization

**Intent**: Define launch personas, the canonical access matrix,
authentication rules, authorization rules, and authorization edge cases.

**Reader task**: Use this document to decide whether an actor can authenticate,
which permissions and outlet scopes can authorize an action, and which actor
combinations require audit or co-approval.

**Sources**: §2 Personas, §3 Access Matrix, §4 Auth Model, E-01-E-05,
E-07-E-10

## Invariants

**BI-16 — A persona cannot approve their own submission.**

- Any action requiring approval or co-approval cannot be approved by the same
  individual who submitted or requested it.
- This applies regardless of role.
- Covered examples include expense above threshold, inventory adjustment, price
  change above guardrail, and sensitive authorization-policy change.

## Terms

| Term | Meaning |
| --- | --- |
| Persona | Canonical business archetype used in this specification. A persona is not an account type. |
| Permission bundle | Explicit capability grant assigned to a human account. |
| Outlet scope | Outlet or outlet set within which a permission may be exercised. |
| Experience | Customer, delivery, staff, or admin UI context derived from permissions and session assurance. |
| Session assurance | Authentication strength and current credential readiness required for privileged access. |

## Personas

These ten personas are canonical business archetypes. Runtime authorization is
determined by explicit permissions and scope assignments. A single human account
may hold multiple permission bundles across one or more outlets.

### P-01 Customer

- Orders gas, refills, and accessories from a registered account.
- Logs in before cart or checkout at launch.
- Uses the authenticated customer experience to track status and access
  customer-visible notices.
- May have multiple saved addresses.
- Pays cash at the doorstep for online COD delivery.
- Collects cash refunds in person at the outlet.
- Cannot manage outlet operations, view other customers' data, access financial
  records, or override business rules or policy controls.

### P-02 Delivery Agent

- Field operative assigned to deliver orders from outlet to customer.
- Sees only assigned active deliveries and customer phone numbers for assigned
  active deliveries.
- Performs required field actions during active delivery.
- Collects COD cash, records returned cylinders, reports failures, and submits
  cash handover.
- Cannot change outlet claims, view other agents' assignments, modify
  inventory, issue refunds, or access financial records.

### P-03 Outlet Cashier

- Outlet staff responsible for customer cash refund payouts when explicitly
  permissioned.
- Verifies customer-presented refund collection codes for collectible refunds.
- Disburses collectible cash refunds from the owning outlet.
- Has no delivery responsibilities.
- Cannot manage online orders, create refund liabilities, adjust inventory, or
  access another outlet's data.

### P-04 Inventory Clerk

- Outlet staff responsible for stock management.
- Manages stock counts, submits inventory adjustment requests, confirms
  returned-cylinder intake, initiates outlet transfer requests, and records
  vendor refill movements.
- May mark claimed orders ready for pickup when explicitly permissioned.
- Works within assigned outlet only.
- Cannot claim orders, collect customer cash, issue refunds, access
  financial records, or approve their own adjustment requests.

### P-05 Dispatcher

- Outlet staff responsible for delivery coordination.
- Assigns delivery agents to single-order delivery tasks.
- Tracks delivery progress.
- May change the assigned delivery agent before pickup within outlet active
  delivery policy.
- Cannot collect customer cash, pay refunds, adjust inventory, access
  financial records, or modify outlet policies.

### P-06 Outlet Manager

- Operational owner of a single outlet.
- Claims pending orders for the outlet when explicitly permissioned.
- Marks claimed orders ready for pickup.
- Reconciles daily cash.
- Initiates scoped refund liabilities and disburses collectible refunds when
  explicitly permissioned.
- Submits above-threshold inventory adjustments for approval.
- Oversees staff, manages delivery operations, and views outlet-specific
  reports.
- Has elevated authority over P-03, P-04, and P-05 within their outlet.
- Cannot access another outlet's data unless explicitly granted.
- Cannot change global prices or catalog, pay refunds outside outlet scope, or
  override Super Admin controls.

### P-07 Customer Support Agent

- Handles launch support fallback actions when explicitly permissioned.
- May perform refund collection-code regeneration, unlock, or audited
  customer-verified reveal when granted those permissions.
- May request approved transactional customer notifications when explicitly
  permissioned.
- May relay operational escalation needs to the owning Outlet Manager or Super
  Admin path.
- Cannot directly mutate orders, payments, inventory, or financial records
  outside explicitly permissioned fallback actions.
- Cannot create, pay, void, or write off refund liabilities.
- Cannot access another outlet's fallback actions or operational records unless
  explicitly granted cross-outlet access.

### P-08 Area Manager

- Regional oversight role assigned to multiple outlets.
- Monitors performance and escalates exceptions across assigned outlets.
- Has read access to outlet operations, inventory, and financial summaries for
  assigned outlets.
- Does not perform direct outlet operations.
- Cannot claim orders, collect cash, adjust inventory, view or manage
  operational risk alerts, access outlets outside assignment, or override Super
  Admin controls.

### P-09 Finance Officer

- Financial operations role with ledger visibility.
- Reviews financial ledger entries.
- Reviews daily closing summaries, expense reporting, and policy outcomes.
- Works across all outlets.
- Cannot mutate order, delivery, or inventory state.
- Cannot approve their own submitted records.
- Cannot manage catalog or pricing.

### P-10 Super Admin

- Global platform authority.
- Has full operational and configuration authority across all outlets, business
  domains, and records.
- Approves above-guardrail price changes, above-threshold inventory adjustments,
  break-glass authorization policy changes, and other actions no other persona
  can execute.
- Sensitive mutations and overrides require explicit reason codes and permanent
  audit logging.
- Cannot issue unaudited mutations.
- Super Admin authority extends audit requirements; it does not bypass them.

## Access Matrix

### Notation

| Symbol | Meaning |
| --- | --- |
| Full | Full read and write within scope |
| Scoped | Access limited to assigned outlet(s) |
| Read | Read-only |
| Own | Access to own records only |
| Request | Can initiate but not approve |
| Approve | Can approve requests from others; cannot approve own |
| Acknowledge | Can record review/acknowledgement of closing records without voiding them |
| Threshold | Authority limited by configured financial or quantity threshold |
| - | No access |

| Capability Domain | P-01 Customer | P-02 Agent | P-03 Cashier | P-04 Inv Clerk | P-05 Dispatcher | P-06 Outlet Mgr | P-07 Support | P-08 Area Mgr | P-09 Finance | P-10 Super Admin |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Own account & address | Full | - | - | - | - | - | - | - | - | Full |
| Address coordinate correction | Own map pin | - | - | - | - | - | - | - | - | Full |
| Product catalog browsing | Read | - | Read | - | - | Read | Read | Read | - | Full |
| Cart creation and update | Full | - | - | - | - | - | - | - | - | Full |
| Cart quote | Full | - | - | - | - | - | - | - | - | Full |
| Cart checkout | Full | - | - | - | - | - | - | - | - | Full |
| Order placement | Full | - | - | - | - | - | - | - | - | Full |
| Order status tracking | Own | Own assigned | Scoped | - | Scoped | Scoped | Scoped | Read assigned outlets | Read | Full |
| Outlet claiming | - | - | - | - | - | Scoped with explicit permission | - | - | - | Full |
| Ready-for-pickup marking | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | - | Full |
| Outlet handover confirmation | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | - | Full |
| Failed-order return receipt | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | - | Full |
| Returned-cylinder receipt | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | - | Full |
| Delivery assignment | - | - | - | - | Scoped | Scoped | - | - | - | Full |
| Delivery execution pickup and COD | - | Own | - | - | - | - | - | - | - | Full |
| Agent cash handover | - | Own | - | - | - | Scoped receive | - | - | - | Full |
| Inventory viewing | - | - | - | Scoped | - | Scoped | Scoped | Read assigned outlets | - | Full |
| Inventory adjustments submit | - | - | - | Scoped request | - | Scoped policy-limited post; above = request | - | - | - | Full |
| Inventory adjustments approve | - | - | - | - | - | - | - | - | - | Full |
| Outlet-to-outlet transfers | - | - | - | Scoped request | - | Scoped request/approve/receive | - | - | - | Full |
| Returned cylinder intake | - | - | - | Scoped | - | Scoped | - | - | - | Full |
| Vendor refill batch management | - | - | - | Scoped | - | Scoped | - | - | - | Full |
| Outlet configuration & policies | - | - | - | - | - | Read | - | Read assigned outlets | - | Full |
| Outlet price rules within guardrail | - | - | - | - | - | Scoped | - | - | - | Full |
| Outlet price rules above guardrail | - | - | - | - | - | Request | - | - | - | Approve / Full |
| Global pricing & catalog | - | - | - | - | - | - | - | - | - | Full |
| Refund initiation | Own request | - | - | - | - | Scoped | Request | - | - | Full |
| Refund payout cash at outlet | - | - | Scoped with explicit permission | - | - | Scoped | - | - | - | Full |
| Refund collection code management | - | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Daily cash closing | - | - | - | - | - | Scoped | - | - | - | Full |
| Financial ledger view | - | - | - | - | - | Scoped | - | Read assigned outlets | Full | Full |
| Expense submission | - | - | - | - | - | Scoped | - | - | - | Full |
| Expense approval | - | - | - | - | - | Scoped threshold | - | - | - | Full |
| Notification template administration | - | - | - | - | - | - | - | - | - | Full |
| Customer notification requests | - | - | - | - | - | Scoped approved transactional only | Scoped approved transactional only | - | - | Full |
| Audit log viewing | - | - | - | - | - | Scoped | - | Read assigned outlets | Read | Full |
| Operational risk alerts | - | - | - | - | - | Scoped with explicit permission | - | - | - | Full |
| Low-stock alerts | - | - | - | - | - | Scoped | - | Read assigned outlets | - | Full |
| Cross-outlet reporting | - | - | - | - | - | - | - | Read assigned outlets | Read | Full |
| User & role management | - | - | - | - | - | - | - | - | - | Full |
| Authorization policy management | - | - | - | - | - | - | - | - | - | Full with dual approval for sensitive changes |

## Matrix Scope Notes

### Scoped Access

- `Scoped` means the persona's assigned outlet(s) only.
- A persona assigned to Outlet A has no visibility into Outlet B unless Super
  Admin explicitly grants additional access.
- Area Managers have read access to their assigned outlet set, not all outlets.
- Customer Support Agent fallback actions are outlet-scoped by default.
- Cross-outlet support fallback access requires explicit Super Admin grant.

### Support Fallback Boundary

- Customer Support Agents can perform only explicitly permissioned fallback
  actions.
- The authorized owner of the affected domain workflow remains responsible for
  claiming, cancelling, or completing that workflow.
- Customer Support Agents do not directly create refund liabilities, pay
  refunds, post ledger entries, mutate orders, adjust inventory, or complete
  delivery workflows through a support-owned workflow.

### Inventory Adjustment Threshold

- At launch, inventory adjustment authority is policy-driven.
- Outlet Managers may post without separate Super Admin approval only
  single-unit damage or quarantine adjustments that do not increase available
  stock and carry a source reference, such as count, delivery, or order.
- This single-unit rule is the launch default Outlet Manager posting threshold.
- No higher Outlet Manager quantity threshold exists at launch.
- Every loss, missing-source adjustment, manual correction, positive
  available-stock increase, count variance, or absolute delta greater than one
  unit is `PENDING_APPROVAL`.
- `PENDING_APPROVAL` inventory adjustments require Super Admin approval before
  ledger movement is posted.
- Every adjustment requires reason, note, active policy code, ledger correlation
  when posted, and audit trail.

### Inventory Reconciliation

- Inventory reconciliation supports quick day-to-day adjustments with reason
  codes.
- Inventory reconciliation supports periodic physical counts with variance
  reports.
- Both paths create audited inventory adjustments and follow the active approval
  threshold before stock is recognized as changed.

### Refund Payout Permission

- An Outlet Cashier with explicit refund-payout permission may verify the
  customer's collection code and disburse collectible refunds within their
  outlet scope.
- Outlet Managers may disburse collectible refunds within outlet scope.
- Launch refund payouts have no per-refund or per-outlet-business-day cash cap.
- Without explicit per-outlet permission, cashiers cannot handle refund payouts.

### Refund Source and Payout Boundary

- Refund liabilities have no amount-based approval threshold at launch.
- A valid authorized and posted source correction, adjustment, or other
  launch-approved source event may create a refund liability eligible for
  collection-code issuance regardless of amount.
- Source authorization remains owned by the source workflow that creates the
  refund liability.
- Payout is the separate cash-disbursement event.
- Payout permission does not authorize creating, voiding, or writing off a
  refund liability.

### Refund Collection Code Management

- This permission covers regenerating expired codes.
- It covers regenerating codes when the customer loses access after verification
  through an audited fallback-action record with reason and audit.
- It covers unlocking or regenerating rate-limited codes.
- It covers audited reveal by a permissioned Customer Support Agent or Super
  Admin after customer verification.
- It does not allow creating, paying, voiding, or writing off a refund
  liability.

### Outlet Configuration vs. Price Rules

- `Outlet configuration & policies` covers settings that only Super Admin may
  change: service zone, vendor acceptance list, delivery mode support,
  operating-hours policy, workload/capacity limits, and outlet priority score.
- `Outlet price rules within guardrail` covers outlet-scoped product, refill,
  accessory prices, and delivery-fee overrides.
- Outlet Managers have write access to price rules within configured guardrails.
- Outlet Managers do not have write access to broader outlet configuration.

## Authentication Model

Authentication establishes the human user's identity. It does not determine
outlet scope or business authority.

### Launch Authentication Rules

- All human users except the root/bootstrap Super Admin enter the platform by
  phone-number signup.
- Phone control is verified through SMS OTP.
- The Identity account is created or reused.
- The customer account/profile is ensured before customer features are
  available.
- Customer-only accounts authenticate with SMS OTP.
- Email OTP is not active at launch.
- Phone control is verified at registration/login.
- Privileged accounts authenticate with username/password plus MFA.
- Privileged accounts include staff, delivery, support, finance, managers, and
  Super Admins.
- When any privileged grant or privileged scope is active or pending activation,
  customer SMS OTP is no longer an allowed login method for that account.
- The user must complete the required password and MFA setup or recovery path
  before authenticating.
- Privileged authentication does not remove customer access.
- After privileged authentication succeeds, the authenticated context may include
  customer capabilities and privileged capabilities allowed by explicit
  permissions, outlet scopes, policy, and session assurance.
- Normal checkout does not require extra OTP beyond the authenticated customer
  session.

### Recovery

- Any user may submit a generic unauthenticated recovery request using login
  alias and safe contact context.
- The generic recovery request does not prove identity.
- It does not disclose account existence or status.
- It does not authenticate the user.
- It does not change credentials.
- Privileged account recovery, lost phone, password reset, and MFA reset remain
  Super Admin-mediated only.
- At launch, Super Admin recovery means recovery of an account with Super Admin
  authority.
- Super Admin recovery requires break-glass dual approval by two distinct active
  Super Admins: one initiating or performing recovery and a second
  non-requesting co-approver.
- The recovered account subject cannot approve the recovery.
- No single-actor Super Admin recovery path exists at launch.
- The recovery record must identify the subject account, affected credential or
  recovery factor, initiating actor, co-approving actor, reason code, note,
  timestamp, before/after account-status and credential-readiness values, and
  confirmation that no authentication secrets are stored.

### Credential Readiness

- Credential readiness is a business access gate.
- Accounts requiring password setup, MFA setup, or recovery cannot exercise
  normal business permissions until the required step is complete.
- Disabled accounts cannot authenticate or access any experience.
- If account-status or credential-readiness facts conflict, access fails closed
  until corrected through an audited path.

## Authorization Model

- Authorization is determined by explicit permissions and scope assignments
  managed by the platform.
- A single human account may hold multiple permission bundles across one or more
  outlets.
- Login itself does not let the user choose a persona or experience as a source
  of authority.
- Experiences are derived from permissions and current session assurance.
- Experiences are not separate account types.
- Selecting an experience never grants new permissions, outlet scope, or
  business authority.
- Outlet rosters, work queues, and operational projections may organize work or
  explain visibility.
- Those projections are not authorization sources and cannot grant permission or
  outlet scope beyond explicit assignments.
- Only the latest approved and valid authorization policy may grant business
  access.
- Draft, invalid, failed, or unapproved authorization-policy changes cannot
  expand access.
- When authoritative permission or scope facts conflict, access fails closed
  until corrected through an audited approval path.
- Authorization denials and sensitive successful grants are audit-relevant.
- Audit-relevant authorization decisions must identify the active authorization
  policy basis used for the decision.
- Ordinary successful reads are not individually audit-logged unless this
  document classifies the read as sensitive.

## Launch Governance

- Launch-scope acceptance, dependency-gate completion, and launch approval are
  business-governance decisions outside the runtime persona model.
- The Product Manager is the sole formal approval authority for those decisions
  unless the Product Manager explicitly adds another approval authority to
  launch scope.
- Engineering, QA, support, operations, and platform personas may provide
  evidence, risk notes, recommendations, and estimates.
- Those inputs do not create independent launch-blocking authority.
- Operational-readiness recommendations do not create separate formal launch
  blockers unless the Product Manager accepts them into launch scope.
- Launch scope remains open until Product Manager launch approval.
- When the Product Manager accepts a new launch-scope requirement, the
  acceptance record must identify affected business workflows, affected audit or
  reporting behavior, evidence reviewed, accepted residual risk, and launch
  timing impact.
- Hands-on trials using real business operations are not allowed until the
  complete launch system is ready for production use.
- No launch-scope workflow is considered operationally usable in isolation
  before the complete launch system is production-ready.
- Scripted non-production sign-off may occur earlier, but it does not authorize
  real operational use.

## Authorization Edge Cases

**E-01**: No launch actor can submit, verify, reuse, or administer an external
payment reference. External electronic payment rails are outside launch scope
and require explicit scope re-entry before any related permission can be
granted.

**E-02**: A Delivery Agent can see the customer's phone number only while an
order is assigned to them and active. They lose this access when the delivery
reaches a terminal state. Phone-number access is scoped to the active assignment
and is audit-sensitive under the active audit policy.

**E-03**: Customer Support Agents cannot access another outlet's fallback
actions or operational records unless a Super Admin has granted explicit
cross-outlet support access. There is no implicit cross-outlet access based on
issue type, priority, or customer complaint.

**E-04**: An Area Manager can view reports and operational data for assigned
outlets, but cannot view operational risk alerts or perform outlet actions such
as claiming orders, collecting customer cash, paying refunds, or adjusting
inventory.

**E-05**: Refund liabilities have no amount-based approval threshold at launch.
A valid authorized and posted source event may create a refund liability
eligible for collection-code issuance regardless of amount. The source workflow
still owns any required source authorization and separation-of-duty rule.

**E-07**: An Outlet Manager may initiate scoped refund liabilities and disburse
collectible refunds within outlet scope when explicitly permissioned. Outlet
Managers cannot pay refunds for another outlet, void or write off refund
liabilities, bypass source-workflow authorization, or approve their own source
submission when the source workflow requires approval.

**E-08**: A Dispatcher can change the assigned delivery agent within their
outlet until pickup with a recorded reason. After pickup, no normal reassignment
is allowed. A Super Admin may record a manual custody exception for an abnormal
after-pickup custody issue, with reason, note, known goods/cash status, and audit.

**E-09**: A single human account may hold both Outlet Cashier and Inventory Clerk
permissions at the same outlet only when the active authorization policy permits
that combination. The combination must be explicit, recorded, and subject to
audit on sensitive actions. The actor still cannot approve their own submissions
or bypass normal approval thresholds.

**E-10**: Authorization policy changes that affect financial behavior,
reporting, audit semantics, user-visible functionality, or launch scope require
Product Manager approval and co-approval under the active authorization-change
policy. The Super Admin proposing the change cannot satisfy that co-approval
alone. Product Manager approval is business-governance authority, not runtime
platform permission.
