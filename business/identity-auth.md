# Identity & Authorization Facts

**Intent**: Define launch personas, authentication rules, account state,
authentication-link facts, permission-grant facts, outlet-scope facts, and
cross-aggregate authorization edge cases.

**Reader task**: Use this document to establish who the actor is, whether the
actor can authenticate, which permission-grant and outlet-scope facts exist,
and which actor combinations require audit. Use the owning aggregate document to
decide whether those facts authorize a specific command or read.

**Sources**: §2 Personas, §3 Access Matrix, §4 Auth Model, E-02, E-04-E-05,
E-07-E-09, MVP authentication simplification decision on 2026-06-14

## Invariants

**BI-16 — A persona cannot approve their own submission.**

- Any action requiring approval cannot be approved by the same individual who
  submitted or requested it.
- This applies regardless of role.
- Covered examples include expense above threshold and inventory adjustment.

**BI-17 — Authorization is enforced by the owning boundary.**

- Identity owns account, authentication-link, explicit permission-grant,
  outlet-scope, and grant-combination facts.
- The aggregate receiving a command or read owns the authorization decision for
  that action.
- Domain aggregates must own their own business eligibility, thresholds,
  approval requirements, state-transition gates, policy outcomes, and audit
  basis.
- A permission grant is an input to an aggregate-owned decision. It is not
  sufficient to bypass aggregate-owned policy.

## Terms

| Term | Meaning |
| --- | --- |
| Persona | Canonical business archetype used in this specification. A persona is not an account type. |
| Cognito subject | Immutable Amazon Cognito user-pool subject used as the external authentication subject for one human account. |
| Verified phone number | Normalized E.164 phone number verified through Cognito SMS OTP and used as the launch login alias. |
| Authentication context | Validated Cognito JWT mapped to a current Identity account. The JWT proves authentication, not business authority. |
| Permission bundle | Explicit capability grant assigned to a human account. A permission bundle is an Identity-owned fact, not the complete decision for an aggregate action. |
| Outlet scope | Outlet or outlet set within which a permission may be exercised. |
| Experience | Customer, delivery, staff, or admin UI context derived from current permissions, outlet scopes, and account status. |
| Boundary authorization rule | Aggregate-owned rule that decides whether actor, account, grant, scope, lifecycle, and policy facts allow a command or read at that aggregate boundary. |

## Personas

These nine personas are canonical business archetypes. Runtime authorization
uses explicit permissions and scope assignments plus the aggregate-owned
boundary authorization rules for the action. A single human account may hold
multiple permission bundles across one or more outlets.

`P-07` is intentionally unused. Preserve the reserved ID to keep existing
persona references stable; do not reassign it without a documented migration.

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

- Outlet staff responsible for walk-in POS cash sales within their assigned
  outlet.
- Creates and completes scoped POS sales at the outlet counter.
- May handle customer cash refund payouts only when explicitly permissioned.
- Verifies customer-presented refund collection codes for collectible refunds
  only when explicitly granted refund-payout permission.
- Disburses collectible cash refunds from the owning outlet only when explicitly
  granted refund-payout permission.
- Has no delivery responsibilities.
- Cannot manage online orders, create refund liabilities, void POS sales,
  adjust inventory outside POS completion, or access another outlet's data.

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
  domains, and records, subject to aggregate-owned authorization rules and audit
  requirements.
- Manages global pricing, approves above-threshold inventory adjustments,
  manages identity permission grants and outlet scopes, and other actions no
  other persona can execute.
- Closes claim-blocked orders as unclaimable when Order-owned policy evidence
  supports terminal closure.
- Sensitive mutations and overrides require explicit reason codes and permanent
  audit logging.
- Cannot issue unaudited mutations.
- Super Admin authority extends audit requirements; it does not bypass them.

## Access Matrix

