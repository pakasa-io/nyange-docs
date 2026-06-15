# Backend Business Domain Implementation Plan

**Intent**: Provide a source-grounded backend implementation plan for
`nyange-api` from the current Nyange business domain specification.

**Reader task**: Use this plan to sequence backend module specs, OpenAPI
contracts, migrations, tests, and implementation tickets without violating
business aggregate ownership.

**Status**: Draft

**Date**: 2026-06-14

## Source Documents

Primary sources:

- [Business domain index](../../../business/README.md)
- [Identity & Authorization Facts](../../../business/identity-auth.md)
- [Catalog & Pricing](../../../business/catalog.md)
- [Cart](../../../business/cart.md)
- [Order](../../../business/order.md)
- [POS Sale](../../../business/pos.md)
- [Payment](../../../business/payment.md)
- [Delivery](../../../business/delivery.md)
- [Inventory](../../../business/inventory.md)
- [Refund](../../../business/refund.md)
- [Finance](../../../business/finance.md)
- [Notifications](../../../business/notifications.md)

Engineering sources:

- [Backend manifest](../manifest.yml)
- [Coding standards](../../common/coding-standards.md)
- [Modular monolith guide](../../../start-here/modular-monolith-guide.md)
- [Documentation style](../../../start-here/doc-style.md)

## Scope

This plan covers backend launch MVP implementation for a Go, Fiber v3,
Postgres-backed modular monolith.

Included launch workflows:

- authenticated customer cart, quote, and online COD order placement;
- global launch catalog, refill pricing, bundle pricing, and delivery-fee
  calculation;
- pending-pool order claiming, claim-block exceptions, and unclaimable closure;
- inventory reservation, commitment, release, transfers, adjustments, vendor
  refill batches, and returned-cylinder intake;
- single-order delivery task assignment, pickup, doorstep completion, failed
  delivery, and delivery cash evidence;
- Payment-owned COD expectations, COD collection facts, POS cash facts, and
  zero-collection facts;
- Finance-owned receipts, cash custody, cash handover, variance records, daily
  closing, expenses, and reporting inputs;
- walk-in POS cash sales and same-day voids;
- Finance-sourced refund liabilities, collection codes, cash payout, void, and
  write-off;
- transactional push-notification fanout attempts.

Excluded launch workflows remain out of scope unless Product accepts them into
launch scope:

- mobile money, cards, customer credit, subscriptions, loyalty, and advanced
  promotions;
- outlet-local product, refill, accessory, tax, or delivery-fee price overrides;
- delivery batching, routing, automatic outlet allocation, ranking, or
  scheduled delivery windows;
- formal stock counts, low-stock alerts, and serial-number-specific cylinder
  tracking;
- POS customer returns, exchanges, post-sale price adjustments, and POS refund
  liabilities;
- WhatsApp, email notifications, customer notification preferences, or SMS
  outside Cognito OTP.

## Implementation Constraints

- Business documents are authoritative. Existing implementation, generated
  artifacts, or tests are evidence only until reconciled into docs.
- Backend-specific docs, plans, module specs, decisions, and gap records must
  live under a `../nyange-docs` path containing `/backend/`.
- Implement with the approved backend stack: Go, Fiber v3, Postgres, GORM
  repositories with explicit mappers, `github.com/pakasa-io/uow`, goose
  migrations, Cognito JWT authentication, Cedar authorization policies,
  OpenAPI-first REST under `/v1`, in-process post-commit events, and
  OpenTelemetry plus structured logs.
- Domain packages must not expose framework, SDK, ORM, DI, UoW, or transport
  types.
- Mutating HTTP commands require `Idempotency-Key` unless the backend module
  spec explicitly exempts the operation.
- Cross-module writes must go through documented commands, participant
  mutations, or coordinator contracts inside the unit of work.
- External side effects, including notification delivery, must run after commit.
- Inventory ledger, Finance ledger, history, event, audit, receipt, and
  idempotency records are append-only unless a business source explicitly
  defines a linked reversal or compensating record.

## Open Specification Gaps

These gaps should be resolved before implementation reaches the affected slice.

