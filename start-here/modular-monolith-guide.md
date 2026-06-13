# Modular Monolith Design and Documentation Guide

## Purpose

Use this guide when designing, specifying, or documenting a modular monolith
system. It covers boundary design, cross-module interaction patterns, state and
lifecycle modeling, data ownership, and how to produce implementation-ready
module documentation.

## What Is a Modular Monolith

A modular monolith is a single deployable unit organized around strong internal
module boundaries. Modules are logical ownership units, not deployment units.
Persistence is physically colocated, but logical ownership is module-scoped.

The goal is the operational simplicity of a monolith with the boundary
discipline of service-oriented design. The primary risk is ownership erosion:
when two modules share mutable state, the boundary is already broken.

## Core Principles

**Single ownership.** Every business lifecycle, durable record, policy
decision, approval rule, and write-side invariant has exactly one owning
module. Two modules cannot co-own mutable state.

**Boundary integrity.** Cross-module access is permitted only through defined
contracts: commands, queries, events, or projections. Direct table sharing or
direct mutation of another module's records breaks the boundary.

**Source authority.** The business source of truth is the product or domain
specification. The canonical ownership map is the module catalog or boundary
map. When sources conflict, flag the ambiguity — do not invent a resolution.

## Module Design

### What Defines a Module

A module is a cohesive unit of business responsibility, sized around a bounded
context or a distinct business capability. A well-scoped module:

- Has a clear owner.
- Owns a coherent set of records and state machines.
- Produces events that other modules may consume.
- Exposes commands and queries for the things it controls.
- Does not depend on the internals of other modules.

### Finding Boundaries

Identify module boundaries by asking:

- **What business capability does this serve?** Group behavior whose
  terminology, rules, and lifecycle are coherent and self-consistent — a
  bounded context is a natural candidate.
- **Who owns it?** A module should map to a clear ownership unit. If two teams
  both need to change the same module frequently, the boundary is wrong.
- **What is its change rate?** Stable policy and core domain rules change
  slowly; operational workflows and integrations change faster. When stable and
  volatile code must change together, consider splitting.

Signals the boundary is correct:
- A team can own and evolve it without coordinating writes to other modules.
- Business terminology and rules are internally consistent.
- Lifecycle state transitions belong entirely within the module.

Signals the boundary is wrong:
- Two modules claim the same record or state.
- A command must mutate records in more than one module directly.
- A lifecycle's states are spread across modules.
- Every change requires a parallel change to another module.

When uncertain, prefer fewer, larger modules and split only when a clear seam
emerges: a different owner, a different change rate, or a lifecycle that is
fully self-contained.

### What a Module Owns

| Owned by this module                                | Not owned                                |
|-----------------------------------------------------|------------------------------------------|
| Its canonical records and fields                    | Records owned by another module          |
| Its business rules and policies                     | Business rules of another domain         |
| Its lifecycle state transitions                     | Canonical state of another lifecycle     |
| The events it emits                                 | Events emitted by another module         |
| Validation, authorization, and mutation of its data | Direct writes to another module's tables |

### Module Catalog

Maintain a canonical module catalog or boundary map as the authoritative
ownership index. When a boundary or ownership decision changes, update the
catalog first, then every affected module specification.

### Dependency Direction

Module dependencies must be acyclic. If module A depends on module B, module B
must not depend on module A.

Direction flows from orchestration toward core domain:

| Layer | Responsibility | Depends On |
|---|---|---|
| **External adapters** | Integrate third-party systems | Domain modules |
| **Orchestration** | Coordinate multi-module workflows | Core + supporting domain |
| **Supporting domain** | Cross-cutting capabilities (auth, audit, notifications) | Core domain |
| **Core domain** | Fundamental records, rules, lifecycles | Nothing above |

A module must never import another module's internal implementation. All access
goes through published contracts. A dependency cycle signals a misplaced
boundary or a missing abstraction — resolve it by extracting the shared concept
into its own module or reversing the direction with an event.

## Cross-Module Interaction

### Interaction Patterns

| Pattern                     | Use When                                                                         |
|-----------------------------|----------------------------------------------------------------------------------|
| **Command**                 | You need another module to decide or mutate; expects a response                  |
| **Query**                   | You need read-only data from another module                                      |
| **Domain Event**            | You are publishing a committed fact; consumers decide what to do                 |
| **Projection / Read Model** | You need a local read-only copy of another module's data                         |
| **Coordinator**             | You are orchestrating a multi-module workflow without owning participant records |

### Events vs Commands

Use a **command** when the interaction requires a decision or mutation by the
target module and the caller needs a response.

Use a **domain event** when publishing a committed fact. The producer does not
know or care who consumes it. Domain events represent committed facts, not
requests — if the interaction asks another module to decide or mutate, it is a
command, not an event.

Events must carry: source module, source record, event version, occurrence
time, and idempotency key.

### Coordinators

A coordinator orchestrates multi-module workflows but does not own participant
records. For each atomic multi-module workflow, specify:

- Coordinator and participant modules.
- Participant commands.
- Commit point.
- Rollback or compensation behavior.
- Retry behavior.

### Data Consumption Semantics

When a module consumes another module's data, specify the semantics:

- **Live query** — reflects current source state at read time.
- **Immutable snapshot** — frozen copy; source changes do not affect it.
- **Cached projection** — local read model maintained by consuming events.
- **Event payload** — data embedded in an event at emission time.

If a snapshot or projection drives a downstream decision, state who creates it,
when it is frozen, and what happens when the source changes.

### Contract Specification

Every cross-module interaction requires a complete contract:

