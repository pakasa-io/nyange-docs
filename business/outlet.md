# Outlet Configuration & Serviceability Facts

**Intent**: Define the authoritative outlet aggregate boundary — what an outlet
is, who owns its configuration and serviceability facts, and which records,
commands, queries, events, and audit rules belong to the outlet boundary.

**Reader task**: Use this document to decide outlet online-fulfillment
eligibility, service area coverage, business day/timezone, vendor acceptance,
outlet disablement blockers, and the outlet facts consumed by Order, Cart,
Catalog, Delivery, Inventory, Identity, and Finance. Use the consuming
aggregate's own document for how it interprets those facts.

**Sources**: Business domain README Outlet Model,
[identity-auth.md](identity-auth.md) (outlet-scope facts, persona matrix),
[order.md](order.md) (claiming, outlet-disable blockers),
[delivery.md](delivery.md) (delivery task outlet, delivery fee basis),
[finance.md](finance.md) (daily closing due-dates, receipt series, cash custody),
[inventory.md](inventory.md) (outlet stock isolation, transfers),
[catalog.md](catalog.md) (vendor acceptance),
[pos.md](pos.md) (outlet-local sales),
Backend implementation plan G-01, IP-02.

## Module Ownership

| Backend module | Owns | Does not own |
| --- | --- | --- |
| `outlet` | Outlet records, outlet status lifecycle, service areas, online-fulfillment capability, business day and timezone, staff scope definition, vendor acceptance list, operating facts, outlet disablement blockers, coarse address serviceability queries, pending-pool candidate outlet queries, daily-closing due-date basis, receipt-series basis, claim-eligibility outlet inputs, outlet-scoped authorization inputs. | Identity permission-grant facts (who may access which outlet), aggregate-specific authorization outcomes, delivery fee calculation, delivery task lifecycle, order claiming policy, stock availability, catalog product/pricing rules, POS sale lifecycle, finance ledger posting, daily-closing execution, refund lifecycle. |

**Boundary principle**: `outlet` owns the facts that describe an outlet.
`identity` owns the facts that describe who may act within which outlet.
Aggregates that consume outlet facts (Order, Cart, Delivery, Finance, Inventory)
own the policies that interpret those facts for their own actions.

## Invariants

**BI-40 — An outlet is a company-owned branch within the same legal entity.**

- Outlet independence is operational and reporting-oriented, not legal or
  financial separation.
- Outlets are not franchisees or marketplace merchants.
- Outlet cash is company cash.

**BI-41 — Outlet stock is isolated.**

- Stock at Outlet A cannot be drawn down by an order claimed by Outlet B.
- Stock transfers between outlets follow an explicit, audited transfer
  lifecycle owned by Inventory.
- No order can implicitly access another outlet's inventory.

**BI-42 — Outlet disablement is blocked by active dependencies.**

- An outlet cannot be disabled while it has an active claim, active reservation,
  active delivery task, unresolved agent custody, unresolved customer order
  custody (RETURN_PENDING), or unresolved cash custody.
- Unclaimed PENDING and CLAIM_BLOCKED orders do not block outlet disablement
  because they have no claimed fulfilling outlet.
- An outlet may be disabled when every active order with an outlet association
  has reached a terminal state and no related custody, reservation, delivery
  task, or refund liability remains unresolved.

**BI-43 — Coarse serviceability requires at least one active online-fulfillment
outlet.**

- An address is serviceable when at least one active outlet has
  `online_fulfillment_enabled = true` and the address falls within that outlet's
  service area.
- Pre-order validation avoids per-outlet SKU/vendor filtering before order
  placement.
- Pending-pool visibility and claim attempts are limited to active
  online-fulfillment outlets that serve the delivery area.

**BI-44 — Outlet facts are immutable for placed orders and completed POS sales.**

- Later catalog, price, outlet, tax, stock, or delivery-fee changes do not
  alter placed orders or completed POS snapshots.
- Order placement snapshots outlet-eligible facts as of placement time.
- POS completion snapshots outlet facts as of completion time.

**BI-45 — Outlet configuration commands require Super Admin authority.**

- Only P-10 Super Admin may create, update, enable, or disable outlet
  configuration records.