| Gap | Impact | Required resolution |
| --- | --- | --- |
| `G-01` Outlet configuration and serviceability ownership is not a dedicated aggregate document, but Order, Cart, Catalog, Delivery, Inventory, Identity, and Finance depend on outlet status, service areas, online-fulfillment capability, outlet timezone/business day, vendor acceptance, and outlet scopes. | Blocks deterministic serviceability, pending-pool visibility, claim eligibility, daily closing due dates, receipt series, and outlet-scoped authorization. | Create a backend/common or business authority document naming the owner, records, commands, queries, events, and audit rules for outlet configuration and serviceability facts. |
| `G-02` Pending-claim timeout duration remains `OQ-01` in Order. | Blocks deterministic timeout-based `PENDING -> CLAIM_BLOCKED` transitions. | Product Manager must set the launch pending-claim timeout duration before Order claim-block jobs are implemented. |
| `G-03` Backend module specs and OpenAPI contracts are not yet authored for the business aggregates. | Blocks code-first implementation under spec-driven TDD. | Before each implementation slice, create or update backend module specs and OpenAPI contracts under `/backend/`, then write failing tests from those specs. |
| `G-04` Push notification provider, device-token ownership, and delivery-attempt retention are not specified. | Blocks production notification delivery beyond durable fanout requests and attempts. | Define the provider adapter boundary and token ownership before implementing real push delivery. Until then, implement durable notification requests and no-op/local adapters only. |

## Module Ownership

Use one backend module per business aggregate unless a later accepted boundary
map changes this.

| Backend module | Owns | Does not own |
| --- | --- | --- |
| `identity` | Identity accounts, Cognito subject links, verified phone facts, account status, saved addresses, permission grants, outlet scopes, grant-combination facts, identity audit. | Aggregate-specific authorization outcomes, order state, catalog rules, cash records, inventory state. |
| `outlet` | Outlet records, outlet status lifecycle, service areas, online-fulfillment capability, business day/timezone, staff scope, vendor acceptance, operating facts, disablement blockers, coarse serviceability queries, pending-pool candidate queries, daily-closing due-date basis, receipt-series basis, claim-eligibility inputs, outlet-scoped authorization inputs, outlet audit records. | Identity permission-grant facts, aggregate-specific authorization outcomes, delivery fee calculation, delivery task lifecycle, order claiming policy, stock availability, catalog product/pricing rules, POS sale lifecycle, finance ledger posting, daily-closing execution, refund lifecycle. |
| `catalog` | Global products, vendors, refill pair eligibility, price rules, bundles, tax rules, version facts, catalog-price audit. | Delivery fees, stock availability, order snapshots, POS or online sale state. |
| `cart` | Customer active cart state, cart lines, selected address reference, quote readiness, catalog-change acknowledgements, abandoned-cart cleanup. | Order lifecycle, stock reservation, payment, delivery, refund, finance, audit ledgers. |
| `order` | Online order placement, immutable order snapshot, order lifecycle, claim association, pending pool, claim-block state, customer cancellation, unclaimable closure. | Inventory stock movements, delivery task state, payment facts, receipts, cash custody, notifications. |
| `inventory` | Stock availability, reservations, commitments, releases, transfers, adjustments, vendor refill batches, returned-cylinder intake, inventory ledger. | Delivery completion trigger, POS lifecycle, customer payment, finance ledgers, receipts. |
| `delivery` | Delivery tasks, assignment, pickup, outgoing goods custody, doorstep field facts, failed-delivery outcome, delivery fee rules, delivery collection evidence. | Inventory intake recognition, Payment outcome, Finance cash acceptance, Refund payout, receipts. |
| `payment` | COD expectation, expected COD amount, COD collected fact, zero-collection fact, POS cash fact, POS payment void fact. | Order outcomes, delivery field action, Finance variance, receipt, refund liability. |
| `finance` | Receipts, cash custody, handover acceptance, variance records, ledgers, daily closing, expenses, delivery-cost reporting, refund payout participant effects. | Payment outcome, order state, delivery task state, inventory ledger, refund lifecycle. |
| `pos` | Walk-in POS draft/completed/abandoned/voided lifecycle, POS public number, POS sale snapshot, same-day void eligibility. | Catalog pricing rules, stock ledger, payment facts, finance ledgers, refund liabilities. |
| `refund` | Refund liability lifecycle, collection code, payout eligibility, payout transition, void, write-off. | Finance source correction, order/payment mutation, electronic refunds, POS voids. |
| `notifications` | Durable notification requests, channel assignment, fanout attempts, template administration. | Business state transitions and transaction commit decisions. |

