# Delivery Assignment Ranking

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: Business documentation MVP deferral review
- Owner: Product Manager

## Summary

Automated delivery-agent ranking is not part of the mid-stage MVP. Dispatchers
or Outlet Managers manually choose an eligible agent for each single-order
delivery task.

## Proposed Scope

If moved back into scope, delivery assignment ranking would need explicit rules
for:

- assignment load calculation;
- least-recent-assignment calculation;
- deterministic tie-breaking;
- workload fairness across agents;
- visibility of ranking rationale to dispatchers;
- manual override behavior;
- interaction with shifts, live location, route distance, and agent capacity;
- audit and reporting for auto-ranked or auto-selected assignments.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation describes
manual dispatcher assignment only.

```text
rank agents by:
  1. queued_assignment_load  ASC
  2. least_recent_assignment ASC
  3. deterministic_tie_breaker
```

- Formal shift schedules, live location, and route-distance optimization did
  not add eligibility requirements, expand scope, or change assignment ranking.
- Dispatchers and Outlet Managers could still assign or change the assigned
  delivery agent before pickup with a recorded reason.

## Why It Is Out of Scope

Manual assignment is sufficient while each outlet has a small delivery team.
Automated ranking adds policy, data, and fairness semantics before dispatch
volume requires them.

## Impact of Deferral

- Dispatchers or Outlet Managers choose eligible agents manually.
- The system may show eligible agents but does not rank or recommend them.
- No workload scoring or assignment fairness reporting is required for MVP.

## Revisit Trigger

Revisit when manual assignment causes measured dispatch delays, uneven agent
workload, or avoidable missed deliveries.

## Notes

- MVP delivery assignment behavior is defined in [../business/delivery.md](../business/delivery.md).
