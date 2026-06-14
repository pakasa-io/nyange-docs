# Identity & Authorization

**Intent**: Define launch personas, the canonical access matrix,
authentication rules, authorization rules, and authorization edge cases.

**Reader task**: Use this document to decide whether an actor can authenticate,
which permissions and outlet scopes can authorize an action, and which actor
combinations require audit.

**Sources**: §2 Personas, §3 Access Matrix, §4 Auth Model, E-02, E-04-E-05,
E-07-E-09

## Invariants

**BI-16 — A persona cannot approve their own submission.**

- Any action requiring approval cannot be approved by the same individual who
  submitted or requested it.
- This applies regardless of role.
- Covered examples include expense above threshold and inventory adjustment.

## Terms

| Term | Meaning |
| --- | --- |
| Persona | Canonical business archetype used in this specification. A persona is not an account type. |
| Permission bundle | Explicit capability grant assigned to a human account. |
| Outlet scope | Outlet or outlet set within which a permission may be exercised. |
| Experience | Customer, delivery, staff, or admin UI context derived from permissions and session assurance. |
| Session assurance | Authentication strength and current credential readiness required for privileged access. |

## Personas

These nine personas are canonical business archetypes. Runtime authorization is
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
- Reviews stock records, submits inventory adjustment requests, confirms
  returned-cylinder intake, initiates outlet transfer requests, and records
  vendor refill movements.
- May mark claimed orders ready for pickup when explicitly permissioned.
- Works within assigned outlet only.
- Cannot claim orders, collect customer cash, issue refunds, access
  financial records, or approve their own adjustment requests.

### P-05 Dispatcher

- Outlet staff responsible for delivery coordination.
- Manually assigns delivery agents to single-order delivery tasks.
- Tracks delivery progress.
- May change the assigned delivery agent before pickup within outlet active
  delivery policy.
- Cannot collect customer cash, pay refunds, adjust inventory, access
  financial records, or modify outlet policies.

### P-06 Outlet Manager

- Operational owner of a single outlet.
- Claims pending orders for the outlet when explicitly permissioned.
- Reopens scoped claim-blocked orders to the pending pool when explicitly
  permissioned and Order confirms a current eligible claim path.
- Marks claimed orders ready for pickup.
- Reconciles daily cash.
- Initiates scoped refund liabilities.
- Disburses collectible refunds only when explicitly granted refund-payout
  permission.
- Submits above-threshold inventory adjustments for approval.
- Oversees staff, manages delivery operations, and views outlet-specific
  reports.
- Has elevated authority over P-03, P-04, and P-05 within their outlet.
- Cannot access another outlet's data unless explicitly granted.
- Cannot change global prices or catalog, pay refunds outside outlet scope, or
  override Super Admin controls.

### P-08 Area Manager

- Regional oversight role assigned to multiple outlets.
- Has read access to outlet operations, inventory, and financial summaries for
  assigned outlets.
- Has read access to claim-block exceptions for assigned outlets.
- Does not perform direct outlet operations.
- Cannot claim orders, collect cash, adjust inventory, access outlets outside
  assignment, close orders as unclaimable, or override Super Admin controls.

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
- Manages global pricing, approves above-threshold inventory adjustments,
  manages authorization policy changes, and other actions no other persona can
  execute.
- Closes claim-blocked orders as unclaimable when Order-owned policy evidence
  supports terminal closure.
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