## Cross-Module Contracts

Author these contracts before implementation code for the workflow that uses
them.

| Contract | Type | Provider | Consumer | Consistency |
| --- | --- | --- | --- | --- |
| Current catalog line pricing and version facts | Query | Catalog | Cart, Order, POS | Live query at quote, placement, and POS completion. |
| Delivery-fee quote and serviceability result | Query | Delivery plus outlet/serviceability owner | Cart, Order | Live query; Order snapshots output at placement. |
| Saved address and resolved-coordinate facts | Query | Identity | Cart, Order | Cart reads current facts; Order stores immutable placement snapshot. |
| Checkout-ready cart to order placement | Command | Order | Cart | Atomic placement with Payment COD expectation creation. |
| COD expectation creation and retirement | Participant mutation | Payment | Order | Same UoW as order placement, cancellation, or unclaimable closure. |
| Stock reservation for order claim | Participant mutation | Inventory | Order | Same UoW as `PENDING -> CLAIMED`; whole-order only. |
| Delivery task creation | Participant mutation | Delivery | Order | Same UoW as `CLAIMED -> READY_FOR_PICKUP`. |
| Delivery completion | Coordinator | Delivery | Order, Inventory, Payment, Finance | Single UoW commits task, order, stock, payment, returned-cylinder field facts, receipt. |
| Terminal failed delivery | Coordinator | Delivery | Order, Inventory, Payment, Finance as needed | Single UoW after custody resolution; records zero collection and terminal task/order. |
| POS completion | Coordinator | POS | Catalog, Inventory, Payment, Finance | Single UoW commits sale, stock, cash fact, cash custody, receipt. |
| POS same-day void | Coordinator | POS | Inventory, Payment, Finance | Single UoW appends linked reversal records. |
| Cash over-collection correction to refund liability | Event or participant handoff | Finance | Refund | Post valid Finance source event creates Refund-owned liability. |
| Refund payout | Coordinator | Refund | Finance | Single UoW invalidates collection code, marks paid, posts outlet cash and ledger effects. |
| Transactional notifications | Domain event | Source aggregate | Notifications | Post-commit fanout; failure never blocks committed business state. |

## Implementation Slices

Each slice should become one or more ticket-scoped branches or Ralph tickets.
For each slice, update backend specs and OpenAPI first, write failing tests, then
implement the smallest compliant behavior.

### IP-00 Backend Skeleton And Quality Gates

**Sources**: Backend manifest, coding standards, modular monolith guide.

**Deliverables**:

- Initialize Go module, Fiber v3 HTTP app, `/livez`, `/readyz`, configuration
  loading, structured logging, request/correlation ID middleware, graceful
  shutdown, and container-first local runtime.
- Add Postgres wiring, goose migration runner, GORM repository conventions,
  explicit mapper conventions, UoW integration, and testcontainers harness.
- Add OpenAPI validation pipeline, contract test harness, generated or checked
  DTO boundary conventions, and API version root `/v1`.
- Add common idempotency store with actor, operation, client key, request-body
  hash, stored outcome, in-progress handling, and replay semantics.
- Add append-only audit/event infrastructure and in-process post-commit event
  dispatcher.

**Validation**:

- `go test ./...`
- migration up/down validation for non-production test DBs;
- OpenAPI validation;
- idempotency replay tests for completed, in-progress, and same-key/different
  body scenarios;
- architecture checks proving domain packages do not depend on framework, ORM,
  SDK, DI, UoW, or transport types.

### IP-01 Identity, Authentication, Authorization Facts

**Sources**: Identity & Authorization Facts.

**Deliverables**:

- Cognito JWT validation at the API boundary.
- Identity account records linked to Cognito `sub` and normalized E.164 verified
  phone.
