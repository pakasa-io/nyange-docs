# Support

**Intent**: Define launch support fallback behavior and operational risk alerts.

**Reader task**: Use this document to determine which support-related actions
are available in launch scope and which domain owns the resulting business
record.

**Sources**: §7.16 Operational Risk Alerts

**Related**:
[identity-auth.md](identity-auth.md) for the full access matrix and
[notifications.md](notifications.md) for approved notification channels.

## Boundary

- Launch support behavior is limited to explicitly permissioned fallback
  actions, approved customer notification requests, operational escalation, and
  operational risk alerts.
- Support does not own order, payment, delivery, inventory, refund, finance,
  notification, or ledger records.
- Owning domain workflows execute their own state transitions and audit records.
- Customer Support Agents do not approve refunds, pay refunds, post ledger
  entries, mutate orders, adjust inventory, complete delivery workflows, or
  close financial records unless a separate owning-domain permission explicitly
  grants that action.

## Launch Support Actions

Customer Support Agents may perform only explicitly permissioned fallback
actions. Launch fallback actions may include:

- refund collection-code regeneration, unlock, or audited customer-verified
  reveal allowed by [refund.md](refund.md);
- approved transactional customer notification requests allowed by
  [notifications.md](notifications.md);
- operational escalation to the owning Outlet Manager or Super Admin.

Each fallback action records its audit evidence in the owning workflow or
fallback-action record.

## Operational Risk Alerts

Repeated custody/cash exceptions by delivery agents and staff are evaluated
against rolling-window thresholds.

Evaluated exception types include:

- repeated short collections;
- missing returned cylinders;
- cash discrepancies;
- custody losses;
- approved cash or stock adjustments;
- unresolved custody exceptions.

Thresholds start from global defaults and may define outlet and role overrides.

### Risk Alert Boundary

Risk alerts are informational only. They do not:

- suspend or block the flagged user by themselves;
- block delivery assignment;
- alter assignment ranking;
- deduct from pay;
- assign personal financial liability.

### Visibility

- Alerts are visible only to permissioned Outlet Managers and Super Admins
  within authorized scope.
- Alert notifications, when configured, use push and are limited to those same
  permissioned actors.
- Alerts are not customer-facing.
- Alerts are not sent to the flagged staff member or agent.
- The flagged staff member or agent does not see the alert.

### Risk Alert Lifecycle

States: `OPEN`, `ACKNOWLEDGED`, `DISMISSED`, `ESCALATED`.

- Acknowledging or dismissing an alert requires note/reason and audit.
- `ESCALATED` means an Outlet Manager or Super Admin has linked the alert to an
  owning-domain review, override, personnel review, or other launch-approved
  operational resolution path.
- If work is manually assigned despite an active relevant risk alert, the
  assigning Outlet Manager or Super Admin must record a reason retained in
  audit.

### Sensitive Reads

- Detailed risk-alert views are sensitive reads.
- Detailed views are audit-logged with actor, outlet scope, alert ID, and
  timestamp.
- Aggregate dashboard counts and list views do not require per-alert read audit
  at launch.

### Retention

- Active alerts remain in review until dismissed, escalated, or reviewed with
  required note/reason and audit.
- Eligible dismissed derived alerts are summarized after the active staff-risk
  retention window.
- The active staff-risk retention window is 24 months at launch.
- Escalated alerts follow the retention policy of the linked owning-domain
  review or operational record, then use the risk-alert retention window.
- Risk-alert retention must not delete underlying custody events, delivery
  tasks, cash variances, ledger adjustments, or audit logs.

## Permissions

Trimmed access matrix rows relevant to support. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-06 | P-07 | P-10 |
| --- | --- | --- | --- |
| Customer notification requests | Scoped approved transactional only | Scoped approved transactional only | Full |
| Operational risk alerts | Scoped with explicit permission | - | Full |

## Non-Goals

- Freeform outbound support messaging.
