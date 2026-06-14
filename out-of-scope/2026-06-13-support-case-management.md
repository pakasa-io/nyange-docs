# Support Case Management

## Status

- Status: Deferred
- Date Added: 2026-06-13
- Source: Product scope decision in repository update request
- Owner: Product Manager

## Summary

Support case management is deferred from the mid-stage MVP. Launch support uses
direct owning-domain workflows, audited fallback actions, approved
transactional notifications, and operational escalation instead of a dedicated
case management system.

## Proposed Scope

If moved back into scope, support case management may include the capabilities
below.

### Boundary

- Support cases would be internal records.
- Support cases could link to domain records, but would not own linked order,
  payment, delivery, inventory, refund, finance, notification, or ledger
  records.
- Customer Support Agents could request approved workflows in linked domains,
  but would not execute those workflows except for explicitly permissioned
  fallback actions.
- The authorized owner of the affected domain workflow would execute and record
  requested workflow changes.

### Lifecycle

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

- `OPEN`: case created and awaiting work.
- `IN_PROGRESS`: queue or individual owner is investigating.
- `WAITING_ON_CUSTOMER`: support needs customer response.
- `WAITING_ON_INTERNAL_TEAM`: support awaits a domain workflow outcome.
- `READY_FOR_SUPPORT_REVIEW`: blocking action requests are terminal and no
  customer response remains.
- `CLOSED`: terminal manual closure by a permissioned support-case actor, with
  resolution code, resolution note, and required customer communication summary.
- `CANCELLED`: terminal case opened in error or no longer needed; requires
  reason.

### Case Records

- Customers would receive notifications and outcomes through approved channels,
  but would not access internal case records.
- Customers would not see internal notes, owner assignments, SLA timers, linked
  entity history, or risk context.
- Case records could track type, status, priority, queue, optional individual
  owner, linked entities, notes, SLA timestamps, assignment history, and status
  history.
- Supported case uses could include customer-facing issues, complaints, refund
  exceptions, reassignment disputes, payment exceptions, delivery exceptions,
  operational escalations, risk-alert follow-up if operational risk alerts also
  enter scope, and cross-domain follow-up.
- Cases would not be required for every exception.

### Domain Record Linkage

- A case could link to one or more domain records.
- Linked records could include orders, payments, deliveries, refunds, inventory
  exceptions, operational risk alerts if those also enter scope, and other
  scoped entities.
- A support case would not mutate linked domain records by itself.

### Notes and Communication Summaries

- Support-case notes would be append-only.
- Corrections would be new notes, not edits to prior notes.
- Case records could store concise customer communication summaries.
- Case records would not act as a two-way inbox.
- Case records would not store raw recordings, full transcripts, chat archives,
  or standalone evidence attachments.
- Customer communication summaries could link to notification logs or existing
  domain records.

### Action Requests

