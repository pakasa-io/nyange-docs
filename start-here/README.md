# Start Here

This folder contains repository-level documentation style standards.

## Documentation Style

- Canonical style:
  `doc-style.md`

Use the canonical guide for every documentation chunk, regardless of size, type,
or location. It defines shared style and tone; it does not prescribe a fixed
document set.

Prefer small, focused documents. Size each document around a coherent reader
task, and split only when that improves clarity, retrieval, ownership, or
evolvability.

## Documentation Placement

This repository contains frontend, backend, and shared documentation.

- Backend-specific docs must live under a path containing `/backend/`.
- Frontend/backend shared docs must live under a path containing `/common/`.
- When editing from a backend repository, use the `/backend/` path unless the
  rule, decision, or guidance applies to both frontend and backend.

## Reader Model

Assume documentation is read primarily by AI agents. Optimize for local context,
stable headings, precise terms, explicit constraints, and traceable decisions.