This matrix is a cross-aggregate orientation index for personas, grants, and
scope. It is not the central source of truth for every authorization decision.
Each aggregate's `Permissions` section owns the enforceable rows for commands
and reads at that aggregate boundary. If this matrix and an aggregate permission
section diverge, treat it as a documentation defect and fix both instead of
using this file as a central override.

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
| POS sale creation and completion | - | - | Scoped | - | - | - | - | - | Full |
| POS sale read/reporting | - | - | Scoped | - | - | Scoped | Read assigned outlets | Read | Full |
| POS same-day void | - | - | - | - | - | Scoped | - | - | Full |
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
| Identity grant and scope administration | - | - | - | - | - | - | - | - | Full |

## Matrix Scope Notes

### Scoped Access

- `Scoped` means the persona's assigned outlet(s) only.
- A persona assigned to Outlet A has no visibility into Outlet B unless Super
  Admin explicitly grants additional access.
- Area Managers have read access to their assigned outlet set, not all outlets.

### Inventory Adjustment Permission

- Identity records inventory adjustment permission grants and outlet scope.
- [inventory.md](inventory.md) owns inventory adjustment thresholds, approval
  outcomes, ledger-posting rules, and whether a grant authorizes a specific
  adjustment command.
- Inventory Clerks may submit scoped inventory adjustment requests.
- Outlet Managers may post only policy-limited scoped adjustments allowed by
  Inventory, and must submit above-policy adjustments for Super Admin approval.
- Super Admins may approve and post inventory adjustments across all outlets.
- Permission to submit, post, or approve an adjustment does not bypass
  Inventory-owned source-reference, reason, note, policy-code, ledger
  correlation, or audit requirements.
- Inventory viewing or adjustment permission does not authorize a formal stock
  count lifecycle, count-window variance workflow, or low-stock alert
  administration at launch. Those workflows are deferred in
  [../out-of-scope/2026-06-14-stock-counts-low-stock-alerts.md](../out-of-scope/2026-06-14-stock-counts-low-stock-alerts.md).

### Refund Payout Permission

- Refund owns payout authorization at the refund boundary. Identity supplies
  actor, permission-grant, and outlet-scope facts.
- When a customer context is used for refund payout, Identity supplies current
  account-status, Identity account ID, Cognito subject, verified phone, and
  authentication-link facts.
- Refund decides payout eligibility from the immutable refund/order identity
  snapshot. Current Identity phone or authentication-link facts are context, not
  the payout authority.
- Identity phone relink, Cognito-subject relink, or account recovery does not
  rewrite historical order or refund payout identity facts.
- If current Identity facts conflict with refund/order snapshot facts, Refund
  fails payout closed unless Refund-owned exception policy authorizes a recorded
  identity-mismatch exception.
- An Outlet Cashier with explicit refund-payout permission may verify the
  customer's collection code and disburse collectible refunds within their
  outlet scope.
- An Outlet Manager with explicit refund-payout permission may verify the
  customer's collection code and disburse collectible refunds within their
  outlet scope.
- Launch refund payouts have no per-refund or per-outlet-business-day cash cap.
- Without explicit per-outlet refund-payout permission, Cashiers and Outlet
  Managers cannot handle refund payouts.

### POS Cashier Authority

- POS Sale owns POS sale authorization at the POS boundary. Identity supplies
  actor, role, account-status, and outlet-scope facts.
- The Outlet Cashier persona carries scoped POS sale creation and completion
  authority for assigned outlets at launch.
- No separate POS-sale permission bundle is required for an assigned Outlet
  Cashier at launch.
- POS sale authority does not authorize refund payout, POS void, online order
  management, inventory adjustment, price override, or another outlet's POS
  sale access.
- Outlet Managers may void eligible same-outlet, same-business-day POS sales
  before daily closing under [pos.md](pos.md); Outlet Cashiers cannot void
  completed POS sales.

### Refund Source and Payout Boundary

- Refund liabilities have no amount-based approval threshold at launch.
- The supported launch refund-liability source is an authorized and posted
  Finance-owned cash over-collection correction.
- Finance owns source authorization for cash over-collection correction.
  Identity supplies actor, permission-grant, and outlet-scope facts used by
  Finance.
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

- Identity records pricing-related permission grants and outlet scope.
- [catalog.md](catalog.md) owns global catalog, product, refill, accessory, tax,
  launch pricing administration rules, and whether a grant authorizes a
  specific catalog or pricing command.
