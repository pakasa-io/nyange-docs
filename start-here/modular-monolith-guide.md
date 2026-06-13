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

## Documentation

### Module Specification Shape

A complete module specification covers:

1. Module identity and owner.
2. Boundary — what the module owns and does not own.
3. Records and data model.
4. Business rules and policies.
5. Lifecycle state machine.
6. Commands — inputs the module accepts.
7. Queries — read access it exposes.
8. Events — committed facts it emits.
9. REST endpoints — HTTP routes, methods, request/response shapes, auth
   requirements, and error responses exposed to external clients.
10. Projections it maintains from other modules.
11. Cross-module contracts.
12. Failure handling.
13. Open questions.

Each specification must be self-contained. Do not rely on "see catalog" or
"handled elsewhere" as the only explanation of ownership, lifecycle, or
contract behavior.

Write module specifications following the canonical documentation style:
`start-here/ai-agent-documentation-style-guide.md`.

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