- Outlet Managers and Area Managers have read access to outlet configuration
  within their scope.
- Sensitive mutations require reason codes and permanent audit logging.

## Terms

| Term | Meaning |
| --- | --- |
| Outlet | A company-owned branch that stocks inventory, fulfills orders, serves walk-in customers, and operates a cash ledger. Each outlet has its own inventory, staff, and reporting. |
| Outlet status | Active, inactive (disabled), or suspended. Only Super Admin may transition between statuses. |
| Online fulfillment | An outlet capability flag. When enabled, the outlet may serve as a fulfilling outlet for online COD orders and appear in pending-pool candidate lists. |
| Service area | A named geographic zone or set of zones that an outlet serves for delivery fulfillment. Address-to-outlet matching uses coarse zone membership, not precise distance. |
| Business day | Outlet-local calendar day defined by the outlet's configured timezone. Used for daily-closing deadlines, POS same-day void windows, and receipt date assignment. |
| Staff scope | The outlet or outlet set within which a permission may be exercised. Defined by outlet identity, consumed by Identity for permission-grant and scope facts. |
| Vendor acceptance list | The set of cylinder vendors an outlet accepts for incoming refill exchanges at claim time. Distinct from global vendor support and catalog product rules. |
| Operating facts | Derived outlet attributes computed from configuration and status — e.g., is_open_now, next_business_day, days_until_closing_due. |
| Claim-eligibility inputs | Outlet facts (status, online-fulfillment, service area membership, vendor acceptance, claim-blocking reasons) consumed by Order to evaluate claim eligibility. |
| Receipt-series basis | Outlet identity plus receipt date, consumed by Finance to produce immutable outlet/date receipt numbers. |
| Daily-closing due-date basis | Outlet business day and timezone, consumed by Finance to compute closing deadlines and overdue gates. |

## Outlet Record

An outlet record is the aggregate root. An outlet is identified by an
immutable outlet ID. Configuration facts are versioned; mutations produce new
version facts and never overwrite committed snapshot history.

| Field | Type | Description |
| --- | --- | --- |
| `outlet_id` | UUID | Immutable outlet identifier. |
| `name` | string | Human-readable outlet name. |
| `status` | enum: `ACTIVE`, `INACTIVE`, `SUSPENDED` | Outlet operational status. |
| `online_fulfillment_enabled` | boolean | Whether this outlet may fulfill online COD orders. |
| `service_areas` | []string | Named delivery zones this outlet serves. |
| `timezone` | IANA timezone string | Outlet local timezone for business day and closing deadline calculation. |
| `business_days` | []day-of-week | Days the outlet operates (e.g., Mon–Sat). |
| `vendor_acceptance` | []vendor_id | Vendors this outlet accepts for incoming refill exchanges. Empty means all globally supported vendors are accepted. |
| `created_at` | timestamp | Record creation time. |
| `updated_at` | timestamp | Last configuration mutation time. |
| `disabled_at` | timestamp | When the outlet was disabled, if INACTIVE. |
| `disabled_reason` | string | Reason code and note for disablement. |
| `version` | integer | Monotonic configuration version. |

## Outlet Status Lifecycle

```
                  ┌──────────┐
                  │  ACTIVE   │
                  └─────┬─────┘
                        │ disable (must pass blocker check)
                        ▼
                  ┌──────────┐
                  │ INACTIVE  │
                  └─────┬─────┘
                        │ enable
                        ▼
                  ┌──────────┐
                  │  ACTIVE   │
                  └──────────┘

ACTIVE ──(emergency suspend)──▶ SUSPENDED
SUSPENDED ──(resume)──▶ ACTIVE
SUSPENDED ──(disable)──▶ INACTIVE
```

- **ACTIVE → INACTIVE**: Rejected if BI-42 blockers exist. Reason code and
  audit record required.
- **INACTIVE → ACTIVE**: Re-enables outlet. Audit record required.
- **ACTIVE → SUSPENDED**: Emergency suspension does not check BI-42 blockers.
  Reason code and audit record required. Suspended outlets are excluded from
  serviceability and pending-pool queries.
