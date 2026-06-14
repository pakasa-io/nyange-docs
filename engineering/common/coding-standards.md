# Coding Standards

## Purpose

These standards define the minimum acceptable implementation quality.

## Worktree, Branch, Ticket, And PR Standards

- **ENG-CODE-098**: Active ticket work SHOULD use a dedicated git worktree for that ticket-scoped branch. Engineers and agents
  SHOULD avoid switching branches inside a dirty or active worktree; create or switch to a separate ticket worktree instead.
- **ENG-CODE-099**: Feature work MUST be developed on a ticket-scoped branch, not on `main`. Branch names MUST follow the standard
  short-prefix shape `<type>/gh-123-short-slug`, where `<type>` is `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`, or
  `perf`. The branch name MUST include the GitHub ticket number in branch-safe form such as `gh-123` plus a short lowercase slug.
- **ENG-CODE-100**: All changes MUST be delivered through a pull request before merging to the `main` branch. Direct commits to
  `main` are prohibited except for explicitly approved emergency repository administration.
- **ENG-CODE-101**: Every pull request MUST reference a GitHub ticket. The ticket number MUST appear in the PR summary description
  in a consistent, searchable form such as `Ticket: #123`.
- **ENG-CODE-102**: Pull requests MUST summarize behavior changes, spec/doc changes, tests run, red/green TDD evidence for
  unit-testable behavior, migration/config impact, and any documented standard exceptions.
- **ENG-CODE-103**: Pull requests MUST NOT merge while required tests, contract checks, migration checks, lint/format checks, or
  required reviews are failing or missing.
