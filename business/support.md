# Support

**Intent**: Define the support case lifecycle, operational risk alert system, and the boundary between support actions and domain workflow execution on the Nyange platform.

**Sources**: §6.7 Support Case Lifecycle, §7.16 Operational Risk Alerts

**Related**: [identity-auth.md](identity-auth.md) — full access matrix; [order.md](order.md), [payment.md](payment.md), [delivery.md](delivery.md), [refund.md](refund.md) — domain records linked from support cases

---

## Support Case Lifecycle

### State

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

Terminal states: `CLOSED`, `CANCELLED`

### Business Rules

**Visibility:**
- Support cases are internal records. Customers receive notifications and outcomes through direct channels (push, email) but have no access to case records, internal notes, owner assignments, SLA timers, linked entity history, or risk context.

**Escalation:**
- Support escalation is represented through case type, priority, queue/owner coverage, linked records, and notifications.
- `ESCALATED` is not a support case status at launch.

**Case structure:**
- Support cases are structured but lightweight at launch.
- Used for: customer-facing issues, complaints, refund exceptions, reassignment disputes, payment exceptions, delivery exceptions, operational escalations, risk-alert follow-up, and cross-domain follow-up.
- Support cases are not required for every exception.
- Each case tracks: type, status, priority, queue, optional individual owner, linked entities, notes, SLA timestamps, assignment history, and status history.

**Domain record linkage:**
- Support cases link to domain records (orders, payments, deliveries, refunds) but do not own those records.
- A support case may link to multiple domain records when the issue spans related orders, payments, deliveries, refunds, or other scoped entities.

**Notes:**
- Support-case notes are append-only; corrections are new notes, not edits to prior notes.
- Support cases may store concise customer communication summaries but do not act as a two-way inbox and do not store raw recordings, full transcripts, chat archives, or standalone evidence attachments at launch.
- Customer communication summaries link to relevant notification logs or existing domain records where applicable.

**Action boundary:**
- Customer Support Agents can request approved workflows in linked business domains but cannot execute them, except for explicitly permissioned fallback actions listed in the access matrix.
- The authorized owner of the affected domain workflow executes and records requested workflow changes.
- For refunds, support may create a refund-request action request with linked order, payment, refund context, and notes; that request does not create, approve, pay, void, write off, or ledger-post the refund by itself.