- **SUSPENDED → ACTIVE**: Resume normal operations. Audit record required.
- **SUSPENDED → INACTIVE**: Permanent disablement from suspended state.
  BI-42 blockers checked at this transition.

## Commands

All outlet configuration commands are Super Admin-only.

| Command | Input | Effect | Audit |
| --- | --- | --- | --- |
| `CreateOutlet` | name, timezone, business_days, service_areas, online_fulfillment_enabled, vendor_acceptance | Creates outlet in ACTIVE status, version 1. | Actor, timestamp, created outlet ID, full configuration snapshot. |
| `UpdateOutlet` | outlet_id, any mutable field | Mutates configuration, increments version. Does not affect placed orders or completed POS snapshots. | Actor, timestamp, outlet ID, before/after field diffs, version. |
| `EnableOutlet` | outlet_id | INACTIVE → ACTIVE. | Actor, timestamp, outlet ID, reason. |
| `DisableOutlet` | outlet_id, reason | ACTIVE → INACTIVE. Rejected if BI-42 blockers exist. | Actor, timestamp, outlet ID, reason, blocker check result. |
| `SuspendOutlet` | outlet_id, reason | ACTIVE → SUSPENDED. Bypasses BI-42 blocker check. | Actor, timestamp, outlet ID, reason. |
| `ResumeOutlet` | outlet_id | SUSPENDED → ACTIVE. | Actor, timestamp, outlet ID, reason. |

## Queries

Queries return current outlet facts at read time. Consumers may cache results
within a bounded read-model freshness window defined by the consuming
aggregate.

| Query | Input | Output | Consumers |
| --- | --- | --- | --- |
| `GetOutlet` | outlet_id | Current outlet record | All aggregates, identity scope resolution |
| `ListOutlets` | status filter, pagination | Paginated outlet list | Admin reporting, Super Admin configuration |
| `CheckServiceability` | address (zone or coordinate) | `{serviceable: bool, candidate_outlet_ids: []uuid}` | Cart, Order for pre-order validation |
| `ListPendingPoolCandidates` | delivery_address_zone | Active online-fulfillment outlets serving the zone | Order for claim eligibility and pending-pool scoping |
| `GetClaimEligibilityInputs` | outlet_id, order_frozen_facts | `{eligible: bool, blocking_reasons: []string, vendor_accepted: bool}` | Order for claim-time authorization |
| `GetDailyClosingBasis` | outlet_id, date | `{due_by: timestamp, business_day_date: date, is_overdue: bool}` | Finance for daily closing deadlines |
| `GetReceiptSeriesBasis` | outlet_id, date | `{outlet_id, receipt_date}` | Finance for receipt numbering |
| `GetOutletScopedAuthorizationFacts` | outlet_id, actor_id | `{outlet_exists: bool, outlet_status, staff_scope_match: bool}` | Identity for scope resolution |
| `CheckDisablementBlockers` | outlet_id | `{blocked: bool, blockers: [{type, id, description}]}` | Outlet for disable command pre-check, Admin UI |

## Events

Events are emitted post-commit by the outlet aggregate. They are durable
domain facts consumed by other aggregates and by the notification fanout.

| Event | Trigger | Payload |
| --- | --- | --- |
| `OutletCreated` | CreateOutlet command succeeds | outlet_id, name, full configuration snapshot, actor, timestamp |
| `OutletUpdated` | UpdateOutlet command succeeds | outlet_id, changed fields with before/after, version, actor, timestamp |
| `OutletEnabled` | EnableOutlet command succeeds | outlet_id, actor, timestamp |
| `OutletDisabled` | DisableOutlet command succeeds | outlet_id, reason, blocker check result, actor, timestamp |
| `OutletSuspended` | SuspendOutlet command succeeds | outlet_id, reason, actor, timestamp |
| `OutletResumed` | ResumeOutlet command succeeds | outlet_id, actor, timestamp |

## Audit

Every outlet configuration mutation creates a permanent audit record. Audit
records are append-only and never mutated or deleted.