- Customer self-signup account/profile ensure flow.
- Super Admin bootstrap through deployment or migration without a permanent
  runtime bypass.
- Saved address records with resolved-coordinate facts and audited coordinate
  corrections.
- Permission grants, outlet scopes, account status, grant-combination facts,
  and Identity audit records.
- Cedar policy integration for action/resource policy evaluation while keeping
  aggregate-specific authorization decisions inside the owning aggregate.

**Validation**:

- authentication failure responses do not reveal account existence or privileged
  authority;
- disabled account, removed grant, removed scope, and grant-combination conflict
  all fail closed on the next protected request;
- sensitive Identity changes create permanent audit records;
- aggregate test fixtures can request current Identity facts without importing
  Identity internals.

### IP-02 Outlet Configuration And Serviceability Foundation

**Sources**: Business domain index, Identity scope notes, Order, Cart, Delivery,
Finance, Inventory.

**Dependency**: Resolve `G-01` before coding this slice.

**Deliverables**:

- Backend authority spec for outlet records, outlet status, online-fulfillment
  flag, service areas, outlet timezone/business day, outlet staff scope, vendor
  acceptance list, and outlet operating facts.
- Queries for coarse address serviceability, pending-pool candidate outlets,
  outlet-scoped authorization facts, daily closing due-date basis, receipt
  series basis, and claim eligibility inputs.
- Super Admin-only outlet configuration commands with required audit records.

**Validation**:

- serviceability query returns at least one active online-fulfillment outlet for
  valid launch addresses and fails closed when no current serviceable outlet
  exists;
- scoped actors cannot read or mutate outlet facts outside their assigned
  scope;
- outlet disablement is blocked when active claims, reservations, delivery
  tasks, order custody, or unresolved cash/custody blockers exist.

### IP-03 Catalog, Pricing, Bundles, And Delivery-Fee Quote Primitives

**Sources**: Catalog & Pricing, Delivery fee rules, Cart, Order, POS.

**Deliverables**:

- Catalog records for launch products, accessories, cylinder sizes, supported
  vendors, refill pair eligibility, price rules, tax rules, bundle composition,
  effective windows, and version facts.
- Super Admin-only price and catalog administration with no backdating.
- Server-side line pricing for online quotes, order placement, and POS.
- Delivery-owned delivery-fee rules, launch zone defaults, and fee quote output.
- Query contracts consumed by Cart, Order, and POS.

**Validation**:

- disabled products, invalid bundles, unpriceable refill pairs, missing base
  refill prices, and missing cross-vendor surcharges are rejected;
- delivery fee defaults produce UGX 3,000, 5,000, or 8,000 for configured launch
  zones and reject unpriceable delivery addresses;
- later catalog, price, tax, or delivery-fee changes produce new version facts
  and never mutate placed order or completed POS snapshots.

### IP-04 Inventory Stock Core

**Sources**: Inventory.

**Deliverables**:

- Aggregate stock model by outlet, vendor, size, fill lifecycle, availability,
  condition, and SKU where applicable.
- Append-only inventory ledger.
- Reservation lifecycle: `RESERVED`, `COMMITTED`, `RELEASED`.
- Atomic whole-order reservation command for Order claim.
- Adjustment submission, policy evaluation, Super Admin approval path, and
  source-referenced ledger posting.
- Transfer request and stock movement lifecycle.
- Vendor refill batch lifecycle.

**Validation**:

- concurrent reservations cannot drive available stock below zero;
- reserved stock is unavailable for new orders, POS, transfers, or refill;
- claim reservation is all-or-nothing;
- ledger corrections are compensating entries, never edits;
- large or positive adjustments remain blocked by overdue daily closing unless
  an owning policy override exists.

### IP-05 Cart And Order Placement

**Sources**: Cart, Order, Catalog, Delivery, Payment, Identity.

**Deliverables**:

- Authenticated customer active cart lifecycle: `ACTIVE`, `CHECKED_OUT`,
  `CLEARED`, `ABANDONED`, `SUMMARY_RETAINED`.
- Cart line management for new cylinders, refill exchange lines, accessories,
  and bundles.
