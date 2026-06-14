# Business Documentation Review

**Intent**: Record a comprehensive review of `business/` for ambiguity,
contradictions, and correctness risks against the repository operating
guidelines, MVP scope, and modular-monolith ownership rules.

**Review date**: 2026-06-14

**Scope reviewed**:

- `AGENTS.md`
- `start-here/doc-style.md`
- `start-here/modular-monolith-guide.md`
- all files under `business/`
- related scope decisions under `out-of-scope/`

**Review limit**: No separate canonical source file for the cited `section` source
sections exists in this repository. Correctness findings therefore cover
internal correctness against the current repository documents, not validation
against an unavailable upstream business specification.

## Executive Summary

The business set is directionally coherent for a cash-only online COD MVP:
online ordering, global online pricing, outlet claim-time fulfillment, no
mobile-money launch rail, no express delivery, and no partial fulfillment are
consistently repeated.

The main implementation risks are not broad product disagreement. They are
cross-module commit boundaries, missing lifecycle owners, domain policy leakage
into identity/authorization, and source workflows that are named but not
specified. Several sections are precise locally but still force implementers to
guess where a write, approval, retry, exception, or terminal state actually
lives.

## Findings

### F-01 - High - `DELIVERED` commit boundary is not implementation-ready

**Line refs**: `business/order.md:142`, `business/order.md:251`,
`business/delivery.md:24`, `business/payment.md:73`,
`business/finance.md:29`, `business/finance.md:49`

**Evidence**:

- `business/order.md` says delivery completion records COD, returned-cylinder
  facts, task and order terminal state, custody changes, and stock commitment in
  one transaction.
- `business/delivery.md` defines delivery completion as the single commit point
  for delivery task status, order status, outgoing stock commitment,
  returned-cylinder field recording, cash collection recording, and payment
  status.
- `business/payment.md` says delivery completion atomically commits payment
  status with delivery, order, stock, returned-cylinder, and cash facts.
- `business/finance.md` says financial closure is `DELIVERED`, receipts are
  issued at `DELIVERED`, and receipt numbers are permanent and sequential.

**Issue**: The atomic completion group spans Order, Delivery, Inventory,
Payment, and Finance, but no coordinator, participant commands, failure owner,
retry/idempotency behavior, or compensation behavior is named. Finance receipt
issuance is required at `DELIVERED` but is not included in the atomic group.

**Why it matters**: Two teams could implement `DELIVERED` differently: one may
commit order/payment/stock first and issue a receipt asynchronously; another may
make receipt issuance part of the transaction. That changes gaps, rollback,
customer proof of payment, and ledger correctness.

**Recommended fix**: Define the `DELIVERED` workflow as a cross-module contract:
coordinator, participants, ordered commands, commit point, receipt-number
allocation timing, rollback boundary, retry behavior, idempotency keys, and
failure owner. Explicitly state whether receipt issuance and financial ledger
posting are part of the same atomic commit.

### F-02 - High - Cash custody, handover, and variance ownership is split

**Line refs**: `business/order.md:135`, `business/order.md:314`,
`business/delivery.md:251`, `business/payment.md:80`,
`business/payment.md:102`, `business/finance.md:3`

**Evidence**:

- `business/order.md` says Finance owns cash handover, counted-cash acceptance,
  outlet cash custody, variance records, and receipt records.
- `business/delivery.md` defines agent cash handling, short/over collection
  thresholds, approval routing, cash handover as a required live action, and
  supervisor acceptance of counted cash.
- `business/payment.md` says Payment owns collected cash facts and preserves a
  variance link, while short/over collection follows Delivery policy.
- `business/finance.md` does not define a cash custody or handover lifecycle.

**Issue**: Delivery owns the variance policy and field action, Payment depends
on the approved variance to become `COLLECTED`, and Finance owns variance
records and handover, but the canonical lifecycle and write owner are not
specified. The term `supervisor` is used without a canonical persona.

**Why it matters**: COD cash is launch-critical. Ambiguous custody and variance
ownership can produce duplicated variance records, payment states without
handover records, or handover records that cannot be reconciled to payment.

