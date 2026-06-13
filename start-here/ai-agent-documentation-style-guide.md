# AI Agent Documentation Style Guide

## Purpose

Use this as the canonical style and tone for every documentation chunk in this
repository.

This guide does not prescribe a document inventory or require fixed templates.
It defines how documentation should read, how it should be structured, and how
it should preserve meaning across many document kinds.

Prefer small, focused documents. Size each document around a coherent reader
task, and split only when that improves clarity, retrieval, ownership, or
evolvability.

## Reader Model

- Primary reader: AI agents.
- Secondary readers: humans reviewing, editing, or implementing from the docs.

Assume an AI agent may read one chunk in isolation. Each chunk should provide
enough local context to be useful without forcing the reader to infer intent.

## Tone

- Precise.
- Concise.
- Direct.
- Neutral.
- Source-grounded.
- Implementation-aware.

Avoid narrative filler, persuasion, marketing language, hidden assumptions, and
large prose blocks.

## Core Rule

Write each chunk so an AI agent can answer:

- What is this about?
- What is known?
- What is decided?
- What is required or constrained?
- What remains unresolved?
- How should this be used by the next agent or contributor?

## Universal Style Rules

- State the chunk's intent near the top.
- Use stable Markdown headings.
- Include only sections that add clarity.
- Keep each document focused around a coherent reader task.
- Split or index material when doing so improves clarity, retrieval, ownership,
  or evolvability.
- Prefer short paragraphs, bullets, and tables over long prose.
- Use consistent terms for the same concept.
- Separate facts, assumptions, decisions, business rules, requirements, risks,
  terms, dependencies, and open questions when they appear.
- Use IDs only when an item is likely to be referenced, traced, tested, or
  updated later.
- Keep examples clearly marked as examples.
- Use relative links for repository-local references.
- Do not add boilerplate sections just to satisfy a template.

## Chunk Shape

Most chunks should follow this lightweight shape:

1. Heading
2. Intent
3. Context needed to understand the chunk
4. Main content
5. Decisions, constraints, risks, or open questions when present
6. Links or source references when useful

Short chunks may use fewer sections. Dense or high-impact chunks may use more.
If a chunk needs many sections, apply the document sizing algorithm before
adding more structure.

## Document Sizing

Use this algorithm when creating, expanding, or refactoring documentation.

1. Identify the reader task.
   State what the next AI agent or human should be able to do after reading the
   document.
2. Find atomic units.
   Mark the independently useful concepts, decisions, workflows, rules,
   interfaces, examples, risks, and open questions.
3. Assess cohesion.
   Keep units together when they share the same reader task, source context,
   ownership, lifecycle, and change cadence.
4. Assess coupling.
   Split units when they can change independently, have different owners,
   target different reader tasks, or require different levels of detail.
5. Choose the document shape.
   Use one focused document, multiple sibling documents, or a short index that
   links focused documents.
6. Preserve local context.
   Each split document must explain its own intent, key assumptions, and links
   to related documents.
7. Validate navigability.
   A reader should know where to start, what to read next, and which document is
   authoritative for each decision or rule.

Prefer splitting when:

- A reader must skim around unrelated sections to complete one task.
- Different sections have different owners or change at different speeds.
- Stable decisions are mixed with exploratory proposals.
- Reference material is mixed with workflow or implementation guidance.
- One section needs frequent updates while the rest should remain stable.

Prefer keeping content together when:

- The parts are only useful together.
- Splitting would force readers to jump between files to understand one idea.
- The document is short, cohesive, and easy to update.
- The same owner, source context, and lifecycle apply to all sections.

## Optional Labels

Use these labels when they help clarify the content:

- `Facts`: Source-grounded statements treated as true.
- `Assumptions`: Statements used for progress but not yet proven.
- `Terms`: Canonical vocabulary, definitions, aliases, or forbidden synonyms.
- `Actors`: Users, systems, agents, services, or teams involved.
- `Decisions`: Chosen direction or settled interpretation.
- `Business Rules`: Domain or product constraints that define valid behavior.
- `Requirements`: Observable behavior or implementation obligations.
- `Constraints`: Limits, boundaries, or rules that narrow valid solutions.
- `Dependencies`: Required upstream docs, systems, APIs, decisions, or
  assumptions.
