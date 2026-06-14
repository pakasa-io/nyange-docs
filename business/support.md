# Support

**Intent**: Define launch support fallback behavior.

**Reader task**: Use this document to determine which support-related actions
are available in launch scope and which domain owns the resulting business
record.

**Related**:
[identity-auth.md](identity-auth.md) for the full access matrix and
[notifications.md](notifications.md) for approved notification channels.

## Boundary

- Launch support behavior is limited to explicitly permissioned fallback
  actions, approved customer notification requests, and operational escalation.
- Support does not own order, payment, delivery, inventory, refund, finance,
  notification, or ledger records.
- Owning domain workflows execute their own state transitions and audit records.
- Customer Support Agents do not create refund liabilities, pay refunds, post
  ledger entries, mutate orders, adjust inventory, complete delivery workflows,
  or close financial records unless a separate owning-domain permission
  explicitly grants that action.

## Launch Support Actions

Customer Support Agents may perform only explicitly permissioned fallback
actions. Launch fallback actions may include:

- refund collection-code regeneration or audited customer-verified reveal
  allowed by [refund.md](refund.md);
- approved transactional customer notification requests allowed by
  [notifications.md](notifications.md);
- operational escalation to the owning Outlet Manager or Super Admin.

Each fallback action records its audit evidence in the owning workflow or
fallback-action record.

## Permissions

Trimmed access matrix rows relevant to support. Full matrix:
[identity-auth.md](identity-auth.md).

| Capability | P-06 | P-07 | P-10 |
| --- | --- | --- | --- |
| Customer notification requests | Scoped approved transactional only | Scoped approved transactional only | Full |

## Non-Goals

- Freeform outbound support messaging.
- Operational risk alerts; deferred in
  [../out-of-scope/2026-06-14-operational-risk-alerts.md](../out-of-scope/2026-06-14-operational-risk-alerts.md).
