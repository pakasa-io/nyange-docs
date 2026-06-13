# Modular Monolith Editing Constraints

Version: 2026-05-31
Status: Normative

## Purpose

Apply boundary and ownership fixes without changing intended product behavior,
implementation scope, or source-of-truth business rules.

## Source Authority

- Business source of truth: product or domain specification.
- Canonical ownership map: module catalog or boundary map.
- Edit order: update the catalog or boundary map first, then every affected
  module specification. Never leave modules in inconsistent boundary states.
- If sources conflict, flag the ambiguity. Do not invent a resolution.

## Ownership Invariants

- Every business lifecycle has exactly one owning module — the sole source of
  truth for that lifecycle's state transitions.
- Every durable record, policy decision, approval rule, and write-side invariant
  has exactly one owning module.
- Other modules may observe, project, cache, coordinate, or emit/consume events
  — never co-own mutable state.
- A coordinator does not gain ownership of participant records. Cross-module
  workflows invoke module-owned commands; they do not directly mutate another
  module's records.
- A non-owning module's local status must be labeled as a projection, snapshot,
  gate result, or participant-local state — not the canonical lifecycle.

## Cross-Module Contracts

Every cross-module interaction requires a complete contract:

| Field | Required |
|---|---|
| Contract Type | yes |
| Provider / Consumer | yes |
| Trigger | yes |
| Payload / Command / Event | yes |
| Consistency Model | yes |
| Idempotency / Retry | yes |
| Failure Handling | yes |
| Source Citation | yes |

Avoid vague verbs ("uses," "syncs," "notifies," "consumes facts") without a
backed contract.

For atomic multi-module workflows: name the coordinator, participants, commit
point, participant commands, rollback behavior, and retry behavior.

When consuming another module's data: specify live query, immutable snapshot,
cached projection, or event payload. If a snapshot drives downstream decisions,
state who created it, when it was frozen, and whether source changes affect it.

Domain events represent committed facts, not requests. If the interaction asks
another module to decide or mutate, model it as a command. Events must carry:
source module, source record, event version, occurrence time, idempotency key.

## State and Failure Integrity

- Every lifecycle state requires: owner, allowed transitions, trigger, actor or
  initiator, and terminality. No orphan states.
- Every failure — rejected command, stale projection, missed event, timeout,
  retry exhaustion — must have an owning module and a resulting state.
- Approval records, thresholds, requester/approver separation, and
  approval-triggered mutations must have explicit owners. Approval alone does
  not imply the domain mutation happened.

## Data and Boundary Discipline

- Duplicate data only as immutable snapshots, derived projections, audit facts,
  or participant-local status. Label source module and canonical vs. derived
  status on every duplicated field.
- Read models (reporting, analytics, projections) must not create mutation
  ownership. No shared mutable tables across modules.
- Authorization modules own identity, permission, scope, and session assurance.
  Domain modules own business eligibility, thresholds, approval requirements,
  and policy outcomes. Do not move business policy into access control.
- Infrastructure (persistence, messaging, scheduling, idempotency) supports
  workflows but does not own records, rules, or state transitions.
- Use precise timing language. Specify when clocks start, what stops them, and
  which module owns timeout evaluation. Avoid "soon," "later," or "eventually"
  unless intentionally non-normative.
- Retention, archive, purge, redaction, and legal-hold rules must have owning
  modules. Retention must not weaken audit or ledger records.
- Do not weaken append-only, immutable, terminal, audit, authorization,
  consistency, or idempotency rules. If an invariant exists in source material,
  all affected modules must preserve it.

## Documentation Discipline

- Use identical names for modules, records, states, actors, policies, and
  contracts across all documents. Flag terminology conflicts; do not normalize
  silently.
- Each module specification must be self-contained. Do not rely on "see
  catalog" or "handled elsewhere" as the only ownership explanation.
- Make the smallest edits needed to fix the boundary issue. Do not restyle or
  rephrase stable sections unless required for consistency.
- Do not rename, split, merge, or delete modules unless explicitly instructed.
  If restructuring seems necessary, document it as a recommendation first.
- Do not invent thresholds, states, permissions, timeouts, approval rules, or
  business policies absent from source material. Flag them as open decisions.
- Provider and consumer descriptions of the same interaction must use the same
  name and semantics. Asymmetric documentation is a boundary defect.

## Validation

After each boundary edit:

1. Ask: could an engineer implement this module without guessing ownership,
   allowed writes, contracts, states, or failure handling? If not, add the
   missing detail.
2. Scan all affected modules for stale owns/does-not-own, workflow, state,
   authorization, and policy text.
3. Run documentation, schema, or architecture validation checks. After automated
   checks pass, manually review cross-module workflows for ownership
   contradictions, missing contracts, and accidental scope expansion.