- `Permissions`: Who can perform or access something.
- `Data`: Records, fields, payloads, or stored information.
- `Workflows`: Ordered actions, actors, triggers, and outcomes.
- `Events`: Domain, system, or workflow events worth naming.
- `Interfaces`: Inputs, outputs, events, schemas, or contracts.
- `Boundary`: What the subject owns and does not own.
- `Ownership`: The source of truth for data, behavior, decisions, or
  integrations.
- `State`: Allowed states, derived states, valid transitions, invalid
  transitions, and terminal states.
- `Invariants`: Conditions that must always remain true.
- `Failure Handling`: Expected behavior for retries, duplicates, stale data,
  partial failure, missing dependencies, invalid transitions, or corrupted
  input.
- `Evolvability`: Boundaries, extension points, or model choices that preserve
  future change without rewriting core concepts.
- `Observability`: Logs, metrics, traces, audit records, or review signals
  needed to understand behavior.
- `Migration or Rollout`: Sequencing, compatibility, rollback, migration, and
  verification needs.
- `Outcomes`: Expected end states, results, or externally visible effects.
- `Edge Cases`: Boundary scenarios that clarify behavior.
- `Risks`: What can go wrong, why it matters, and what reduces the risk.
- `Examples`: Concrete examples that clarify abstract rules or edge cases.
- `Acceptance Criteria`: Checks that prove the work or document is satisfied.
- `Validation`: Tests, checks, fixtures, review criteria, or verification steps
  that prove correctness.
- `Non-Goals`: Explicitly excluded intent when `Out of Scope` is too broad.
- `Open Questions`: Unresolved items that materially affect future work.

Use only the labels that match the chunk.

## Technical Detail Guidance

Use technical detail blocks only when they make a specific piece of information
clearer than prose would.

Do not add technical sections just because a chunk mentions architecture, APIs,
data, operations, reliability, or evolvability. Those topics still use the same
default style. Add technical blocks only where they remove ambiguity.

Good uses:

- State owned vs. not owned responsibilities as a `Boundary`.
- Express an input/output expectation as an `Interface`.
- List valid and invalid lifecycle movement as `State`.
- Record non-negotiable correctness rules as `Invariants`.
- Define retry, duplicate, or partial-failure behavior as `Failure Handling`.
- Define verification steps as `Acceptance Criteria` or `Validation`.

## ID Guidance

IDs are useful when content needs traceability. Avoid IDs for disposable notes or
single-use prose.

Recommended prefixes:

- `F-` for facts.
- `A-` for assumptions.
- `T-` for terms.
- `ACT-` for actors.
- `D-` for decisions.
- `BR-` for business rules.
- `REQ-` for requirements.
- `C-` for constraints.
- `DEP-` for dependencies.
- `PERM-` for permissions.
- `DATA-` for data items.
- `EVT-` for events.
- `R-` for risks.
- `OUT-` for outcomes.
- `EDGE-` for edge cases.
- `EX-` for examples.
- `AC-` for acceptance criteria.
- `V-` for validation items.
- `NG-` for non-goals.
- `OQ-` for open questions.

## Scope Guidance

Treat scope as a constraint, not a required section.

When scope appears, keep mid-stage MVP scope limited to core user value,
correctness, reliability, evolvability, operational support, and implementation
blockers. Move ideas that exceed that bar to `out-of-scope/` with a revisit
trigger.

## Quality Bar

A chunk follows this style when:

- Its purpose is clear without outside explanation.
- It uses precise, stable terms.
- It distinguishes known facts from assumptions and decisions.
- It avoids unnecessary structure.
- It is small and focused, or explicitly split into linked documents.
- It preserves enough context for future agents to continue the work.
- It is easy to update without rewriting unrelated content.
