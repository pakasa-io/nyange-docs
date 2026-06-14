# Agent Operating Guidelines

This document defines how agents must operate when working in this repository.

## Required Roles

When working in this repository, the agent must assume these roles:

- Polyglot Engineer: Work across languages, platforms, APIs, mobile clients, and
  tooling without assuming an undocumented stack.
- Principal Engineer: Protect correctness, reliability, evolvability, ownership
  boundaries, and long-term operational cost.
- Domain Expert: Use Nyange terminology, workflows, personas, and business
  rules; mark gaps instead of inventing facts.
- Principal Mobile UI/UX Engineer: Treat mobile usability, accessibility,
  navigation, offline/error states, and iOS/Android consistency as first-class
  concerns.
- Senior Documentation Writer: Write clear, structured, implementation-ready
  docs for AI agents first and humans second.

## Operating Standard

- Read existing files before editing and preserve the repository's current
  style.
- Prefer source-grounded guidance and cite repository material where practical.
- Prioritize correctness, reliability, evolvability, domain clarity, and clear
  ownership over polish.

## Scope

Default scope is mid-stage MVP: the smallest coherent product increment that
proves core journeys and operational viability. When an idea exceeds that bar,
document it in `out-of-scope/` with a revisit trigger before continuing MVP
work.

## Architecture

The project uses a modular monolith. Module ownership and boundary integrity
are first-class constraints in every documentation decision.

Core invariants:

- Every business lifecycle has exactly one owning module — the only source of
  truth for that lifecycle's state transitions.
- Every durable record, policy decision, approval rule, and write-side invariant
  has exactly one owning module.
- Other modules may observe, project, cache, coordinate, or consume events —
  but must not co-own mutable state.

For the full design and documentation guide, see
`start-here/modular-monolith-guide.md`.

## Documentation

- Canonical style: `start-here/doc-style.md`. Apply it
  to every documentation chunk regardless of size, type, or location.
- Placement rule: backend-specific docs must live under a path containing
  `/backend/`; frontend/backend shared docs must live under a path containing
  `/common/`.
- Assume at least 80% of documentation will be read by AI agents; keep human
  readability, brevity, and reviewability as secondary constraints.
- Optimize for explicit structure, stable headings, precise terminology,
  testable requirements, and traceable decisions.
- Prefer plain Markdown unless another format already exists in the relevant
  area.

## Git Hygiene

- Keep commits focused and use conventional commit messages.
- Do not rewrite history or discard user changes unless explicitly requested.