**Recommended fix**: Add a Finance-owned cash custody/handover lifecycle or
explicitly move that lifecycle elsewhere. Define states, actors, approval
owners, variance record ownership, handover commands, Payment link semantics,
and who `supervisor` means in persona terms.

### F-03 - High - Refund liability sources are named but not owned

**Line refs**: `business/refund.md:20`, `business/refund.md:112`,
`business/payment.md:102`, `business/delivery.md:267`,
`business/order.md:326`

**Evidence**:

- `business/refund.md` allows refund liabilities from authorized and posted cash
  over-collection correction and post-collection price adjustment.
- `business/payment.md` says approved post-collection overage correction or
  post-collection price adjustment may create a refund liability.
- `business/delivery.md` records over-collection excess as a cash discrepancy
  reconciled at close.
- `business/order.md` excludes delivery-time price-delta or refund negotiation
  from scope.

**Issue**: The source workflows that create launch refund liabilities are not
specified. No module owns the post-collection price adjustment lifecycle,
over-collection correction decision, source authorization, posting command, or
conditions that convert a discrepancy into a refund liability.

**Why it matters**: Refund depends on a source event but cannot implement or
validate that event. This creates either no path to valid launch refunds or
multiple ad hoc paths to create customer liabilities.

**Recommended fix**: Either define the launch source workflows and their owners
or remove them from launch scope. For each source, name the owner, states,
approval rules, write command, audit fields, and event that creates
`LIABILITY_OPEN`.

### F-04 - High - Domain approval policy is embedded in Identity

**Line refs**: `business/identity-auth.md:209`,
`business/inventory.md:185`, `business/inventory.md:291`,
`start-here/modular-monolith-guide.md:214`

**Evidence**:

- `business/identity-auth.md` defines the detailed inventory adjustment
  threshold: single-unit damage/quarantine may post without separate Super Admin
  approval; losses, positive increases, count variance, missing-source
  adjustments, and deltas greater than one unit require approval.
- `business/inventory.md` references inventory adjustment approval and the
  policy code but does not own the threshold rules.
- `start-here/modular-monolith-guide.md` says authorization owns permissions and
  scope, while domain modules own business eligibility, thresholds, approval
  requirements, state transitions, and policy outcomes.

**Issue**: Identity/Auth is carrying the canonical business policy for
Inventory adjustment thresholds.

**Why it matters**: Implementers could put inventory approval decisions into the
authorization layer, creating the exact policy-placement failure the architecture
guide warns against.

**Recommended fix**: Move canonical inventory adjustment threshold rules to
`business/inventory.md`. Keep Identity limited to who may submit or approve
under the active inventory policy.

### F-05 - High - Inventory ledger ownership is unclear

**Line refs**: `business/finance.md:21`, `business/order.md:141`,
`business/inventory.md:51`, `business/inventory.md:185`

**Evidence**:

- `business/finance.md` states that entries in any financial or inventory ledger
  are never modified or deleted.
- `business/order.md` states inventory and cash ledgers are append-only.
- `business/inventory.md` references inventory ledger posting and movements but
  does not explicitly own the inventory ledger.

**Issue**: The inventory ledger invariant is asserted outside Inventory, while
Inventory does not clearly claim ledger ownership.

**Why it matters**: Ledger ownership is a write-side invariant. If Finance,
Order, and Inventory all enforce or write inventory ledger rules, the boundary
is broken.

**Recommended fix**: State that Inventory owns the inventory ledger and its
append-only invariant. Let Finance own financial/cash ledgers and only reference
Inventory ledger facts as external facts.

### F-06 - High - Refill exchange failure states are orphaned

**Line refs**: `business/delivery.md:46`, `business/delivery.md:101`,
`business/delivery.md:229`, `business/order.md:283`

**Evidence**:

- `business/delivery.md` defines the exchange field leg as
  `PENDING -> IN_PROGRESS -> RETURN_RECORDED -> INTAKE_PENDING`, plus
  `CANCELLED`.
- The same file says missing, damaged, unacceptable, unexpected-vendor, or
  size-mismatch returns are controlled field facts and can cause the whole
  delivery to fail.
