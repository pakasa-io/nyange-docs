# Support

**Intent**: Define the support case lifecycle, support action boundary, queue and
routing rules, and operational risk alert behavior.

**Reader task**: Use this document to decide what support can observe, request,
route, or close without accidentally moving ownership of order, payment,
delivery, inventory, refund, or finance workflows into Support.

**Sources**: §6.7 Support Case Lifecycle, §7.16 Operational Risk Alerts

**Related**:
[identity-auth.md](identity-auth.md) for the full access matrix;
[order.md](order.md), [payment.md](payment.md), [delivery.md](delivery.md),
[refund.md](refund.md), and [finance.md](finance.md) for linked domain records.

## Boundary

- Support owns support cases, support notes, assignment history, support action
  requests, and support-case closure.
- Support cases may link to domain records.
- Support cases do not own linked order, payment, delivery, inventory, refund,
  finance, notification, or ledger records.
- Customer Support Agents can request approved workflows in linked domains, but
  they do not execute those workflows except for explicitly permissioned
  fallback actions in the access matrix.

## Support Case Lifecycle

```
OPEN
  ├─► IN_PROGRESS
  │     ├─► WAITING_ON_CUSTOMER
  │     │     └─► IN_PROGRESS
  │     ├─► WAITING_ON_INTERNAL_TEAM
  │     │     └─► IN_PROGRESS
  │     └─► READY_FOR_SUPPORT_REVIEW
  │           └─► CLOSED
  └─► CANCELLED
```

| State | Meaning |
| --- | --- |
| `OPEN` | Case created and awaiting work. |
| `IN_PROGRESS` | Queue or individual owner is investigating. |
| `WAITING_ON_CUSTOMER` | Support needs customer response. |
| `WAITING_ON_INTERNAL_TEAM` | Support awaits domain workflow outcome, such as refund or adjustment. |
| `READY_FOR_SUPPORT_REVIEW` | Blocking action requests are terminal and no customer response remains. |
| `CLOSED` | Terminal manual closure by permissioned support-case actor with resolution code, resolution note, and required customer communication summary. |
| `CANCELLED` | Terminal case opened in error or no longer needed; requires reason. |

## Case Rules

### Visibility

- Support cases are internal records.
- Customers receive notifications and outcomes through direct channels, such as
  push and email.
- Customers do not access case records, internal notes, owner assignments, SLA
  timers, linked entity history, or risk context.

### Scope and Structure

- Support cases are structured but lightweight at launch.
- Supported case uses include customer-facing issues, complaints, refund
  exceptions, reassignment disputes, payment exceptions, delivery exceptions,
  operational escalations, risk-alert follow-up, and cross-domain follow-up.
- Support cases are not required for every exception.
- Each case tracks type, status, priority, queue, optional individual owner,
  linked entities, notes, SLA timestamps, assignment history, and status history.

### Escalation

- Support escalation is represented through case type, priority, queue/owner
  coverage, linked records, and notifications.
- `ESCALATED` is not a support case status at launch.

### Domain Record Linkage

- Support cases link to domain records but do not own those records.
- A support case may link to multiple domain records when the issue spans
  related orders, payments, deliveries, refunds, or other scoped entities.

### Notes and Communication Summaries

- Support-case notes are append-only.
- Corrections are new notes, not edits to prior notes.
- Support cases may store concise customer communication summaries.
- Support cases do not act as a two-way inbox at launch.
- Support cases do not store raw recordings, full transcripts, chat archives, or
  standalone evidence attachments at launch.
- Customer communication summaries link to relevant notification logs or
  existing domain records where applicable.

### Action Requests

