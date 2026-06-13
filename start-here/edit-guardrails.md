# Modular Monolith Editing Constraints

Version: 2026-05-31
Status: Normative editing guardrails
Scope: Modular monolith architecture and module specification documentation
Audience: Architects, engineers, AI agents

## Purpose

Use these constraints when editing modular monolith architecture documentation. The goal is to apply boundary and ownership fixes without changing intended product behavior, implementation scope, or source-of-truth business rules.

## Constraints

### 1. Source Hierarchy

- Treat the product or domain specification as the business source of truth.
- Treat the architecture or specification framework as the required documentation structure.
- Treat the module catalog or boundary map as the canonical module ownership map.
- If sources conflict, do not invent a resolution. Flag the ambiguity.

### 2. Edit Order

- If a boundary or ownership decision changes, update the canonical module catalog or boundary map first.
- Then update every affected module specification.
- Do not update one module into a new boundary model while related modules still describe the old one.

### 3. Single Lifecycle Owner

- Every business lifecycle has exactly one canonical owning module.
- The owning module is the only source of truth for that lifecycle's state transitions.
- Other modules may observe the lifecycle, gate their own workflows on its facts, cache derived read models, or project it for reporting.
- Other modules must not store competing canonical states for the same lifecycle.
- If another module needs a local status, it must be explicitly labeled as a projection, snapshot, gate result, or participant-local lifecycle state, not the canonical lifecycle.

### 4. Single Mutation Owner

- Every durable record, policy decision, approval rule, and write-side invariant must have exactly one owning module.
- Other modules may query, request, coordinate, emit events, consume events, or build projections.
- Never describe two modules as co-owning the same mutable business state.

### 5. Coordinator Rule

- A coordinator does not gain ownership of participant records.
- For multi-module workflows, the coordinator must invoke module-owned participant commands or consume module-owned facts.
- Do not allow one module to directly mutate another module's records.

### 6. State-Machine Completeness

- Every lifecycle state must have a clear owner, allowed transitions, trigger, actor or system initiator, and terminality.
- Do not add a state unless its entry path, exit path, and owning module are explicit.
- Do not leave orphan states that appear in one section but not in workflows or rules.

### 7. Contract Completeness

Every cross-module interaction must be expressed as a complete contract with:

- Contract Type
- Provider
- Consumer
- Trigger
- Payload / Command / Event
- Consistency Model
- Idempotency / Retry
- Failure Handling
- Source Citation or Source Basis

Avoid vague phrases such as "uses," "handles," "syncs," "notifies," or "consumes facts" unless backed by a concrete contract.

### 8. Atomic Workflow Clarity

- For any multi-module atomic workflow, explicitly identify the coordinator, participants, commit point, participant commands, rollback behavior, and retry behavior.
- Do not describe "all-or-nothing" behavior without naming which module owns each participant mutation.

### 9. Snapshot vs Live-Read Semantics

- When one module consumes another module's data, specify whether it receives a live query result, immutable snapshot, cached projection, or event payload.
- If a snapshot is used for downstream decisions, state who creates it, when it is frozen, and whether later source changes affect it.

### 10. Event Discipline

- Domain events must represent committed facts, not requests for another module to maybe do work.
- If the interaction asks another module to decide or mutate, model it as a command, not an event.
- Events should identify source module, source record, event version, occurrence time, and idempotency key.

### 11. Failure Ownership

- Every failed command, rejected participant action, stale projection, missed event, timeout, and retry exhaustion must have an owning module.
- Avoid phrases like "handled by the system" unless the owning module and resulting state are named.

### 12. Authorization vs Business Policy

- Authorization modules decide identity, permission, scope, session assurance, and separation-of-duty checks.
- Domain modules decide business eligibility, thresholds, approval requirements, state transitions, and policy outcomes.
- Do not move business policy into access control just because permission is required.

### 13. Approval Boundary

- Approval records, approval thresholds, requester/approver separation, and approval-triggered mutations must have explicit owners.
- Approval by itself should not imply the target domain mutation happened unless the owning module records the mutation.

### 14. Duplication Rule

- Duplicate data only as immutable snapshots, derived projections, audit facts, or participant-local status.
- Any duplicated field must state its source module and whether it is canonical, copied, derived, or historical.
- Do not introduce synchronized mutable copies across modules.