| Audit field | Description |
| --- | --- |
| `audit_id` | Unique immutable audit record identifier. |
| `outlet_id` | Target outlet. |
| `actor_id` | Identity account that performed the mutation. |
| `action` | Command name. |
| `before` | Full outlet snapshot before mutation (null for CreateOutlet). |
| `changed_fields` | List of changed field names with before/after values. |
| `reason` | Required reason code and optional note for disable/suspend/resume. |
| `blocker_check` | For disable: list of active blockers at check time and the pass/fail outcome. |
| `timestamp` | Mutation commit time. |

## Permissions

The outlet aggregate owns the authorization rules for outlet configuration
commands. Identity supplies actor, account status, permission-grant, and
outlet-scope facts.

| Action | P-01 Customer | P-02 Agent | P-03 Cashier | P-04 Inv Clerk | P-05 Dispatcher | P-06 Outlet Mgr | P-08 Area Mgr | P-09 Finance | P-10 Super Admin |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Read outlet config | - | - | - | - | - | Scoped | Read assigned outlets | - | Full |
| Create outlet | - | - | - | - | - | - | - | - | Full |
| Update outlet | - | - | - | - | - | - | - | - | Full |
| Enable outlet | - | - | - | - | - | - | - | - | Full |
| Disable outlet | - | - | - | - | - | - | - | - | Full |
| Suspend outlet | - | - | - | - | - | - | - | - | Full |
| Resume outlet | - | - | - | - | - | - | - | - | Full |

- Outlet Managers may read outlet configuration for their assigned outlet only.
- Area Managers may read outlet configuration for their assigned outlet set
  only.
- Super Admin is the only persona authorized to mutate outlet configuration.

## Cross-Aggregate Relationships

### Outlet → Identity

- Identity owns permission-grant facts and outlet-scope facts. Outlet scope
  is defined by outlet identity.
- Identity queries `GetOutletScopedAuthorizationFacts` to validate that an
  outlet exists and is active when resolving scope.
- Outlet does not own or mutate permission grants, account status, or
  authentication facts.

### Outlet → Order

- Order queries `CheckServiceability` for pre-order validation.
- Order queries `ListPendingPoolCandidates` to scope pending-pool visibility.
- Order queries `GetClaimEligibilityInputs` at claim time to evaluate vendor
  acceptance and claim-blocking reasons.
- Order owns the claim decision. Outlet supplies eligibility inputs.
- Order's BI-22 (outlet disable blocked by active orders) is enforced by
  Outlet's `CheckDisablementBlockers` query.

### Outlet → Delivery

- Delivery-fee rules reference launch zones, which are defined by outlet
  service areas.
- Delivery tasks are scoped to the fulfilling outlet. The outlet must be
  active and online-fulfillment enabled at task creation.
- Outlet-local delivery-fee overrides are not launch behavior (deferred in
  out-of-scope).

### Outlet → Finance

- Finance queries `GetDailyClosingBasis` for closing deadline computation.
- Finance queries `GetReceiptSeriesBasis` for outlet/date receipt numbering.
- Outlet cash custody is owned by Finance, not Outlet.

### Outlet → Inventory

- Inventory enforces BI-41 (outlet stock isolation). Outlet identity is used
  for stock location partitioning.
- Inventory transfers reference source and destination outlet IDs.
- Outlet does not own stock counts, reservation lifecycle, or transfer
  lifecycle.

### Outlet → Catalog

- Outlet vendor acceptance list is consulted at claim time (via Order).
- Catalog owns global vendor support and product rules. Outlet filters which
  vendors are accepted locally.
- Outlet-local pricing, SKU, or tax overrides are not launch behavior (deferred
  in out-of-scope).

### Outlet → POS

- POS sales are outlet-local. The selling outlet ID is recorded at sale
  creation.
- POS completion snapshots outlet facts (outlet ID, status at completion time).
- POS same-day void eligibility uses the selling outlet's business day.

## Use Rules

- This document is the authoritative source for outlet configuration and
  serviceability facts.
- Do not move outlet configuration ownership to Identity, Order, Delivery,
  or Finance without updating this document, the module ownership table, and
  the aggregate index in [README.md](README.md).
- Mark unresolved outlet facts as open questions in this document instead of
  inventing missing policy.
- Consuming aggregates must reference this document for the outlet facts they
  depend on, and must define their own interpretation policies.