- [delivery.md](delivery.md) owns delivery-fee calculation rules and whether a
  grant authorizes a delivery-fee command.
- At launch, `Global pricing & catalog` is Super Admin-only.
- No outlet-scoped delivery-fee override permission exists at launch.
- No Outlet Manager, Dispatcher, Delivery Agent, Area Manager, or other
  outlet-scoped actor may create, change, approve, or apply outlet-local
  delivery-fee rules.
- Outlet-local product, refill, accessory, and delivery-fee price permissions are
  deferred in
  [../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md](../out-of-scope/2026-06-14-outlet-local-pricing-guardrails.md).

### Claim-Block Permissions

- Order owns claim-block authorization at the order boundary. Identity supplies
  actor, permission-grant, and outlet-scope facts.
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

- Launch uses one Amazon Cognito user pool and one app client for human users.
- The Cognito user pool permits SMS OTP as the passwordless first
  authentication factor.
- The Cognito app client permits the choice-based `USER_AUTH` flow.
- All human users authenticate through passwordless SMS OTP.
- Email OTP, password login, and privileged MFA are not active at launch.
- The launch client uses Cognito SDK/API authentication, not Cognito managed
  login or hosted UI.
- All human users sign up or sign in with a normalized E.164 phone number.
- Phone control is verified through Cognito SMS OTP at registration or login.
- One normalized verified phone number maps to one Cognito user and one Nyange
  Identity account.
- The Cognito `sub` claim is the immutable external authentication subject.
- The verified phone number is the launch login alias and account matching key.
- A valid Cognito JWT proves the authenticated Cognito subject, verified phone
  context, and authentication time.
- Cognito JWTs do not prove account status, permission grants, outlet scopes,
  grant-combination validity, or aggregate-specific authority.
- Access tokens and ID tokens expire after 15 minutes.
- Refresh token rotation is enabled.
- Refresh tokens expire after 7 days.
- The API validates Cognito JWTs statelessly at the boundary before loading
  current Identity facts.
- Public self-signup creates or reuses a customer-capable Identity account.
- The customer account/profile is ensured before customer features are
  available.
- Staff, delivery, finance, manager, and Super Admin authority is assigned
  through Identity permission grants and outlet scopes by a Super Admin.
- Self-signup, login, or experience selection cannot create privileged
  authority.
- The first Super Admin authority is seeded out-of-band during deployment or
  migration against a verified phone number and Cognito subject.
- After bootstrap, no root password, separate bootstrap app client, or permanent
  authentication bypass remains.
- Normal checkout and sensitive launch operations do not require extra OTP
  step-up beyond the authenticated Cognito session.

### OTP and Credential Ownership

- Cognito owns SMS OTP issuance, verification, expiry, retry, and challenge
  state.
- AWS SMS, SNS, Pinpoint, WAF, or Cognito threat-protection configuration owns
  SMS delivery and abuse controls.
- Nyange does not store OTP codes, OTP attempt counters, passwords, MFA factors,
  refresh tokens, device fingerprints, or application-owned server sessions.
- Nyange records only the Cognito subject, verified phone number, account
  status, customer-profile link, permission grants, outlet scopes,
  grant-combination facts, timestamps, and audit records for sensitive Identity
  changes.
- Authentication failures must not disclose account existence or privileged
  authority.

### Recovery

- Any user may submit a generic unauthenticated support request using login
  alias and safe contact context.
- The generic request does not prove identity.
- It does not disclose account existence or status.
- It does not authenticate the user.
- It does not change credentials, grants, scopes, or account links.
- Normal OTP recovery is handled by Cognito/AWS behavior and the user's phone
  carrier access.
- Nyange launch recovery is limited to an audited Super Admin relink of an
  existing Identity account to a new verified phone number and Cognito subject.
- The relink record must identify the subject Identity account, previous
  Cognito subject when known, new Cognito subject, previous phone number when
  known, new verified phone number, performing Super Admin actor, reason code,
  note, timestamp, before/after account-status and auth-link values, and
  confirmation that no authentication secrets are stored.