Support action request states:
`REQUESTED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, `CANCELLED`.

- `COMPLETED`, `REJECTED`, and `CANCELLED` are terminal for review-readiness.
- A support action request does not alter linked domain records by itself.
- The authorized owner of the affected domain workflow executes and records the
  requested workflow change.
- For refunds, support may create a refund-request action request with linked
  order, payment, refund context, and notes.
- A refund-request action request does not create, approve, pay, void, write
  off, or ledger-post the refund by itself.
- Linked domain workflow completion adds a support-case note.
- Linked workflow completion may move the case to
  `READY_FOR_SUPPORT_REVIEW` only when no blocking support action requests
  remain.
- A case is not ready for support review while a customer response is still
  required.
- Non-blocking notes or linked records do not prevent review readiness.
- The case cannot close until a permissioned support actor records the required
  closure decision.

### No Goodwill Credits

- Goodwill credits are not supported at launch.
- Support outcomes use approved refunds or adjustments tied to orders.

### Customer Communications

- Support-triggered customer communications must use approved transactional
  notifications and templates with safe structured parameters.
- Support-case permissions do not grant notification-template administration.
- Support does not send arbitrary freeform outbound messages from the case
  workflow.

### Case Creation

- Support cases are created manually at launch.
- Operational alerts, risk alerts, closure blockers, payment exceptions,
  delivery exceptions, and refund exceptions may supply default case context.
- Default context may include case type, linked entities, priority suggestion,
  summary, and supporting context.
- No case exists until a permissioned Customer Support Agent, in-scope Outlet
  Manager, or Super Admin explicitly creates it.

## Queue and Routing

- Cases enter a centrally configured default queue based on type, priority, and
  outlet scope.
- Queue membership is derived from the actor's support permissions and outlet
  scope.
- Individual owner assignment is optional.
- Assigning an individual owner does not remove queue coverage.
- When no individual owner is assigned or the owner is unavailable, the queue
  remains accountable for coverage.
- Permissioned Customer Support Agents, Outlet Managers, or Super Admins may
  manually move a case only to another allowed queue and must record a reason.
- Queue moves and individual owner changes are append-only assignment history
  events with actor, previous/new ownership, reason, and timestamp.
- Assignment changes affect support coverage and accountability only.
- Assignment changes do not mutate linked orders, payments, refunds, delivery,
  inventory, or financial ledger records.
- Individual owner assignment sends a best-effort internal notification to that
  owner and records the notification timestamp.
- Queue-owned cases rely on queue dashboards and SLA/priority views instead of
  notifying every queue member.

## Outlet Scope

- Cases are outlet-scoped.
- A case's outlet scope is determined by its primary outlet and linked domain
  records.
- Customer Support Agents cannot view cases for an outlet they are not assigned
  to unless granted explicit cross-outlet access.
- Cases linked to multiple outlets are not partially visible at launch.
- A Customer Support Agent, Outlet Manager, or Super Admin must have permission
  for all linked outlets, or explicit cross-outlet/global support access, to
  view or manage a multi-outlet case.

## Launch Routing Constraints

- Launch support routing uses default queues and permissioned manual moves only.
- Workload balancing, shift scheduling, round-robin assignment, skill-based
  routing, and complex support rosters do not grant case ownership, visibility,
  or routing authority at launch.

## Queue Configuration

- Queue definitions are global configuration only.
- Only Super Admins or actors with explicit global support queue-configuration
  permission may create, edit, deactivate, reprioritize, or change routing
  structures.
- Outlet Managers may work cases within scope.
- Outlet Managers with explicit assignment permission may move cases between
  allowed existing queues.
- Outlet Managers request new queues, routing-structure changes, or per-outlet
  queue behavior only through Customer Support Agent or Super Admin escalation.
- Outlet Managers do not define or manage queue configuration directly.
- No per-outlet queue-configuration approval path exists at launch.
- Queue configuration changes require audit with actor, before/after values,
  reason, and timestamp.

## Sensitive Reads

- Detailed support-case views are sensitive reads.
- Detailed views must be audit-logged with actor, case ID, outlet scope, and
  timestamp.
- Aggregate queue counts and list views do not require per-case read audit at
  launch.

## SLA Timers

- Support-case SLA timers are advisory only.
- SLA timers can drive notifications and dashboards.
- Urgent unowned or overdue cases may notify permissioned Customer Support
  Agents, Outlet Managers, or Super Admins according to priority and scope.
- SLA timers do not block or advance orders, payments, refunds, deliveries,
  inventory, daily closing, financial closure, or ledger posting.

## Retention

- Case records have their own retention window, independent of subject domain
  records.
- At launch, support-derived records are retained for 36 months after the case
  reaches `CLOSED` or `CANCELLED`.
- Open support cases are not purged by age.
- The retention countdown starts only after `CLOSED` or `CANCELLED`.
- Support retention applies only to support-derived records.
- Support retention must not delete or weaken linked orders, payments, refunds,
  deliveries, financial ledger records, ledger entries, notification logs,
  closure blockers, operational risk alerts, or audit logs.

## Operational Risk Alerts

Repeated custody/cash exceptions by delivery agents and staff are evaluated
against rolling-window thresholds.

Evaluated exception types include:

- repeated short collections;
- missing returned cylinders;
- cash discrepancies;
- custody losses;
- forced-closure adjustments;
- unresolved custody exceptions.

Thresholds start from global defaults and may define outlet and role overrides.

### Risk Alert Boundary

Risk alerts are informational only. They do not:

- suspend or block the flagged user by themselves;
- block delivery assignment;
- alter assignment ranking;
- deduct from pay;
- assign personal financial liability;
- create a support case without a manual Outlet Manager or Super Admin decision.

### Visibility

- Alerts are visible only to permissioned Outlet Managers and Super Admins
  within authorized scope.
- Alert notifications, when configured, use push and are limited to those same
  permissioned actors.
- Alerts are not customer-facing.
- Alerts are not sent to the flagged staff member or agent.
- The flagged staff member or agent does not see the alert.

### Risk Alert Lifecycle

States: `OPEN`, `ACKNOWLEDGED`, `DISMISSED`, `CASE_OPENED`.

- Acknowledging or dismissing an alert requires note/reason and audit.
- When an Outlet Manager for that outlet or a Super Admin chooses to open a
  support case from a risk alert, the alert may supply default case context.
- Default risk-alert case context may include subject user, alert type, and
  trigger details.
- No support case exists until that actor confirms creation.
- Once the manual case is created, the case is permanently linked to the alert.
- The alert transitions to `CASE_OPENED`.
- If work is manually assigned despite an active relevant risk alert, the
  assigning Outlet Manager or Super Admin must record a reason retained in
  audit.

### Risk Alert Sensitive Reads

- Detailed risk-alert views are sensitive reads.
- Detailed views are audit-logged with actor, outlet scope, alert ID, and
  timestamp.
- Aggregate dashboard counts and list views do not require per-alert read audit
  at launch.

### Risk Alert Retention

- Active alerts remain in review until dismissed, escalated, or reviewed with
  required note/reason and audit.
- Eligible dismissed derived alerts are summarized after the active staff-risk
  retention window.
- The active staff-risk retention window is 24 months at launch.
- `CASE_OPENED` alerts follow the linked support-case closure and retention path
  first, then use the risk-alert retention window.
- Risk-alert retention must not delete underlying custody events, delivery runs,
  cash variances, forced closures, ledger adjustments, or audit logs.

## Permissions

Trimmed access matrix rows relevant to support. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-06 | P-07 | P-10 |
| --- | --- | --- | --- |
| Support case creation | Scoped | Scoped | Full |
| Support case management | Scoped | Scoped | Full |
| Support queue configuration | - | - | Full |
| Support communication logging | Scoped | Scoped | Full |
| Customer notification requests | Scoped approved transactional only | Scoped approved transactional only | Full |
| Support action requests create/request | Scoped | Scoped | Full |
| Support action requests execute | Scoped | - | Full |
| Operational risk alerts | Scoped with explicit permission | - | Full |

## Authorization Edge Cases

**E-03**: Customer Support Agents cannot view another outlet's cases unless a
Super Admin has granted explicit cross-outlet support access. There is no
implicit cross-outlet access based on case type or priority.
