# Authorization Policy Co-Approval

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Runtime co-approval for authorization-policy changes is not part of the
mid-stage MVP. MVP authorization policy management is a Super Admin action with
audit requirements.

## Proposed Scope

If moved back into scope, authorization policy co-approval would need explicit
rules for:

- which authorization-policy changes require co-approval;
- who can request, approve, reject, or cancel a change;
- whether Product Manager approval is required for runtime policy changes;
- separation of requester and approver;
- policy-change draft, pending, approved, rejected, and failed states;
- audit semantics for before/after permission policy;
- emergency or break-glass behavior;
- how co-approval affects active sessions and pending grants.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes
Super Admin authorization policy management with audit.

- The reader task included identifying actor combinations that require
  co-approval.
- `BI-16` applied the no-self-approval rule to actions requiring co-approval.
- Sensitive authorization-policy change was listed as a covered no-self-approval
  example.
- Super Admin responsibilities included break-glass authorization policy
  changes.
- The access matrix granted `Authorization policy management` as `Full with dual
  approval for sensitive changes`.
- `E-10` required Product Manager approval and co-approval for authorization
  policy changes affecting financial behavior, reporting, audit semantics,
  user-visible functionality, or launch scope.
- `E-10` stated that the Super Admin proposing the change could not satisfy the
  co-approval alone.
- `E-10` distinguished Product Manager business-governance approval from
  runtime platform permission.

## Why It Is Out of Scope

Runtime co-approval adds policy workflow states, additional actor routing, and
approval semantics before authorization-policy changes are frequent enough to
justify that product surface.

## Impact of Deferral

- Super Admin manages authorization policy changes in the MVP.
- Sensitive mutations still require reason codes and audit logging.
- Product Manager launch governance remains outside runtime platform
  permissions.
- Any additional review can happen off-system until runtime co-approval is
  justified.

## Revisit Trigger

Revisit when runtime permission-policy changes become frequent or sensitive
enough that Super Admin audit alone is insufficient.

## Notes

- MVP authorization behavior is defined in [../business/identity-auth.md](../business/identity-auth.md).