- Quote readiness and checkout readiness with server-computed totals.
- Catalog-change acknowledgement records tied to exact version tuple, quantity,
  and server-computed line price.
- Order placement from a checkout-ready cart into `PENDING`.
- Immutable order snapshot, `ORD-%08d` public number, structured delivery
  address snapshot, and order-scoped customer read/cancel permissions.
- Payment-owned COD expectation creation in the same atomic placement commit.

**Validation**:

- anonymous cart, quote, and checkout are rejected;
- client-submitted totals, fees, taxes, discounts, and COD amount are ignored;
- stale acknowledgements block quote and checkout;
- unresolved address coordinates block checkout and placement;
- order placement idempotency replays the same order and COD expectation;
- failure to create the COD expectation rolls back order placement.

### IP-06 Pending Pool, Claiming, Claim Block, And Unclaimable Closure

**Sources**: Order, Inventory, Catalog, Delivery, Payment, Notifications,
Identity.

**Dependencies**: Resolve `G-01` and `G-02` before timeout-based claim block.

**Deliverables**:

- Pending-pool read model scoped to active online-fulfillment outlets that serve
  the delivery area and hold claim permission.
- Whole-order claim command: `PENDING -> CLAIMED` with same-UoW Inventory
  reservation.
- Claim-block detection for timeout, empty candidate set, and all-candidates
  blocked by controlled reason codes.
- Claim-block resolution: scoped reopen to `PENDING` or Super Admin
  `UNCLAIMABLE` closure.
- Customer cancellation from `PENDING`, `CLAIM_BLOCKED`, `CLAIMED`, and
  `READY_FOR_PICKUP`; outlet claim cancellation from `CLAIMED` and
  `READY_FOR_PICKUP`.
- Payment expectation retirement participant for cancellation and unclaimable.
- Post-commit notification requests for claim blocked and unclaimable.

**Validation**:

- only eligible scoped outlets can read pending-pool detail or claim;
- claim is rejected unless every frozen line can be fulfilled by the claiming
  outlet;
- claim and reservation are inseparable and idempotent;
- claim-block marking creates no stock, delivery, payment terminal, receipt,
  refund, cash custody, or fulfilling-outlet effects;
- customer cancellation and unclaimable rollback if Payment expectation
  retirement rejects.

### IP-07 Ready For Pickup, Delivery Assignment, Pickup, And Custody

**Sources**: Order, Delivery, Inventory, Identity.

**Deliverables**:

- Claimed outlet ready-for-pickup command: `CLAIMED -> READY_FOR_PICKUP`.
- Delivery task lifecycle through `READY_FOR_PICKUP`, `ASSIGNED`, and
  `PICKED_UP`.
- Manual assignment and pre-pickup reassignment with recorded reason.
- Delivery Agent assignment-scoped read access to full address and phone only
  while assignment is active.
- Outlet handover and agent receipt confirmation as pickup prerequisites.
- Outgoing goods custody records for the assigned agent.
- Task-only cancellation before pickup without order claim cancellation.

**Validation**:

- only claimed outlet actors can mark ready;
- assignment eligibility requires active agent, outlet scope, and no active
  picked-up task;
- missing agent receipt blocks pickup;
- after pickup, normal reassignment is rejected;
- phone/address access is removed after terminal delivery task state.

### IP-08 Delivery Completion Commit

**Sources**: Delivery, Order, Inventory, Payment, Finance, Notifications.

**Deliverables**:

- Delivery-coordinated completion command from picked-up task.
- Doorstep returned-cylinder field facts for refill exchanges.
- COD cash collected evidence and approved variance linkage where collected
  cash differs from expected COD.
- Same-UoW participant mutations:
  - Delivery task `PICKED_UP -> DELIVERED`;
  - Order `OUT_FOR_DELIVERY -> DELIVERED` and claim completion;
  - Inventory outgoing stock commitment and returned-cylinder
    `INTAKE_PENDING` handoff records;
  - Payment `PENDING_COLLECTION -> COLLECTED`;
  - Finance immutable receipt number and receipt record.
- Post-commit notification requests for out for delivery, payment
  confirmation, and successful delivery where configured.