Support action request states could be:
`REQUESTED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, `CANCELLED`.

- `COMPLETED`, `REJECTED`, and `CANCELLED` would be terminal for support review
  readiness.
- A support action request would not alter linked domain records by itself.
- For refunds, support could create a refund-request action request with linked
  order, payment, refund context, and notes.
- A refund-request action request would not create, approve, pay, void, write
  off, or ledger-post the refund by itself.
- Linked domain workflow completion could add a support-case note.
- Linked workflow completion could move the case to
  `READY_FOR_SUPPORT_REVIEW` only when no blocking action requests remain.
- A case would not be ready for support review while a customer response is
  still required.
- Non-blocking notes or linked records would not prevent review readiness.
- The case could not close until a permissioned support actor records the
  required closure decision.

### Customer Communications

- Support-triggered customer communications would use approved transactional
  notifications and templates with safe structured parameters.
- Support-case permissions would not grant notification-template
  administration.
- Support would not send arbitrary freeform outbound messages from the case
  workflow.

### Goodwill Credits

- Goodwill credits would remain excluded unless separately approved.
- Support outcomes would use approved refunds or adjustments tied to orders.

### Case Creation

- Cases could be created manually.
- Operational alerts, risk alerts if in scope, closure blockers, payment
  exceptions, delivery exceptions, and refund exceptions could supply default
  case context.
- Default context could include case type, linked entities, priority suggestion,
  summary, and supporting context.
- No case would exist until a permissioned Customer Support Agent, in-scope
  Outlet Manager, or Super Admin explicitly creates it.

### Queue and Routing

- Cases could enter a centrally configured default queue based on type, priority,
  and outlet scope.
- Queue membership could derive from the actor's support permissions and outlet
  scope.
- Individual owner assignment could be optional.
- Assigning an individual owner would not remove queue coverage.
- When no individual owner is assigned or the owner is unavailable, the queue
  would remain accountable for coverage.
- Permissioned actors could manually move a case only to another allowed queue
  and would record a reason.
- Queue moves and individual owner changes would be append-only assignment
  history events with actor, previous/new ownership, reason, and timestamp.
- Assignment changes would affect support coverage and accountability only.
- Assignment changes would not mutate linked orders, payments, refunds,
  delivery, inventory, or financial ledger records.
- Individual owner assignment could send a best-effort internal notification to
  that owner and record the notification timestamp.
- Queue-owned cases could rely on queue dashboards and SLA/priority views
  instead of notifying every queue member.

### Outlet Scope

- Cases would be outlet-scoped.
- A case's outlet scope would be determined by its primary outlet and linked
  domain records.
- Customer Support Agents could not view cases for an outlet they are not
  assigned to unless granted explicit cross-outlet access.
- Cases linked to multiple outlets would not be partially visible.
- A Customer Support Agent, Outlet Manager, or Super Admin would need permission
  for all linked outlets, or explicit cross-outlet/global support access, to
  view or manage a multi-outlet case.

### Routing Constraints

- Initial routing could use default queues and permissioned manual moves only.
- Workload balancing, shift scheduling, round-robin assignment, skill-based
  routing, and complex support rosters would not grant case ownership,
  visibility, or routing authority unless separately designed and approved.

### Queue Configuration

- Queue definitions would be global configuration.
- Only Super Admins or actors with explicit global support queue-configuration
  permission could create, edit, deactivate, reprioritize, or change routing
  structures.
- Outlet Managers could work cases within scope.
- Outlet Managers with explicit assignment permission could move cases between
  allowed existing queues.
- Outlet Managers would request new queues, routing-structure changes, or
  per-outlet queue behavior through escalation.
- Outlet Managers would not define or manage queue configuration directly unless
  a later policy grants that authority.
- Queue configuration changes would require audit with actor, before/after
  values, reason, and timestamp.

### Sensitive Reads

- Detailed support-case views would be sensitive reads.
- Detailed views would be audit-logged with actor, case ID, outlet scope, and
  timestamp.
- Aggregate queue counts and list views would not require per-case read audit
  unless future policy says otherwise.

### SLA Timers

- Support-case SLA timers would be advisory unless future policy makes them
  workflow gates.
- SLA timers could drive notifications and dashboards.
- Urgent unowned or overdue cases could notify permissioned Customer Support
  Agents, Outlet Managers, or Super Admins according to priority and scope.
- SLA timers would not block or advance orders, payments, refunds, deliveries,
  inventory, daily closing, financial closure, or ledger posting.

### Retention

- Case records would have their own retention window, independent of subject
  domain records.
- Candidate retention: 36 months after the case reaches `CLOSED` or
  `CANCELLED`.
- Open cases would not be purged by age.
- The retention countdown would start only after `CLOSED` or `CANCELLED`.
- Support retention would apply only to support-derived records.
- Support retention must not delete or weaken linked orders, payments, refunds,
  deliveries, financial ledger records, ledger entries, notification logs,
  closure blockers, in-scope operational risk alerts, or audit logs.

### Operational Risk Alert Integration

If operational risk alerts also move back into scope, risk-alert integration
could add `CASE_OPENED` to the risk alert lifecycle.

- When an Outlet Manager for that outlet or a Super Admin chooses to open a case
  from a risk alert, the alert could supply default case context.
- Default risk-alert case context could include subject user, alert type, and
  trigger details.
- No case would exist until the actor confirms creation.
- Once the manual case is created, the case would be permanently linked to the
  alert.
- The alert would transition to `CASE_OPENED`.
- `CASE_OPENED` alerts would follow the linked support-case closure and
  retention path first, then the risk-alert retention window.

### Permission Rows

If moved back into scope, the access matrix would need explicit rows for:

- support case creation;
- support case management;
- support queue configuration;
- support communication logging;
- support action requests create/request;
- support action requests execute.

### Authorization Edge Case

A deferred authorization edge case would be:

- Customer Support Agents cannot view another outlet's cases unless a Super
  Admin has granted explicit cross-outlet support access.
- There is no implicit cross-outlet access based on case type or priority.

## Why It Is Out of Scope

Support case management adds a separate workflow, queue model, ownership model,
retention policy, sensitive-read policy, cross-domain action request layer, and
configuration surface. Those concerns exceed the mid-stage MVP because the core
product can operate with direct owning-domain workflows, explicit permissions,
audit records, and approved notification paths.

Deferral protects module ownership boundaries. Orders, payments, delivery,
inventory, refunds, and finance remain the only owners of their mutable state
transitions during MVP.

## Impact of Deferral

- Launch has no support case inbox, case lifecycle, case queue, case owner, or
  case SLA.
- Operational exceptions are resolved through the owning domain workflow or
  Super Admin / Outlet Manager paths.
- Customer Support Agents may perform only explicitly permissioned fallback
  actions.
- Exception audit evidence lives on the owning workflow or fallback-action
  record.
- Operations may need manual coordination outside the platform until support
  case management is reconsidered.

## Revisit Trigger

Reconsider support case management when launch operations show that direct
owning-domain workflows cannot provide enough traceability for customer
complaints, cross-domain exceptions, SLA handling, or support workload
management.

## Notes

- In-scope launch support boundaries are defined in
  [../business/support.md](../business/support.md).
- Do not add the deferred permission rows back to MVP requirements unless this
  item is explicitly moved back into scope.
