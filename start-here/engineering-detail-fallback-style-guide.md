# Engineering Detail Fallback Style Guide

## Purpose

Use this as an optional technical-detail layer when a documentation chunk needs
stronger engineering precision.

This guide does not define a separate document type. Apply it inside any chunk
when implementation detail, correctness, reliability, or evolvability needs to
be made explicit.

## When To Use

Use this fallback when the chunk must clarify:

- Ownership or boundaries.
- Interfaces, events, schemas, or data contracts.
- State, lifecycle, or transitions.
- Invariants or correctness rules.
- Failure handling and retry behavior.
- Reliability expectations.
- Evolvability constraints.
- Migration, rollout, or compatibility concerns.
- Validation, tests, or acceptance checks.

## Tone

- Direct.
- Constraint-focused.
- Correctness-oriented.
- Operationally aware.

Keep the default documentation voice. Add engineering detail only where it
removes ambiguity.

## Fallback Rule

Start with `ai-agent-documentation-style-guide.md`. Add only the technical
blocks needed for the chunk at hand.

Do not force a technical block into a chunk just because it exists here.

## Optional Technical Blocks

### Boundary

State what the subject owns and does not own.

### Ownership

Name the source of truth for data, behavior, decisions, or integrations.

### Contract

Define inputs, outputs, events, schemas, protocols, files, or handoff
expectations.

### State

Define allowed states, derived states, valid transitions, invalid transitions,
and terminal states.

### Invariants

State what must always remain true.

### Failure Handling

Describe behavior for retries, duplicates, stale data, partial failure, missing
dependencies, invalid transitions, or corrupted input.

### Reliability

Describe behavior needed to keep the system trustworthy under expected failure
modes.

### Evolvability

Describe boundaries, extension points, or model choices that preserve future
change without rewriting core concepts.

### Observability

Define logs, metrics, traces, audit records, or review signals needed to
understand behavior.

### Migration or Rollout

Define sequencing, compatibility, rollback, migration, and verification needs.

### Validation

Define tests, checks, fixtures, review criteria, or acceptance criteria needed
to prove correctness.

## Quality Bar

The fallback is working when:

- It makes implementation expectations clearer.
- It does not change the chunk's primary purpose.
- It adds constraints without adding ceremony.
- It keeps ownership, behavior, and validation explicit.
- It remains readable by AI agents using the default documentation style.