**Validation**:

- completion requires `acknowledged_cash_collected=true`;
- expected COD derives from the frozen order total and cannot be overridden;
- returned refill cylinder size mismatch fails the whole delivery;
- if any participant rejects, no receipt number is issued and all state remains
  pre-commit;
- identical completion idempotency replay returns the original completed result.

### IP-09 Failed Delivery, Returned Goods, And Intake

**Sources**: Delivery, Order, Inventory, Payment, Finance, Notifications.

**Deliverables**:

- Failed attempt command: delivery task `PICKED_UP -> RETURN_PENDING` while
  Order remains `OUT_FOR_DELIVERY` and Payment remains `PENDING_COLLECTION`.
- Custody resolution by physical outlet return receipt or approved custody
  exception.
- Terminal failed delivery: `RETURN_PENDING -> FAILED`, Order
  `OUT_FOR_DELIVERY -> DELIVERY_FAILED`, Payment zero-collection fact, and
  claim completion.
- Failed delivery fee waiver behavior.
- Inventory return receipt behavior for physically returned outgoing goods.
- Inventory returned-cylinder intake from `INTAKE_PENDING` to
  `INTAKE_CONFIRMED` or `FAILED`, including correction rules.

**Validation**:

- failed attempt creates no zero-collection fact until terminal failure;
- terminal failure is blocked until every picked-up outgoing goods custody row
  is resolved;
- failed deliveries never create customer refund liabilities;
- returned cylinders do not become outlet empty stock until Inventory intake
  confirmation;
- failed intake does not undo delivered orders.

### IP-10 POS Cash Sale And Same-Day Void

**Sources**: POS Sale, Catalog, Inventory, Payment, Finance, Refund, Identity.

**Deliverables**:

- POS draft lifecycle: `DRAFT`, `COMPLETED`, `ABANDONED`, `VOIDED`.
- Outlet-scoped cashier sale creation and completion.
- Catalog-priced POS line snapshots with no delivery fee.
- Same-UoW POS completion:
  - POS `DRAFT -> COMPLETED`;
  - `POS-%08d` number assignment;
  - Inventory outgoing stock commitment and accepted returned-cylinder
    recognition;
  - Payment `POS_CASH_COLLECTED`;
  - Finance outlet cash custody, ledger basis, and immutable receipt.
- Same-outlet, same-business-day, pre-closing POS void with linked reversal
  records across POS, Inventory, Payment, Finance, and receipt audit.

**Validation**:

- POS never creates Cart, Order, Delivery, COD expectation, or Refund state;
- exact UGX cash is required;
- insufficient outlet stock rejects completion and leaves no participant effect;
- same-day void after daily closing is rejected;
- original receipt is immutable and reversal records are appended.

### IP-11 Finance Cash Operations, Daily Closing, Expenses, And Reporting Inputs

**Sources**: Finance, Delivery, Payment, POS, Refund, Inventory.

**Deliverables**:

- Receipt numbering by outlet/date series with no post-commit reuse or rewrite.
- COD cash custody lifecycle: `AGENT_CASH_HELD`,
  `HANDOVER_SUBMITTED`, `HANDOVER_VARIANCE_OPEN`, `OUTLET_CASH_ACCEPTED`.
- COD collection variance approval policy.
- Handover submission and acceptance.
- Outlet daily closing due by 10:00 next operating day, overdue gates, carry
  forward refund liabilities, and summarized records.
- Expense controls, approval threshold, receipt attachment rules, and ledger
  posting.
- Delivery cost reporting inputs and estimated margin labels.
- Finance ledger and cash ledger append-only records.

**Validation**:

- receipt number is issued only inside successful delivery or POS completion
  commit;
- post-commit receipt rollback is impossible;
- open cash discrepancy blocks full shift close;
- overdue closing blocks only the actions named in Finance and allows ordinary
  fulfillment actions;
- Outlet Manager cannot approve their own above-threshold expense.

### IP-12 Refund Liability, Collection Code, And Cash Payout

**Sources**: Refund, Finance, Payment, Notifications, Identity.

**Deliverables**:

