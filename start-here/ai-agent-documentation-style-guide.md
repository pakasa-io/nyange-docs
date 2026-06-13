# AI Agent Documentation Style Guide

## Purpose

Use this as the default style and tone for every documentation chunk in this
repository.

This guide does not prescribe a document inventory or require fixed templates.
It defines how documentation should read, how it should be structured, and how
it should preserve meaning across many document kinds.

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
- Prefer short paragraphs, bullets, and tables over long prose.
- Use consistent terms for the same concept.
- Separate facts, assumptions, decisions, requirements, risks, and open
  questions when they appear.
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

## Optional Labels

Use these labels when they help clarify the content:

- `Facts`: Source-grounded statements treated as true.
- `Assumptions`: Statements used for progress but not yet proven.
- `Decisions`: Chosen direction or settled interpretation.
- `Requirements`: Observable behavior or implementation obligations.
- `Constraints`: Limits, boundaries, or rules that narrow valid solutions.
- `Workflows`: Ordered actions, actors, triggers, and outcomes.
- `Interfaces`: Inputs, outputs, events, schemas, or contracts.
- `Risks`: What can go wrong, why it matters, and what reduces the risk.
- `Acceptance Criteria`: Checks that prove the work or document is satisfied.
- `Open Questions`: Unresolved items that materially affect future work.

Use only the labels that match the chunk.

## ID Guidance

IDs are useful when content needs traceability. Avoid IDs for disposable notes or
single-use prose.

Recommended prefixes:

- `F-` for facts.
- `A-` for assumptions.
- `D-` for decisions.
- `REQ-` for requirements.
- `C-` for constraints.
- `R-` for risks.
- `AC-` for acceptance criteria.
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
- It preserves enough context for future agents to continue the work.
- It is easy to update without rewriting unrelated content.
