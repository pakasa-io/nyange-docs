# Engineering Detail Fallback Style Guide

## Purpose

Use this fallback when the default AI-agent documentation style needs deeper
technical detail.

This style applies to any software project document that must define
implementation contracts, correctness constraints, reliability behavior, or
evolvability boundaries.

## Use When

Use this fallback for documents that define:

- Architecture boundaries.
- Module responsibilities.
- API or event contracts.
- Data models and persistence rules.
- State machines.
- Invariants.
- Error and failure handling.
- Operational behavior.
- Migration or rollout constraints.
- Reliability and evolvability requirements.
- Integration responsibilities.

## Tone

- Direct.
- Constraint-heavy.
- Correctness-oriented.
- Operationally aware.

Avoid broad product narrative unless it clarifies an implementation decision.

## Fallback Rule

Start from `ai-agent-documentation-style-guide.md`. Add only the engineering
sections needed to make implementation unambiguous.

## Engineering Section Library

Use the relevant sections below. Do not force every section into every document.

### System Boundary

State what the system, module, API, workflow, or process owns and what it does
not own.

### Ownership

Define the source of truth for data, behavior, documents, decisions, and
external integrations.

### Core Model

Show the smallest useful model. Use tables, bullets, schemas, or simple text
diagrams.

### State Model

Define allowed states, derived states, valid transitions, invalid transitions,
and terminal states.

### Interfaces and Contracts

Define inputs, outputs, events, operations, schemas, protocols, files, or
handoff expectations.

### Required Capabilities

List the behaviors the implementation must support. Keep capabilities separate
from design choices.

### Invariants

State what must always remain true.

### Failure Handling

Describe behavior for retries, duplicates, stale data, partial failure, missing
dependencies, invalid transitions, and corrupted input.

### Reliability Requirements

Describe behavior needed to keep the system trustworthy under expected failure
modes.

### Evolvability Requirements

Describe boundaries, extension points, and model choices that preserve future
change without rewriting core concepts.

### Observability

Define logs, metrics, traces, audit records, or review signals needed to
understand behavior.

### Migration or Rollout

Define sequencing, compatibility, rollback, data migration, and verification
needs.

### Validation

Define tests, checks, review criteria, fixtures, or acceptance criteria needed
to prove the implementation is correct.

## Common Engineering Profiles

### API Contract

- System boundary
- Ownership
- Operations
- Request and response contracts
- Error behavior
- Idempotency
- Validation
- Open questions

### Data Model

- Ownership
- Entities or records
- Relationships
- Invariants
- Migration concerns
- Query or access patterns
- Validation

### Module Spec

- Boundary
- Responsibilities
- Dependencies
- Capabilities
- Interfaces
- Invariants
- Failure handling
- Evolvability requirements

### Operational Runbook

- Ownership
- Trigger
- Preconditions
- Procedure
- Failure modes
- Recovery
- Verification
- Escalation

## Quality Bar

A document should use this fallback only when it answers engineering questions
that the default style cannot answer cleanly.

The result must still be readable by AI agents, with stable headings, explicit
requirements, clear ownership, and testable validation criteria.