| Field                     | Required |
|---------------------------|----------|
| Contract Type             | yes      |
| Provider / Consumer       | yes      |
| Trigger                   | yes      |
| Payload / Command / Event | yes      |
| Consistency Model         | yes      |
| Idempotency / Retry       | yes      |
| Failure Handling          | yes      |
| Source Citation           | yes      |

Avoid vague verbs ("uses," "syncs," "notifies," "consumes facts") without a
backed contract.

## State and Lifecycle Design

- Every lifecycle state requires: owning module, allowed transitions, trigger,
  actor or system initiator, and terminality. No orphan states.
- The owning module is the sole source of truth for state transitions. Other
  modules may observe or project the lifecycle but must not store competing
  canonical states.
- A non-owning module's local status must be labeled as a projection, snapshot,
  gate result, or participant-local state — never the canonical lifecycle.
- Every failure — rejected command, stale projection, missed event, timeout,
  retry exhaustion — must have an owning module and a resulting state.
- Approval records, thresholds, requester/approver separation, and
  approval-triggered mutations must have explicit owners. Approval alone does
  not imply the domain mutation occurred.

## Data Ownership

### Canonical vs Derived

| Kind                        | Characteristics                                                  |
|-----------------------------|------------------------------------------------------------------|
| **Canonical**               | Mutable; owned by one module; source of truth                    |
| **Snapshot**                | Immutable copy frozen at a point in time                         |
| **Projection**              | Read model derived from events; rebuilt from source              |
| **Participant-local state** | Local status in a non-owning module; not the canonical lifecycle |

Every duplicated field must state its source module and whether it is
canonical, copied, derived, or historical. Do not introduce synchronized
mutable copies across modules.

### Read/Write Separation

Read models — reporting, analytics, dashboards, projections, cross-module views
— must not create mutation ownership. Read models may consume events or
source-owned read APIs; they must not imply shared mutable tables.

### Authorization vs Business Policy

- Authorization modules own: identity, permission, scope, session assurance,
  and separation-of-duty checks.
- Domain modules own: business eligibility, thresholds, approval requirements,
  state transitions, and policy outcomes.
- Do not move business policy into access control just because permission is
  also required.

### Infrastructure Boundary

Infrastructure (persistence, messaging, scheduling, idempotency, framework
mechanics) supports workflows but does not own records, rules, or state
transitions. A module that delegates to infrastructure still owns its business
outcomes.

### Temporal and Retention

- Use precise timing language. State when clocks start, what stops them, and
  which module owns timeout evaluation. Avoid "soon," "later," or "eventually"
  unless intentionally non-normative.
- Retention, archive, purge, redaction, and legal-hold rules must have owning
  modules. Retention must not weaken audit or ledger records.
- Do not weaken append-only, immutable, terminal, audit, or idempotency rules.
  If an invariant exists in source material, all affected modules must preserve
  it.

### Observability

Each module owns its observability surface: the logs, metrics, traces, and
audit records it emits. Observability ownership follows the same rule as data
ownership — the module that performs an action owns the observable record of
that action.

Each module's observability surface should be sufficient to answer:

- Did the operation succeed or fail, and why?
- Which actor or system triggered it?
- Which records were affected?
- How long did it take?

For cross-module workflows, each participating module emits its own span and
includes a shared correlation ID so the full workflow can be reconstructed
across module boundaries.

Audit records — immutable logs of who did what to which record and when — are
owned by the module that performed the mutation. Do not delegate audit
responsibility to a shared logging layer.

## Documentation

### Module Specification Shape

A complete module specification covers:

1. Module identity and owner.
2. Boundary — what the module owns and does not own.
3. Dependencies — other modules this module depends on: events subscribed,
   commands invoked, queries made, and projections maintained.
4. Records and data model.
5. Business rules and policies.
6. Lifecycle state machine.
7. Commands — inputs the module accepts.
8. Queries — read access it exposes.
9. Events — committed facts it emits.
10. REST endpoints — HTTP routes, methods, request/response shapes, auth
    requirements, and error responses exposed to external clients.
11. Projections it maintains from other modules.
12. Cross-module contracts.
13. Failure handling.
14. Observability — logs, metrics, traces, and audit records the module emits.
15. Open questions.

Each specification must be self-contained. Do not rely on "see catalog" or
"handled elsewhere" as the only explanation of ownership, lifecycle, or
contract behavior.

Write module specifications following the canonical documentation style:
`start-here/doc-style.md`.

### Editing Discipline

- Update the module catalog or boundary map before affected module specs. Never
  leave modules in inconsistent boundary states.
- Make the smallest edits needed. Do not restyle or rephrase stable sections
  unless required for consistency.
- Do not rename, split, merge, or delete modules unless explicitly instructed.
  If restructuring seems necessary, document it as a recommendation first.
- Do not invent thresholds, states, permissions, timeouts, or policies absent
  from source material. Flag them as open decisions.
- Use identical names for modules, records, states, actors, policies, and
  contracts across all documents. Flag terminology conflicts; do not normalize
  silently.
- Provider and consumer descriptions of the same interaction must use the same
  name and semantics. Asymmetric documentation is a boundary defect.

## Validation

After each boundary change or new module specification:

1. **Implementation readiness** — could an engineer implement this module
   without guessing ownership, allowed writes, contracts, states, or failure
   handling? If not, add the missing detail.
2. **Consistency scan** — check all affected modules for stale
   owns/does-not-own, workflow, state, authorization, and policy text.
3. **Automated checks** — run documentation, schema, or architecture
   validation. After automated checks pass, manually review cross-module
   workflows for ownership contradictions, missing contracts, and accidental
   scope expansion.
