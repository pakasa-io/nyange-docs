# AI Agent Documentation Style Guide

## Purpose

Use this as the default documentation style for this repository.

The style makes any software project document easy for AI agents to parse,
compare, update, and use as implementation context while remaining readable for
humans.

## Applies To

Use this guide for:

- Repository overviews and onboarding docs.
- Product requirements and scope docs.
- Business domain and workflow docs.
- Architecture and ADR docs.
- Module, API, data, and integration docs.
- Mobile UX, screen flow, and interaction docs.
- QA, test strategy, and acceptance docs.
- Runbooks, operations, release, and support docs.
- Planning, decision, risk, and out-of-scope docs.

## Audience

- Primary: AI agents.
- Secondary: engineers, product owners, designers, QA, operators, and reviewers.

## Tone

- Precise.
- Bounded.
- Direct.
- Implementation-ready.
- Source-grounded.

Avoid narrative filler, persuasion, marketing language, and vague qualifiers.

## Format Rules

- Use stable Markdown headings.
- State the document purpose in the first two sections.
- Prefer short sections over long prose.
- Use IDs for facts, requirements, decisions, risks, open questions, workflows,
  interfaces, and rules when they may be referenced later.
- Separate facts, assumptions, decisions, requirements, risks, and open
  questions.
- Prefer tables for inventories, ownership, state, interfaces, and comparison.
- Prefer bullets for constraints, requirements, rules, and acceptance criteria.
- Mark examples clearly as examples.
- Keep links relative for repository-local references.

## Default Document Shape

Use this order unless the document type needs a more specific shape:

1. Title
2. Document Intent
3. Metadata
4. Context
5. Scope
6. Source-Grounded Facts
7. Requirements or Expected Behavior
8. Workflows, Interfaces, or Structure
9. Decisions
10. Risks
11. Validation or Acceptance Criteria
12. Open Questions

## Metadata Fields

Use only fields that matter:

- Status:
- Owner:
- Audience:
- Product stage:
- Related docs:
- Source inputs:
- Last reviewed:

## Common Section Guidance

### Context

Explain the problem, system area, workflow, decision, or operational concern the
document covers.

### Scope

Separate in-scope and out-of-scope items. Keep mid-stage MVP scope limited to
core user value, correctness, reliability, evolvability, operational support,
and implementation blockers.

### Source-Grounded Facts

Facts must come from source material or explicit user input.

Recommended ID prefix: `F-`.

### Requirements or Expected Behavior

Requirements must describe observable behavior, implementation obligations, or
acceptance conditions.

Recommended prefixes:

- `FR-` for functional requirements.
- `NFR-` for non-functional requirements.
- `CR-` for correctness requirements.
- `RR-` for reliability requirements.
- `ER-` for evolvability requirements.
- `UX-` for user experience requirements.

### Workflows, Interfaces, or Structure

Choose the representation that fits the document:

- Workflows: ordered steps, actors, triggers, and outcomes.
- Interfaces: inputs, outputs, events, schemas, operations, and contracts.
- Structure: modules, files, ownership, dependencies, or information hierarchy.

### Decisions

Record decisions separately from facts and requirements.

Recommended ID prefix: `D-`.

### Risks

State what can go wrong, why it matters, and what reduces the risk.

Recommended ID prefix: `R-`.

### Validation or Acceptance Criteria

Define how a reader or agent can verify the document has been implemented,
followed, or satisfied.

Recommended ID prefix: `AC-`.

### Open Questions

Use open questions only for unresolved items that block or materially shape
future work.

Recommended ID prefix: `OQ-`.

## Document Profiles

Use these profiles as starting points.

### Product or Requirements Doc

- Context
- Users or actors
- Scope
- Requirements
- User workflows
- Acceptance criteria
- Out of scope
- Open questions

### Architecture or ADR Doc

- Context
- Decision
- Options considered
- Consequences
- Ownership
- Risks
- Validation
- Follow-up

### Module or API Doc

- Context
- Boundary
- Ownership
- Capabilities
- Interfaces
- Data contracts
- Requirements
- Risks
- Open questions

### Workflow or UX Doc

- Context
- Actors
- Entry points
- Flow steps
- States and error states
- UX requirements
- Acceptance criteria
- Open questions

### Operations or Runbook Doc

- Context
- Ownership
- Triggers
- Procedure
- Expected outcomes
- Failure modes
- Rollback or recovery
- Verification

### QA or Test Doc

- Context
- Scope
- Test objectives
- Test matrix
- Acceptance criteria
- Known risks
- Coverage gaps

## Quality Bar

A document follows this style when:

- An AI agent can identify the document purpose within the first two sections.
- Scope boundaries are explicit.
- Requirements or expected behavior are testable.
- Terms and owners are stable.
- Decisions are not mixed with facts or assumptions.
- Open questions are actionable.
- Future changes can reference stable headings or IDs.