- Refund liability creation from valid Finance-owned over-collection correction
  source events.
- Refund states: `LIABILITY_OPEN`, `COLLECTIBLE`, `PAID`, `VOIDED`,
  `WRITTEN_OFF`.
- Single-use collection-code issue and verification.
- Authenticated customer access to collection code without sending code in push,
  SMS, or email.
- Refund-coordinated payout with Finance participant cash custody and ledger
  mutation.
- Super Admin void/write-off with reason, note, and permanent audit.
- Identity snapshot checks for original customer account ID and order phone
  snapshot.

**Validation**:

- cancellation, unclaimable closure, failed delivery, and POS same-day void do
  not create Refund liabilities;
- same collection code cannot be used twice;
- wrong account, phone mismatch, wrong outlet, or missing explicit payout
  permission fails payout closed;
- Finance rejection leaves refund collectible and collection code usable;
- refund payout idempotency replays original payout result.

### IP-13 Notifications

**Sources**: Notifications, Order, Delivery, Payment, Refund.

**Dependency**: Resolve `G-04` before real push delivery.

**Deliverables**:

- Durable notification request table and fanout-attempt table.
- Event-to-channel assignments for launch transactional push messages.
- No-op/local push adapter for non-production until provider is specified.
- Template administration limited to Super Admin.
- Safe message rendering that excludes refund collection codes.

**Validation**:

- notification failure never rolls back source domain state;
- refund collectible notification does not include collection code;
- SMS is used only by Cognito OTP, not Notifications;
- retry and failure records are observable for operational review.

## Test Strategy

Use focused tests per slice and broaden coverage where cross-module atomicity or
boundary contracts are involved.

Required test categories:

- state-machine tests for every lifecycle transition and invalid transition;
- authorization tests for each aggregate boundary using Identity facts plus
  local aggregate policy;
- idempotency tests for covered critical mutations;
- transaction rollback tests for participant failure in order placement, claim,
  delivery completion, POS completion, POS void, and refund payout;
- concurrency tests for stock reservation and receipt-number allocation;
- append-only ledger/audit tests proving corrections append linked records
  rather than editing originals;
- OpenAPI request/response contract tests for every public endpoint;
- migration tests with Postgres Testcontainers;
- event/fanout tests proving post-commit behavior and no external side effects
  inside uncommitted transactions.

## Suggested Ticket Order

1. `docs`: backend module specs, OpenAPI layout, and boundary map for IP-00 to
   IP-03.
2. `feat`: backend skeleton, persistence, UoW, idempotency, audit, and OpenAPI
   validation.
3. `feat`: Identity/authentication foundation.
4. `docs`: resolve outlet configuration/serviceability ownership and
   pending-claim timeout.
5. `feat`: Catalog, delivery-fee quote, and serviceability query primitives.
6. `feat`: Inventory stock core and reservation.
7. `feat`: Cart and order placement with Payment COD expectation.
8. `feat`: Pending pool, claiming, claim block, cancellation, and unclaimable.
9. `feat`: Delivery assignment, pickup, custody, completion, and failed
   delivery.
10. `feat`: Finance receipts, cash custody, daily closing, expenses, and
    reporting inputs.
11. `feat`: POS sale completion and same-day void.
12. `feat`: Refund liability, collection code, and payout.
13. `feat`: Notifications fanout and provider adapter.
14. `test`: end-to-end launch workflow coverage and operational hardening.

## Definition Of Done

For every implementation ticket:

- authoritative backend docs and OpenAPI contracts exist before code;
- at least one meaningful red test or structural check exists before green;
- migrations are forward-only and validated against Postgres;
- mutating commands either require `Idempotency-Key` or document an explicit
  exemption;
- authorization tests prove aggregate-owned boundary decisions, not just grant
  lookup;
- cross-module participant contracts define trigger, payload, consistency,
  idempotency, retry, and failure handling;
- audit, event, ledger, and receipt invariants are covered where the ticket
  touches them;
- `go test ./...`, OpenAPI validation, migration validation, and container build
  pass before PR;
- PR summary includes behavior changes, spec/doc changes, tests run,
  migration/config impact, and `Ticket: #...`.