| Capability Domain | P-01 Customer | P-02 Agent | P-03 Cashier | P-04 Inv Clerk | P-05 Dispatcher | P-06 Outlet Mgr | P-08 Area Mgr | P-09 Finance | P-10 Super Admin |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Own account & address | Full | - | - | - | - | - | - | - | Full |
| Address coordinate correction | Own map pin | - | - | - | - | - | - | - | Full |
| Product catalog browsing | Read | - | Read | - | - | Read | Read | - | Full |
| Cart creation and update | Full | - | - | - | - | - | - | - | Full |
| Cart quote | Full | - | - | - | - | - | - | - | Full |
| Cart checkout | Full | - | - | - | - | - | - | - | Full |
| Cart cleanup policy administration | - | - | - | - | - | - | - | - | Full |
| Order placement | Full | - | - | - | - | - | - | - | Full |
| Order status tracking | Own | Own assigned | Scoped | - | Scoped | Scoped | Read assigned outlets | Read | Full |
| Outlet claiming | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Claim-block resolution | - | - | - | - | - | Scoped reopen with explicit permission | Read assigned outlets | - | Full |
| Unclaimable closure | - | - | - | - | - | - | - | - | Full |
| Ready-for-pickup marking | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Cancellation | Own before pickup | - | - | - | - | Scoped outlet claim cancellation | - | - | Full |
| Outlet handover confirmation | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Failed-order return receipt | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Returned-cylinder receipt | - | - | - | Scoped with explicit permission | - | Scoped with explicit permission | - | - | Full |
| Delivery assignment | - | - | - | - | Scoped | Scoped | - | - | Full |
| Delivery completion | - | Own assigned | - | - | - | - | - | - | Full |
| COD recording | - | Own assigned | - | - | - | - | - | - | Full |
| Delivery execution pickup and COD | - | Own | - | - | - | - | - | - | Full |
| Agent cash handover | - | Own | - | - | - | Scoped receive | - | - | Full |
| Inventory viewing | - | - | - | Scoped | - | Scoped | Read assigned outlets | - | Full |
| Reservation from order claim | - | - | - | - | - | Scoped with explicit permission | - | - | Full |
| Inventory adjustments submit | - | - | - | Scoped request | - | Scoped policy-limited post; above = request | - | - | Full |
| Inventory adjustments approve | - | - | - | - | - | - | - | - | Full |
| Outlet-to-outlet transfers | - | - | - | Scoped request | - | Scoped request/approve/receive | - | - | Full |
| Returned cylinder intake | - | - | - | Scoped | - | Scoped | - | - | Full |
| Vendor refill batch management | - | - | - | Scoped | - | Scoped | - | - | Full |
| Outlet configuration & policies | - | - | - | - | - | Read | Read assigned outlets | - | Full |
| Global pricing & catalog | - | - | - | - | - | - | - | - | Full |
| Refund initiation | Own request | - | - | - | - | Scoped | - | - | Full |
| Refund payout cash at outlet | - | - | Scoped with explicit permission | - | - | Scoped with explicit permission | - | - | Full |
| Daily cash closing | - | - | - | - | - | Scoped | - | - | Full |
| Financial ledger view | - | - | - | - | - | Scoped | Read assigned outlets | Full | Full |
| Expense submission | - | - | - | - | - | Scoped | - | - | Full |
| Expense approval | - | - | - | - | - | Scoped threshold | - | - | Full |
| Notification template administration | - | - | - | - | - | - | - | - | Full |
| Customer notification requests | - | - | - | - | - | Scoped approved transactional only | - | - | Full |
| Audit log viewing | - | - | - | - | - | Scoped | Read assigned outlets | Read | Full |
| Cross-outlet reporting | - | - | - | - | - | - | Read assigned outlets | Read | Full |
| User & role management | - | - | - | - | - | - | - | - | Full |
| Authorization policy management | - | - | - | - | - | - | - | - | Full |

## Matrix Scope Notes

### Scoped Access

- `Scoped` means the persona's assigned outlet(s) only.
- A persona assigned to Outlet A has no visibility into Outlet B unless Super
  Admin explicitly grants additional access.
- Area Managers have read access to their assigned outlet set, not all outlets.

### Inventory Adjustment Permission

- Identity owns inventory adjustment permissions and outlet scope.
- [inventory.md](inventory.md) owns inventory adjustment thresholds, approval
  outcomes, and ledger-posting rules.
- Inventory Clerks may submit scoped inventory adjustment requests.
- Outlet Managers may post only policy-limited scoped adjustments allowed by
  Inventory, and must submit above-policy adjustments for Super Admin approval.
- Super Admins may approve and post inventory adjustments across all outlets.
- Permission to submit, post, or approve an adjustment does not bypass
  Inventory-owned source-reference, reason, note, policy-code, ledger
  correlation, or audit requirements.

### Refund Payout Permission

- An Outlet Cashier with explicit refund-payout permission may verify the
  customer's collection code and disburse collectible refunds within their
  outlet scope.
- An Outlet Manager with explicit refund-payout permission may verify the
  customer's collection code and disburse collectible refunds within their
  outlet scope.
- Launch refund payouts have no per-refund or per-outlet-business-day cash cap.
- Without explicit per-outlet refund-payout permission, Cashiers and Outlet
  Managers cannot handle refund payouts.

### Refund Source and Payout Boundary

- Refund liabilities have no amount-based approval threshold at launch.
- The supported launch refund-liability source is an authorized and posted
  Finance-owned cash over-collection correction.