- Password reset, MFA reset, privileged credential recovery, dual-admin recovery
  ceremony, and app-owned session recovery are not launch behavior.

### Account Status and Revocation

- Active account status is required after successful Cognito authentication.
- Disabled accounts cannot access any protected experience or exercise any
  business permission.
- Account disablement, permission-grant removal, and outlet-scope removal take
  effect on the next protected API request.
- Token expiry is not the business-authority revocation mechanism.
- Cognito global sign-out or provider token revocation may be used as
  defense-in-depth, but Identity account, grant, and scope facts remain the
  authority source.
- If account-status, authentication-link, grant, scope, or grant-combination
  facts conflict, access fails closed until corrected through an audited path.

## Authorization Model

- Identity provides actor identity, active account status, authentication
  context, explicit permission grants, outlet scopes, and grant-combination
  facts.
- Each aggregate decides authorization for commands and reads at its own
  boundary.
- A single human account may hold multiple permission bundles across one or more
  outlets.
- Aggregate authorization decisions must evaluate the Identity facts required by
  that aggregate plus the aggregate-owned lifecycle state, business eligibility,
  thresholds, approval requirements, policy outcomes, and audit basis.
- A permission grant or outlet scope does not authorize an action unless the
  enforcing aggregate's boundary rule accepts it.
- Login itself does not let the user choose a persona or experience as a source
  of authority.
- Experiences are derived from current permissions, outlet scopes, and account
  status.
- Experiences are not separate account types.
- Selecting an experience never grants new permissions, outlet scope, or
  business authority.
- Outlet rosters, work queues, and operational projections may organize work or
  explain visibility.
- Those projections are not authorization sources and cannot grant permission or
  outlet scope beyond explicit assignments.
- Only the latest valid Identity account-status, permission-grant, outlet-scope,
  and grant-combination facts may be used as inputs to aggregate authorization
  decisions.
- Draft, invalid, failed, or unapproved grant or scope changes cannot expand
  access.
- When authoritative account, authentication-link, permission, scope, or
  aggregate policy facts conflict, access fails closed until corrected through
  an audited path.
- Authorization denials and sensitive successful grants are audit-relevant.
- Audit-relevant authorization decisions must identify the active authorization
  basis used for the decision, including Identity facts and the aggregate-owned
  policy basis where applicable.
- Ordinary successful reads are not individually audit-logged unless this
  document classifies the read as sensitive.

### Identity Audit Scope

- Identity audit records are required for Super Admin bootstrap, account
  enablement, account disablement, phone/Cognito-subject relink, permission
  grant creation, permission grant update, permission grant removal,
  outlet-scope creation, outlet-scope update, outlet-scope removal, and
  grant-combination allow/deny changes.
- Authorization denials remain audit-relevant when this document or the
  enforcing aggregate classifies the decision as audit-relevant.
- Ordinary successful SMS OTP logins are observed through Cognito/AWS logs and
  are not duplicated as Nyange domain audit records at launch.
- Ordinary successful reads are not individually audit-logged unless this
  document or the enforcing aggregate classifies the read as sensitive.

## Out of Scope

The following authentication controls are deferred from launch MVP:

- password login;
- privileged MFA;
- step-up authentication;
- invite lifecycle;
- multiple Cognito user pools;
- multiple Cognito app clients for human users;
- custom OTP implementation;
- application-owned server sessions;
- device management;
- Nyange auth telemetry warehouse;
- token-embedded business grants or outlet scopes.

Deferred authentication controls are tracked in
[../out-of-scope/2026-06-14-advanced-auth-controls.md](../out-of-scope/2026-06-14-advanced-auth-controls.md).

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
A valid authorized and posted Finance-owned cash over-collection correction that
confirms customer cash collected above expected COD creates a refund liability
eligible for collection-code issuance regardless of amount. Finance owns source
authorization and any required separation-of-duty rule for the source correction.

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
permissions at the same outlet only when Identity grant-combination rules permit
that combination. The combination must be explicit, recorded, and subject to
audit on sensitive actions. Each enforcing aggregate still applies its own
boundary authorization rules, and the actor still cannot approve their own
submissions or bypass normal approval thresholds.