- `business/order.md` says failed delivery after pickup marks the order
  `DELIVERY_FAILED` and records zero collection.

**Issue**: The exchange request lifecycle does not say what state each exchange
request reaches when a refill delivery fails after pickup because the returned
cylinder is missing, unacceptable, unexpected, or size-mismatched.

**Why it matters**: A failed refill delivery can leave exchange requests stuck
  in `IN_PROGRESS` or `RETURN_RECORDED`, while the order is terminal.

**Recommended fix**: Add Delivery-owned terminal or exception states for failed
field-leg outcomes, or explicitly define how `RETURN_RECORDED` plus order
`DELIVERY_FAILED` is represented and queried.

### F-07 - High - Unclaimable `PENDING` orders have no owner or outcome

**Line refs**: `business/cart.md:95`, `business/catalog.md:75`,
`business/catalog.md:120`, `business/order.md:186`,
`business/order.md:217`

**Evidence**:

- `business/cart.md`, `business/catalog.md`, and `business/order.md` deliberately
  avoid per-outlet stock and vendor filtering before placement.
- `business/catalog.md` says outlets may restrict accepted incoming vendors at
  claim time and the claiming outlet must accept frozen incoming vendor terms.
- `business/order.md` says no automatic outlet allocation, ranking, or
  outlet-change chain is defined.

**Issue**: A placed order can be valid at coarse serviceability time but
unclaimable by every outlet because of stock or vendor constraints. No timeout,
escalation, customer cancellation prompt, staff cancellation path, or owner is
defined.

**Why it matters**: `PENDING` is not terminal. Without a resolution path, the
system can accumulate valid but operationally impossible orders.

**Recommended fix**: Define an Order-owned unresolved-pending policy, even if
minimal: detection owner, allowed staff/customer actions, notification behavior,
timeout if any, cancellation attribution, and whether the order remains
customer-cancellable indefinitely.

### F-08 - High - Delivery-fee override authority conflicts with global online fee rules

**Line refs**: `business/delivery.md:287`, `business/delivery.md:313`,
`business/catalog.md:188`, `business/identity-auth.md:264`,
`out-of-scope/2026-06-14-express-delivery-fee.md:35`

**Evidence**:

- `business/delivery.md` says online delivery fee is computed from the resolved
  delivery address and online fee policy, not the eventual claiming outlet.
- `business/catalog.md` says local outlet price-rule configuration does not
  affect online COD totals unless an approved global online pricing policy uses
  it before placement.
- `business/identity-auth.md` says `Outlet price rules within guardrail` covers
  delivery-fee overrides.
- `business/delivery.md` defines delivery-fee guardrails, while express delivery
  fee behavior is deferred.

**Issue**: Outlet-scoped delivery-fee override permission exists, but launch
online delivery fee behavior is global-only and no non-online delivery-fee
workflow is in scope.

**Why it matters**: Implementers may build outlet delivery-fee overrides that
silently do nothing, affect online totals incorrectly, or create hidden
non-MVP pricing behavior.

**Recommended fix**: Decide whether outlet delivery-fee overrides are launch
scope. If not, remove them from Identity and Delivery. If yes, define the exact
pricing basis, scope, guardrails, approval owner, and whether they can affect
online COD placement.

### F-09 - Medium - Task-only delivery cancellation lacks authorization and command shape

**Line refs**: `business/order.md:66`, `business/order.md:266`,
`business/delivery.md:78`, `business/delivery.md:137`,
`business/identity-auth.md:157`

**Evidence**:

- `business/order.md` includes delivery task cancellation before pickup in the
  required cancellation-attribution invariant and alternate paths.
- `business/delivery.md` includes `READY_FOR_PICKUP|ASSIGNED -> CANCELLED` and
  says task-only cancellation allows replacement task creation or assignment.
- The access matrix does not contain a delivery task cancellation capability.

**Issue**: The lifecycle exists, but actor authority, command name, required
reason fields, replacement-task creation command, and owning module contract are
not defined.

**Recommended fix**: Add a Delivery-owned pre-pickup task cancellation command,
permissions, attribution fields, replacement-task behavior, and failure cases.

### F-10 - Medium - Permission tables drift from the canonical matrix

