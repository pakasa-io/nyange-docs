# Authorization Change Co-Approval

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Runtime co-approval for authorization-related changes is not part of the
mid-stage MVP. MVP Identity grant and scope administration is a Super Admin
action with audit requirements. Aggregate-owned authorization rules remain with
the aggregate that enforces the command or read.

## Proposed Scope

If moved back into scope, authorization change co-approval would need explicit
rules for:

- which Identity grant, scope, grant-combination, or aggregate-owned
  authorization-rule changes require co-approval;
- who can request, approve, reject, or cancel a change;
- whether Product Manager approval is required for runtime policy changes;
- separation of requester and approver;
- change draft, pending, approved, rejected, and failed states;
- audit semantics for before/after grant, scope, and aggregate policy facts;
- emergency or break-glass behavior;
- how co-approval affects active sessions and pending grants.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation avoids one
module owning all authorization and keeps Super Admin Identity grant and scope
administration auditable.

- The reader task included identifying actor combinations that require
  co-approval.
- `BI-16` applied the no-self-approval rule to actions requiring co-approval.
- Sensitive authorization-related change was listed as a covered
  no-self-approval example.
- Super Admin responsibilities included break-glass grant and scope changes.
- The access matrix previously granted a cross-cutting authorization-change
  capability as `Full with dual approval for sensitive changes`.
- `E-10` required Product Manager approval and co-approval for authorization
  changes affecting financial behavior, reporting, audit semantics,
  user-visible functionality, or launch scope.
- `E-10` stated that the Super Admin proposing the change could not satisfy the
  co-approval alone.
- `E-10` distinguished Product Manager business-governance approval from
  runtime platform permission.

## Why It Is Out of Scope

Runtime co-approval adds change workflow states, additional actor routing, and
approval semantics before authorization-related changes are frequent enough to
justify that product surface.

## Impact of Deferral

- Super Admin manages Identity grant and scope administration in the MVP.
- Aggregate-owned authorization rules remain with the enforcing aggregate.
- Sensitive mutations still require reason codes and audit logging.
- Product Manager launch governance remains outside runtime platform
  permissions.
- Any additional review can happen off-system until runtime co-approval is
  justified.

## Revisit Trigger

Revisit when runtime grant, scope, grant-combination, or aggregate-owned
authorization-rule changes become frequent or sensitive enough that Super Admin
audit alone is insufficient.

## Notes

- MVP Identity facts are defined in [../business/identity-auth.md](../business/identity-auth.md).
  Aggregate-owned authorization behavior is defined in each aggregate document.