### 15. Read/Write Separation

- Reporting, analytics, search, dashboards, projections, and cross-module views must not create mutation ownership.
- Read models may consume events or source-owned read APIs, but must not imply shared mutable tables.

### 16. No Shared-Table Implication

- Do not describe interactions in a way that implies modules share mutable tables or bypass owning APIs.
- Persistence may be physically colocated in a modular monolith, but logical ownership must remain module-scoped.

### 17. Preserve Scope

- Do not add future, speculative, or out-of-scope capabilities while clarifying boundaries.
- If a capability is not already approved by source material, mark it as an ambiguity or future candidate.

### 18. Preserve Invariants

- Do not weaken append-only, immutable, terminal, approval, audit, authorization, consistency, retention, or idempotency rules.
- If an invariant exists in source material, every affected module must preserve it consistently.

### 19. Temporal Precision

- Use precise timing language for windows, expiry, retention, scheduling, and deadlines.
- Specify when clocks start, what pauses or stops them, and what module owns timeout evaluation.
- Avoid vague terms like "soon," "later," "eventually," or "periodically" unless intentionally non-normative.

### 20. Retention and Audit Ownership

- If records can be archived, purged, summarized, redacted, retained, or placed on legal hold, identify the owning module.
- Retention rules must not weaken audit, ledger, or other permanent accountability records.

### 21. Infrastructure Boundary

- Do not let persistence, framework, messaging, scheduling, idempotency, or platform mechanics become business ownership.
- Infrastructure may support workflows, but business modules still own their records, rules, and state transitions.

### 22. Support and Action-Request Boundary

- A support, workflow, or case-management module may own the request or case envelope.
- The affected domain module owns typed action schemas, validation, authorization, execution, and resulting domain mutations.
- Do not create generic action execution paths that bypass domain ownership.

### 23. Terminology Consistency

- Use the same names for modules, records, states, actors, policies, and contracts across all affected documents.
- If terminology differs between sources, flag the inconsistency instead of silently normalizing it.

### 24. Keep Specs Standalone

- Each module specification must be understandable on its own.
- Do not rely on "see catalog," "see other module," or "handled elsewhere" as the only explanation of ownership, lifecycle, or contract behavior.

### 25. Avoid Broad Rewrites

- Make the smallest documentation edits needed to fix the boundary issue.
- Do not restyle, reorder, rename, or rephrase stable sections unless required for consistency.

### 26. Keep Module Identities Stable

- Do not rename, split, merge, or delete modules unless explicitly instructed.
- If a split, merge, rename, or reframing seems necessary, document it as a recommendation first.

### 27. No Hidden Policy Invention

- If a fix requires a new threshold, state, permission, timeout, exception path, actor role, approval rule, retention rule, or business policy not present in source material, do not invent it.
- Flag it as an open decision.

### 28. Open-Decision Handling

- If a boundary fix reveals an unanswered product or architecture decision, add it as an explicit open ambiguity.
- Do not bury unresolved decisions inside prose that sounds final.

### 29. Cross-Reference Consistency

- When a module says it provides a command, query, event, projection, or participant command, every consuming module must describe the same interaction using the same name and semantics.
- Provider and consumer descriptions must not contradict each other.

### 30. Backward Compatibility of Docs

- Preserve existing stable module slugs, state names, contract names, and terminology unless the change is intentional and propagated everywhere.
- When renaming a concept, update all affected references in the same edit batch.

### 31. Implementation-Readiness Test

- After editing a module, ask: could an engineer implement this module without guessing ownership, allowed writes, contracts, states, or failure handling?
- If not, add the missing boundary detail instead of adding explanatory filler.

### 32. Review After Each Boundary Edit

- After each ownership or contract change, scan all affected modules for stale "owns," "does not own," workflow, state, authorization, and policy text.
- Do not leave asymmetric documentation where one module believes a boundary changed and the other does not.

### 33. Validation Gate

- After edits, run the project's documentation, schema, or architecture validation checks.
- If automated validation passes, still manually review edited cross-module workflows for ownership contradictions, missing contracts, and accidental scope expansion.