**Line refs**: `business/cart.md:153`, `business/order.md:332`,
`business/identity-auth.md:157`, `business/notifications.md:1`

**Evidence**:

- `business/cart.md` lists `Cart cleanup policy administration`, but
  `business/identity-auth.md` does not.
- `business/order.md` lists `Cancellation`, but `business/identity-auth.md` does
  not.
- `business/identity-auth.md` lists notification template administration and
  customer notification requests, but `business/notifications.md` has no
  permissions section for them.

**Issue**: Aggregate files say their tables are trimmed rows from the full
matrix, but several rows are not present on both sides.

**Recommended fix**: Reconcile every aggregate-specific permission row with the
canonical Identity matrix. Add missing rows or remove non-canonical trimmed
rows.

### F-11 - Medium - Outlet Manager refund payout permission is ambiguous

**Line refs**: `business/refund.md:199`, `business/refund.md:217`,
`business/identity-auth.md:93`, `business/identity-auth.md:185`,
`business/identity-auth.md:234`, `business/identity-auth.md:408`

**Evidence**:

- `business/refund.md` grants P-06 `Scoped` refund payout.
- `business/identity-auth.md` grants P-06 `Scoped` refund payout.
- `business/refund.md` edge case E-07 says Outlet Managers may disburse
  collectible refunds when explicitly permissioned.
- `business/identity-auth.md` persona text also says Outlet Managers disburse
  collectible refunds when explicitly permissioned.

**Issue**: The matrix reads as inherent scoped P-06 authority, while prose says
explicit permission is required.

**Recommended fix**: Decide whether refund payout is inherent to P-06 or a
separate explicit permission bundle. Use the same wording in the matrix, persona
text, and Refund permissions.

### F-12 - Medium - Payment expectation creation and cancellation retirement are underspecified

**Line refs**: `business/payment.md:44`, `business/payment.md:85`,
`business/order.md:180`

**Evidence**:

- `business/payment.md` defines `PENDING_COLLECTION` as active COD expectation
  derived from the placed order.
- `business/order.md` says order placement creates `PENDING`.
- `business/payment.md` says pre-collection cancellation creates no terminal
  payment state and retires the COD expectation through Order cancellation.

**Issue**: The creation trigger for the Payment record or expectation is not
specified as part of Order placement, and the cancellation-retirement contract
is not defined.

**Recommended fix**: Define Order-to-Payment contract semantics for COD
expectation creation, idempotency, cancellation retirement, query behavior for
cancelled orders, and whether `PENDING_COLLECTION` is durable.

### F-13 - Medium - Overdue-closing restrictions reference undefined workflows

**Line refs**: `business/finance.md:110`, `business/finance.md:161`,
`business/identity-auth.md:209`, `business/catalog.md:207`

**Evidence**:

- `business/finance.md` blocks large inventory adjustments, manual financial
  ledger adjustments, above-threshold expense approval, and outside-guardrail
  price changes while closing is overdue unless overrides apply.
- Inventory defines adjustment approval thresholds in Identity, not Inventory.
- Manual financial ledger adjustment is not specified as a workflow.
- Catalog only specifies outside-guardrail price activation blocking.

**Issue**: Finance names blocked actions and override concepts that are not
fully defined by the owning domains.

**Recommended fix**: For each blocked action, add the owning module, exact
capability, blocked states, override actor, override evidence, and whether the
restriction blocks submission, approval, activation, or ledger posting.

### F-14 - Medium - Field-proof policy is active but ownerless

**Line refs**: `business/delivery.md:240`, `business/order.md:326`,
`business/finance.md:3`

**Evidence**:

- `business/delivery.md` says GPS, photo, and note requirements are controlled
  by active field-proof policy.
- The same file says required live actions include cash handover.
- `business/order.md` puts doorstep code fallback and evidence-retention detail
  out of scope.
- No module owns field-proof policy configuration, retention, or enforcement.

**Issue**: Field proof gates launch delivery actions, but policy ownership and
minimum launch requirements are undefined.

**Recommended fix**: Either define a minimal Delivery-owned launch field-proof
policy or explicitly state that all GPS/photo/note requirements are disabled at
launch except authoritative timestamps and live submission.