**Support action request states:**
- `REQUESTED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, `CANCELLED`
- `COMPLETED`, `REJECTED`, and `CANCELLED` are terminal for review-readiness purposes.
- A case is not ready for support review while a customer response is still required, but non-blocking notes or linked records do not prevent review readiness.
- A support action request does not alter linked domain records by itself.
- Linked domain workflow completion adds a support-case note and may move the case to `READY_FOR_SUPPORT_REVIEW` only when no blocking support action requests remain.
- The case cannot close until a permissioned support actor records the required closure decision.

**No goodwill credits:**
- No goodwill credits are supported at launch. Support outcomes use approved refunds or adjustments tied to orders.

**Communications:**
- Support-triggered customer communications must use approved transactional notifications and templates with safe structured parameters.
- Support-case permissions do not grant notification-template administration, and support does not send arbitrary freeform outbound messages from the case workflow.

**Case creation:**
- Support cases are created manually at launch.
- Operational alerts, risk alerts, closure blockers, payment exceptions, delivery exceptions, and refund exceptions may supply default case context (case type, linked entities, priority suggestion, summary, and supporting context), but no case exists until a permissioned Customer Support Agent, in-scope Outlet Manager, or Super Admin explicitly creates it.

**Queue and routing:**
- Cases enter a centrally configured default queue based on type, priority, and outlet scope.
- Queue membership is derived from the actor's support permissions and outlet scope.
- Individual owner assignment is optional; assigning an individual owner does not remove queue coverage.
- When no individual owner is assigned or the owner is unavailable, the queue remains accountable for coverage.
- Permissioned Customer Support Agents, Outlet Managers, or Super Admins may manually move a case only to another allowed queue and must record a reason.
- Queue moves and individual owner changes are append-only assignment history events with actor, previous/new ownership, reason, and timestamp.
- Assignment changes affect support coverage and accountability only; they do not mutate linked orders, payments, refunds, delivery, inventory, or financial ledger records.
- Individual owner assignment sends a best-effort internal notification to that owner and records the notification timestamp; queue-owned cases rely on queue dashboards and SLA/priority views rather than notifying every queue member.

**Outlet scope:**
- Cases are outlet-scoped. A case's outlet scope is determined by its primary outlet and linked domain records.
- A Customer Support Agent cannot view cases for an outlet they are not assigned to unless granted explicit cross-outlet access.
- Cases linked to multiple outlets are not partially visible at launch. A Customer Support Agent, Outlet Manager, or Super Admin must have permission for all linked outlets, or explicit cross-outlet/global support access, to view or manage the case.

**Launch routing:**
- Launch support routing uses default queues and permissioned manual moves only.
- Workload balancing, shift scheduling, round-robin assignment, skill-based routing, and complex support rosters do not grant case ownership, visibility, or routing authority at launch.

**Queue configuration:**
- Queue definitions are global configuration only.
- Only Super Admins or actors with explicit global support queue-configuration permission may create, edit, deactivate, reprioritize support queues, or change routing structures.
- Outlet Managers may work cases within scope and, when explicitly granted assignment permission, move cases between allowed existing queues.
- Outlet Managers request new queues, routing-structure changes, or per-outlet queue behavior only through Customer Support Agent or Super Admin escalation; they do not define or manage queue configuration directly, and no per-outlet queue-configuration approval path exists at launch.
- Queue configuration changes must be audited with actor, before/after values, reason, and timestamp.

**Sensitive reads:**
- Detailed support-case views are sensitive reads and must be audit-logged with actor, case ID, outlet scope, and timestamp.
- Aggregate queue counts and list views do not require per-case read audit at launch.

**SLA timers:**
- Support-case SLA timers are advisory only.
- They can drive notifications and dashboards, and urgent unowned or overdue cases may notify permissioned Customer Support Agents, Outlet Managers, or Super Admins according to priority and scope.
- They do not block or advance orders, payments, refunds, deliveries, inventory, daily closing, financial closure, or ledger posting.

**Retention:**
- Case records have their own retention window, independent of the subject domain records.
- At launch, support-derived records are retained for 36 months after the case reaches `CLOSED` or `CANCELLED`.
- Open support cases are not purged by age; the retention countdown starts only after `CLOSED` or `CANCELLED`.
- Support retention applies only to support-derived records and must not delete or weaken linked orders, payments, refunds, deliveries, financial ledger records, ledger entries, notification logs, closure blockers, operational risk alerts, or audit logs.

---

## Operational Risk Alerts (§7.16)

Repeated custody/cash exceptions by delivery agents and staff are evaluated against rolling-window thresholds. Evaluated exception types include:
- Repeated short collections
- Missing returned cylinders
- Cash discrepancies
- Custody losses
- Forced-closure adjustments
- Unresolved custody exceptions

When a threshold is exceeded, an operational risk alert is raised. Thresholds start from global defaults and may define outlet and role overrides.

### What Risk Alerts Do Not Do

Risk alerts are **informational only**. They do not:
- Suspend or block the flagged user by themselves.
- Block delivery assignment or alter assignment ranking.
- Deduct from pay or assign personal financial liability.
- Create a support case without a manual Outlet Manager or Super Admin decision.

### Visibility

- Alerts are visible only to permissioned Outlet Managers and Super Admins within their authorized scope.
- Alert notifications, when configured, use push and are limited to those same permissioned actors.
- They are not customer-facing and are not sent to the flagged staff member or agent.
- The flagged staff member or agent does not see the alert.

### Risk Alert Lifecycle

States: `OPEN`, `ACKNOWLEDGED`, `DISMISSED`, `CASE_OPENED`

- Acknowledging or dismissing an alert requires a note/reason and audit.
- When an Outlet Manager for that outlet or a Super Admin chooses to open a support case from a risk alert, the risk alert may supply default case context (subject user, alert type, trigger details); no case exists until that actor confirms creation.
- Once the manual case is created, the case is permanently linked to the alert and the alert transitions to `CASE_OPENED`.
- If work is manually assigned despite an active relevant risk alert, the assigning Outlet Manager or Super Admin must record a reason, which is retained in audit.

**Sensitive reads:**
- Detailed risk-alert views are sensitive reads and are audit-logged with actor, outlet scope, alert ID, and timestamp.
- Aggregate dashboard counts and list views do not require per-alert read audit at launch.

### Retention

- Active alerts remain in review until dismissed, escalated, or reviewed with the required note/reason and audit.
- Eligible dismissed derived alerts are summarized after the active staff-risk retention window, which is 24 months at launch.
- `CASE_OPENED` alerts follow the linked support-case closure and retention path first, then use the same risk-alert retention window.
- Risk-alert retention must not delete the underlying custody events, delivery runs, cash variances, forced closures, ledger adjustments, or audit logs.

---

## Permissions

Trimmed access matrix rows relevant to support. Full matrix: [identity-auth.md](identity-auth.md).

| Capability | P-06 | P-07 | P-10 |
|---|---|---|---|
| Support case creation | Scoped | Scoped | Full |
| Support case management | Scoped | Scoped | Full |
| Support queue configuration | – | – | Full |
| Support communication logging | Scoped | Scoped | Full |
| Customer notification requests | Scoped (approved transactional only) | Scoped (approved transactional only) | Full |
| Support action requests (create/request) | Scoped | Scoped | Full |
| Support action requests (execute) | Scoped | – | Full |
| Operational risk alerts | Scoped (with explicit permission) | – | Full |

---

## Authorization Edge Cases

**E-03**: Customer Support Agents cannot view another outlet's cases unless a Super Admin has granted explicit cross-outlet support access. There is no implicit cross-outlet access based on case type or priority.