- Finance owns source authorization for cash over-collection correction.
- Additional refund-liability source workflows require explicit launch-scope
  entry with a named source owner before they can create Refund liabilities.
- Payout is the separate cash-disbursement event.
- Payout permission does not authorize creating, voiding, or writing off a
  refund liability.

### Outlet Configuration

- `Outlet configuration & policies` covers settings that only Super Admin may
  change: service zone, vendor acceptance list, delivery mode support,
  operating-hours policy, workload/capacity limits, and outlet priority score.
- Outlet configuration does not include product prices, refill prices,
  accessory prices, tax rules, delivery-fee rules, or outlet-local pricing
  guardrails.
- Outlet Managers do not have write access to outlet configuration.

### Pricing Permissions

- Identity owns pricing-related permission grants and outlet scope.
- [catalog.md](catalog.md) owns global catalog, product, refill, accessory, tax,
  and launch pricing administration rules.
- [delivery.md](delivery.md) owns delivery-fee calculation rules.
- At launch, `Global pricing & catalog` is Super Admin-only.
- No outlet-scoped delivery-fee override permission exists at launch.
- No Outlet Manager, Dispatcher, Delivery Agent, Area Manager, or other
  outlet-scoped actor may create, change, approve, or apply outlet-local
  delivery-fee rules.
- Outlet-local product, refill, accessory, and delivery-fee price permissions are
  deferred in
  [../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md](../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md).

### Claim-Block Permissions

- Identity owns who may view, reopen, or close claim-blocked orders.
- [order.md](order.md) owns claim-block eligibility, timeout policy,
  claim-blocking reason codes, reopen requirements, terminal `UNCLAIMABLE`
  rules, and audit fields.
- Claim-block resolution permission allows reopening `CLAIM_BLOCKED -> PENDING`
  only under Order-owned policy.
- Scoped claim-block resolution is limited to orders whose current
  claim-evaluation candidate set includes one of the actor's assigned outlets.
- Claim-block resolution permission does not authorize changing the frozen order
  snapshot, manually assigning an outlet, bypassing stock reservation, or
  closing the order as `UNCLAIMABLE`.
- `UNCLAIMABLE` closure is Super Admin-only at launch.

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
- Privileged accounts include staff, delivery, finance, managers, and Super
  Admins.
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
- Super Admin recovery requires two distinct active Super Admin actors: one
  initiating or performing recovery and a second non-requesting confirming
  actor.
- The recovered account subject cannot approve the recovery.
- No single-actor Super Admin recovery path exists at launch.
- The recovery record must identify the subject account, affected credential or
  recovery factor, initiating actor, confirming actor, reason code, note,
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
- Engineering, QA, operations, and platform personas may provide evidence, risk
  notes, recommendations, and estimates.
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

**E-02**: A Delivery Agent can see the customer's phone number only while an
order is assigned to them and active. They lose this access when the delivery
reaches a terminal state. Phone-number access is scoped to the active assignment
and is audit-sensitive under the active audit policy.

**E-04**: An Area Manager can view reports and operational data for assigned
outlets, but cannot perform outlet actions such as claiming orders, collecting
customer cash, paying refunds, or adjusting inventory.

**E-05**: Refund liabilities have no amount-based approval threshold at launch.
A valid authorized and posted Finance-owned cash over-collection correction may
create a refund liability eligible for collection-code issuance regardless of
amount. Finance owns source authorization and any required separation-of-duty
rule for the source correction.

**E-07**: An Outlet Manager may initiate scoped refund liabilities. An Outlet
Manager may disburse collectible refunds within outlet scope only when
explicitly granted refund-payout permission. Outlet Managers cannot pay refunds
for another outlet, void or write off refund liabilities, bypass Finance source
authorization, or approve their own Finance source correction when source
approval is required.

**E-08**: A Dispatcher can change the assigned delivery agent within their
outlet until pickup with a recorded reason. After pickup, no normal reassignment
is allowed. A Super Admin may record a manual custody exception for an abnormal
after-pickup custody issue, with reason, note, known goods/cash status, and audit.

**E-09**: A single human account may hold both Outlet Cashier and Inventory Clerk
permissions at the same outlet only when the active authorization policy permits
that combination. The combination must be explicit, recorded, and subject to
audit on sensitive actions. The actor still cannot approve their own submissions
or bypass normal approval thresholds.