### F-15 - Medium - Intake correction can be read as contradicting delivery size rules

**Line refs**: `business/delivery.md:46`, `business/inventory.md:223`

**Evidence**:

- `business/delivery.md` says a refill size mismatch blocks successful delivery
  and the whole delivery fails.
- `business/inventory.md` says a returned-cylinder intake actor may correct an
  agent recording mismatch in returned-cylinder vendor or size.

**Issue**: The correction text does not clearly distinguish correcting a data
entry error from accepting a physical size mismatch after successful delivery.

**Recommended fix**: State that intake correction may correct erroneous field
recording only when the physical exchange satisfied the accepted delivery facts.
Actual physical size mismatch must remain a failed delivery or failed intake
exception, as applicable.

### F-16 - Medium - Stock count behavior lacks a lifecycle and posting contract

**Line refs**: `business/inventory.md:253`,
`business/identity-auth.md:217`, `business/identity-auth.md:225`

**Evidence**:

- `business/inventory.md` says stock counts do not freeze operations, expected
  quantities are fixed at count start, ledger movements during the count window
  are tracked, and variance is calculated at close.
- `business/identity-auth.md` says count variance creates audited inventory
  adjustments and follows approval thresholds.

**Issue**: Count states, close command, variance posting owner, approval path,
and conflict behavior during ongoing operations are not specified.

**Recommended fix**: Add an Inventory-owned stock count lifecycle with start,
active, close, variance proposal, approval/posting, cancellation, and audit
rules.

### F-17 - Medium - Transfer and vendor-refill exception paths are incomplete

**Line refs**: `business/inventory.md:98`, `business/inventory.md:156`

**Evidence**:

- `business/inventory.md` defines happy-path outlet transfer and vendor refill
  movement states.
- Transfer has no cancellation, failed dispatch, partial receipt, loss, or
  receiver rejection path.
- Vendor refill handles shortage/overage through adjustment but does not define
  cancellation, lost-in-refill, damaged return, or never-returned handling.

**Issue**: These are physical stock movement lifecycles, but failure ownership
and terminal exception states are incomplete.

**Recommended fix**: Add minimal exception paths for launch operations or state
that exception handling is performed only through named Inventory adjustment
records with non-available stock retained until resolution.

### F-18 - Medium - Notification contracts are not implementable

**Line refs**: `business/notifications.md:11`,
`business/notifications.md:28`, `business/order.md:233`,
`business/identity-auth.md:192`

**Evidence**:

- `business/notifications.md` says notifications are side effects of domain
  events and failed notifications do not block committed activity.
- `business/order.md` says ready-order notification fanout is attempted.
- No file defines notification event payloads, idempotency, retry, failure
  retention, template owner, or provider/consumer contracts.

**Issue**: The boundary is directionally correct, but the event-to-notification
contract is too incomplete for implementation.

**Recommended fix**: Add notification contracts for each launch event: source
event, producer, consumer, payload basis, channel, idempotency key, retry
policy, failure owner, and audit/operational review behavior.

### F-19 - Medium - `out-of-scope` references walk-in POS as launch behavior while business excludes counter-sale

**Line refs**: `out-of-scope/2026-06-13-mobile-money-payments.md:13`,
`out-of-scope/2026-06-13-mobile-money-payments.md:212`,
`business/order.md:318`, `business/README.md:14`

**Evidence**:

- `out-of-scope/2026-06-13-mobile-money-payments.md` says launch payment behavior
  includes immediate cash for walk-in POS sales and that walk-in POS customers
  pay cash at the outlet.
- `business/order.md` lists counter-sale workflows as out of scope.
- `business/README.md` orients the domain around online outlet delivery orders.

**Issue**: Related scope documentation implies a launch POS cash workflow, but
the business docs exclude counter-sale workflows.

**Recommended fix**: Clarify the out-of-scope mobile-money note: either walk-in
POS cash is not in the current business scope, or add the missing POS business
documents before treating it as launch behavior.

### F-20 - Medium - Cart retained summary is undefined

**Line refs**: `business/cart.md:144`, `business/finance.md:66`

**Evidence**:

- `business/cart.md` says abandoned cart detail is removed after 90 days plus
  30 days, and a safe cart summary remains.
- No file defines which fields are safe, who owns retention/redaction, or how
  this interacts with audit records and customer data.

**Issue**: Retention language is normative but not implementable.

**Recommended fix**: Define the summary fields, prohibited fields, retention
owner, deletion trigger, audit preservation rule, and customer-data rationale.

### F-21 - Medium - Refund identity verification uses ambiguous phone source

**Line refs**: `business/refund.md:148`,
`business/identity-auth.md:304`

**Evidence**:

- `business/refund.md` requires payout actors to confirm that the
  customer/order phone matches before cash is marked paid.
- Identity allows account recovery and credential changes, and customers may
  authenticate by phone.

**Issue**: It is unclear whether payout checks the current account phone, order
snapshot phone, customer profile phone, or some combination after phone changes
or recovery.

**Recommended fix**: Define the exact identity facts used at payout and the
exception path when current account phone and order snapshot phone differ.

### F-22 - Medium - Catalog-change acknowledgement lacks version semantics

**Line refs**: `business/cart.md:103`, `business/cart.md:127`,
`business/catalog.md:243`

**Evidence**:

- `business/cart.md` requires customer acknowledgement for catalog or price
  changes before quote or checkout.
- It does not say which catalog version, bundle composition version, price rule
  version, or acknowledgement timestamp must be stored.

**Issue**: Acknowledgement can be implemented as a generic flag, which may stay
true across later price or bundle changes.

**Recommended fix**: Define acknowledgement keys and invalidation behavior:
price rule version, bundle composition version, affected line IDs, timestamp,
and re-acknowledgement trigger.

### F-23 - Low - Persona numbering skips P-07 without explanation

**Line refs**: `business/identity-auth.md:33`,
`business/identity-auth.md:109`, `business/identity-auth.md:118`

**Evidence**:

- `business/identity-auth.md` defines P-01 through P-06 and P-08 through P-10.
- The document says there are nine personas.

**Issue**: The count is correct, but the skipped ID can look like a missing
persona or stale removal.

**Recommended fix**: Add a note that P-07 is intentionally reserved/removed, or
renumber only if no external references depend on the IDs.

### F-24 - Low - Future notification classification is mentioned but not enumerated

**Line refs**: `business/notifications.md:21`,
`business/notifications.md:28`

**Evidence**:

- `business/notifications.md` says notification types are classified as
  transactional or non-transactional for future policy use.
- The channel assignment table does not label events by classification.

**Issue**: This is low risk because preferences are out of scope, but the text
creates a classification concept without a table.

**Recommended fix**: Add a small classification column or remove the sentence
until the policy matters.

## Areas That Look Internally Consistent

- Cash-only online COD launch scope is consistently represented in
  `business/order.md`, `business/payment.md`, `business/refund.md`,
  `business/finance.md`, `business/notifications.md`, and
  `out-of-scope/2026-06-13-mobile-money-payments.md`, aside from the walk-in POS
  scope note in F-19.
- Express delivery fee behavior is consistently deferred in
  `business/delivery.md` and
  `out-of-scope/2026-06-14-express-delivery-fee.md`.
- Global online pricing before order placement is consistently represented
  across Cart, Catalog, Order, and README, aside from the outlet delivery-fee
  override ambiguity in F-08.
- Whole-order, all-or-nothing fulfillment is consistently repeated across
  Catalog, Order, Delivery, and Inventory.
- Returned-cylinder inventory recognition after intake, not at doorstep
  delivery, is consistently represented across Delivery, Inventory, README, and
  Order main flow.

## Recommended Fix Order

1. Resolve F-01, F-02, and F-03 first. They define core COD financial closure,
   cash custody, and refund correctness.
2. Resolve F-04 and F-05 next to restore domain-policy and ledger ownership.
3. Resolve F-06 and F-07 to prevent orphaned operational states.
4. Resolve F-08 through F-13 to align pricing authority, permissions, and
   operational gates.
5. Resolve F-14 through F-24 as implementation-readiness hardening.
