# Coding Standards

## Metadata

- `id`: `coding-standards`
- `status`: `required`
- `scope`: production implementation code, tests, migrations, scripts, generated artifacts, and implementation-adjacent documentation
- `authority`: this document refines `docs/02-engineering/architecture.md`, `docs/02-engineering/package-layout.yml`,
  `docs/02-engineering/api-standards.yml`, `docs/02-engineering/data-access-policy.yml`, and
  `docs/02-engineering/testing-strategy.yml`; unit-test-specific requirements are refined by
  `docs/02-engineering/go-unit-testing.spec.md`

## Purpose

These standards define the minimum acceptable implementation quality for the Nyange backend. They are written for human maintainers
and AI implementation agents. They are binding unless a later, more specific authority document states a stricter rule.

Implementation work MUST preserve the approved architecture: one Go modular monolith, Fiber v3 HTTP edge, Postgres source of truth,
GORM repositories with explicit data mappers, edge-managed UoW transactions, Cognito authentication, Cedar authorization, and
container-first local/test/deploy execution.

## Authority And Exceptions

- **ENG-CODE-001**: Engineers and agents MUST read the relevant authority documents before changing behavior. At minimum, read this
  document, `docs/manifest.yml`, `docs/00-start-here/document-conventions.md`, and the engineering/module documents governing the
  files being changed.
- **ENG-CODE-002**: A user request, implementation plan, generated code suggestion, or framework convention MUST NOT override the
  approved module boundaries, API contracts, authorization model, transaction boundaries, data access policy, or testing strategy.
- **ENG-CODE-003**: Exceptions MUST be explicit. If a standard cannot be met, document the violated rule ID, why the exception is
  necessary, the risk, the compensating control, and the follow-up needed to remove the exception.
- **ENG-CODE-004**: Implementation code MUST stay aligned with the specs. When code changes an API shape, state transition, schema,
  invariant, error code, authorization action, event payload, or workflow, update the owning specification in the same change.
- **ENG-CODE-005**: Documentation files under this package MUST remain contract-oriented. Do not add implementation snippets to
  authority docs unless the document explicitly permits examples.

## Design And Boundaries

- **ENG-CODE-010**: Keep business behavior inside the owning module. Kernel packages contain only domain-neutral primitives.
  Platform packages contain technical infrastructure only.
- **ENG-CODE-011**: Core business code MUST NOT import Fiber, GORM, AWS SDKs, logging adapters, config libraries, HTTP clients,
  database drivers, or other edge/platform dependencies.
- **ENG-CODE-012**: Modules MUST interact through documented contracts or ports. A module MUST NOT write another module's table
  directly except through a documented contract inside an edge-managed UoW.
- **ENG-CODE-013**: Do not create generic business buckets named `common`, `utils`, `helpers`, or `core`. Shared code must have a
  specific domain-neutral purpose and live in the proper kernel or platform package.
- **ENG-CODE-014**: Prefer simple, explicit types and functions over framework magic, reflection-heavy abstractions, or generic
  indirection. Add abstractions only when they reduce real duplication or enforce a documented boundary.
- **ENG-CODE-015**: Use the project's domain vocabulary in names. Avoid abbreviations, role-like names, and ambiguous nouns when the
  module specs provide precise language.
- **ENG-CODE-016**: Do not add post-MVP features while implementing an MVP slice. Deferred capabilities remain unavailable unless the
  authority docs are updated first.

## Go Source Standards

- **ENG-CODE-020**: All Go source MUST pass `gofmt` and project import formatting. Formatting is a gate, not a review preference.
- **ENG-CODE-021**: Package APIs MUST be small and intentional. Export only types, functions, constants, and interfaces used outside
  the package.
- **ENG-CODE-022**: Avoid package-level mutable state. Runtime dependencies flow through constructors, provider sets, or explicit
  function parameters.
- **ENG-CODE-023**: Prefer domain-specific value types for domain identifiers, money, timestamps, status values, and document numbers
  instead of passing raw strings or integers through business logic.
- **ENG-CODE-024**: Request-controlled input MUST be validated at the edge before mutation. Core logic must still enforce invariants
  and fail closed when called from tests or other modules.
- **ENG-CODE-025**: Panics MUST NOT be used for normal request, validation, authorization, state, persistence, or provider failure
  paths. Panics are acceptable only for impossible programmer errors during startup/composition.
- **ENG-CODE-026**: Error handling MUST preserve machine-readable error codes and user-safe messages. Do not leak SQL, provider,
  secret, token, PII, or stack details into API responses.
- **ENG-CODE-027**: Comments should explain non-obvious decisions, invariants, or boundary constraints. Do not comment mechanical
  assignments, obvious control flow, or generated boilerplate.
- **ENG-CODE-028**: Use ASCII for implementation files unless a file or domain value has a clear reason to use non-ASCII text.

## API And Handler Standards

- **ENG-CODE-030**: Fiber handlers are edge adapters. They parse requests, call authentication/authorization/idempotency/validation
  middleware, invoke application use cases, and map DTOs. They MUST NOT contain business workflows.
- **ENG-CODE-031**: Module OpenAPI files are the canonical HTTP wire contract. Handlers MUST match operation IDs, DTO names,
  response envelopes, error codes, Cedar metadata, path parameters, and idempotency metadata declared by the owning module.
- **ENG-CODE-032**: All success and error responses MUST use the shared envelopes from `docs/02-engineering/api-standards.yml`.
- **ENG-CODE-033**: Mutating endpoints MUST require `Idempotency-Key`, compute request fingerprints consistently, and replay stored
  responses according to `docs/02-engineering/idempotency-and-concurrency.yml`.
- **ENG-CODE-034**: Disallowed origins, oversized bodies, authentication failures, inactive users, authorization denials, malformed
  IDs, and validation failures MUST be rejected in the documented order before idempotency or mutation work where the relevant
  engineering document requires it.
- **ENG-CODE-035**: API DTOs MUST be explicit client contracts, not ORM models, domain object dumps, or generic maps.
- **ENG-CODE-036**: API reads MUST return only data the actor is authorized to see. Redaction requirements in workflow and privacy
  docs are implementation requirements, not presentation preferences.

## Transactions And Workflow Standards

- **ENG-CODE-040**: Mutating business workflows MUST run inside an edge-managed UoW transaction unless the owning workflow explicitly
  documents a different transaction boundary.
- **ENG-CODE-041**: State, authorization-sensitive resource resolution, idempotency records, audit events, domain events, and owned
  persistence changes MUST occur in the documented transaction order.
- **ENG-CODE-042**: Workflows MUST recheck state and business invariants inside the transaction after acquiring the necessary row
  locks or equivalent concurrency control.
- **ENG-CODE-043**: External side effects MUST NOT run inside an uncommitted database transaction. Persist an outbox record or commit
  first, then perform the side effect through the documented adapter path.
- **ENG-CODE-044**: Rollback behavior is part of the workflow contract. A failed command MUST leave no partial mutation unless the
  owning workflow explicitly documents durable failure records.
- **ENG-CODE-045**: Idempotent replay and no-op duplicate commands MUST NOT allocate new document numbers, increment aggregate
  versions, append duplicate ledgers, or emit duplicate domain events.

## Persistence And Migration Standards

- **ENG-CODE-050**: Postgres is the source of truth. Production code MUST NOT rely on in-memory state, application caches, or
  provider state as authoritative business state.
- **ENG-CODE-051**: Domain objects MUST NOT contain persistence tags or GORM-specific fields. Repositories map between persistence
  models and domain/application types explicitly.
- **ENG-CODE-052**: Migrations are the source of truth for constraints and indexes. GORM associations MUST NOT replace migration
  defined foreign keys, uniqueness, check constraints, or indexes.
- **ENG-CODE-053**: Production MUST NOT run GORM AutoMigrate. All schema changes are forward-only migrations with smoke coverage.
- **ENG-CODE-054**: Append-only ledgers, history rows, audit rows, and domain events MUST stay append-only. Corrections use documented
  compensating records, never destructive updates.
- **ENG-CODE-055**: Mutable aggregate root tables MUST use server-managed version columns exactly as defined in the data access and
  API standards.
- **ENG-CODE-056**: Schema definitions, migrations, repository mappers, and OpenAPI DTOs MUST agree on ULID casing, timestamp UTC
  serialization, money minor units, enum vocabulary, nullability, and length constraints.

## Authorization And Security Standards

- **ENG-CODE-060**: Authorization is fine-grained Cedar action/resource evaluation only. Do not add roles, role columns, role tables,
  Cognito group authorization, role-like policy bundles, or tenant-scoped user ownership shortcuts.
- **ENG-CODE-061**: Protected operations MUST authenticate Cognito access tokens and resolve the application user before Cedar
  authorization.
- **ENG-CODE-062**: Non-ACTIVE users and disabled service principals MUST fail closed before endpoint-specific work.
- **ENG-CODE-063**: Authorization decisions MUST be auditable where required by the identity, audit, and platform specs.
- **ENG-CODE-064**: Secrets, bearer tokens, OTPs, cloud credentials, provider API keys, raw delivery addresses, recipient names,
  delivery instructions, notification SMS bodies, and provider payloads MUST NOT appear in logs, traces, metrics, events,
  idempotency records, readiness output, or API responses unless an authority doc explicitly allows a redacted form.
- **ENG-CODE-065**: LocalStack and local AWS endpoint overrides are allowed only in local/test environments.

## Observability And Operations Standards

- **ENG-CODE-070**: Runtime code MUST emit structured, bounded-cardinality logs, traces, and metrics according to the observability
  and operational alerting specs.
- **ENG-CODE-071**: Logs MUST include request, trace, actor, action, resource, and workflow identifiers where available and safe.
- **ENG-CODE-072**: Readiness and liveness endpoints MUST stay outside business workflows and MUST NOT leak dependency details or
  secret-bearing configuration.
- **ENG-CODE-073**: Background workers MUST use service-principal authorization, bounded leases, retry limits, redaction, and failure
  isolation where required by the module specs.
- **ENG-CODE-074**: Runtime configuration MUST be typed, validated at startup, and fail closed when required production settings are
  missing or unsafe.

## Configuration Standards

- **ENG-CODE-075**: Environment-specific values MUST be externalized through the documented configuration system. Do not hard-code
  deployment endpoints, ports, credentials, regions, provider names, allowed browser origins, timeouts, retention windows, currency,
  throttles, feature switches, or environment-dependent behavior in business or adapter code.
- **ENG-CODE-076**: Defaults MUST be safe, explicit, and appropriate only for local/test use unless an authority doc states otherwise.
  Production-required settings MUST be deployment-injected and startup-validated.
- **ENG-CODE-077**: Secrets MUST be read only through approved secret/config providers. Secrets MUST NOT be committed to source,
  generated fixtures, tests, migration files, sample payloads, logs, traces, metrics, or PR descriptions.
- **ENG-CODE-078**: Configuration keys MUST be typed, named by domain/runtime purpose, documented in the owning platform config spec,
  and covered by validation tests for missing, malformed, unsafe, and environment-forbidden values.

## Testing Standards

- **ENG-CODE-080**: Every implementation slice MUST include the required tests from `docs/02-engineering/testing-strategy.yml` for
  behavior it introduces or materially changes. Go package-local unit tests MUST follow
  `docs/02-engineering/go-unit-testing.spec.md`.
- **ENG-CODE-081**: Domain and application tests should cover invariant-heavy behavior without external services. Adapter,
  repository, migration, acceptance, and provider tests that need dependencies MUST use Testcontainers.
- **ENG-CODE-082**: No integration, adapter, migration, or acceptance test suite may require a shared pre-running local development
  stack.
- **ENG-CODE-083**: Public HTTP behavior MUST have OpenAPI contract coverage for request DTOs, response envelopes, path/query
  parameters, idempotency metadata, Cedar metadata, and expected operation-specific error codes.
- **ENG-CODE-084**: Defects that reach review or production MUST receive a regression test at the lowest level that would have
  caught the failure.
- **ENG-CODE-085**: Tests MUST use deterministic fixtures, deterministic IDs where the docs require them, isolated data, and explicit
  assertions. Avoid timing-dependent sleeps and broad snapshot assertions.
- **ENG-CODE-086**: Verification commands run for a change MUST be reported in the final change summary. If a relevant check cannot
  run, report the reason and residual risk.
- **ENG-CODE-087**: Unit-testable behavior MUST be developed with a red/green TDD loop: write one failing unit test for one observable
  behavior, prove the test fails for the expected reason, write the smallest implementation needed to pass, prove the test passes,
  then refactor only while tests stay green.
- **ENG-CODE-088**: Red/green TDD MUST proceed in vertical slices. Do not write a batch of speculative tests before implementation,
  and do not implement untested behavior ahead of the next failing test.
- **ENG-CODE-089**: Integration, contract, migration, and acceptance tests supplement unit-level red/green TDD; they do not replace
  unit tests for domain, application, mapper, validation, configuration, and other deterministic logic unless a documented exception
  explains why no useful unit boundary exists.

## Dependency And Generation Standards

- **ENG-CODE-090**: Add third-party dependencies only when they solve a real project problem better than the standard library or an
  existing dependency. New dependencies must be versioned, container-compatible, and compatible with the architecture boundaries.
- **ENG-CODE-091**: Generated artifacts MUST be reproducible from checked-in sources and generator commands. Mark generated files
  clearly and do not hand-edit them unless the generator workflow explicitly permits it.
- **ENG-CODE-092**: Build, migration, seed, test, and deployment commands MUST work through the container-first path documented by the
  engineering and platform specs.
- **ENG-CODE-093**: Scripts must fail fast, print actionable errors, avoid hidden host-only prerequisites, and avoid writing secrets
  or PII to stdout/stderr.

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

## Change Review Checklist

Every implementation change MUST be reviewable against these questions:

- Which module, platform area, or kernel primitive owns the changed behavior?
- Which authority docs and contracts changed or confirmed the behavior?
- Does the change preserve module boundaries and dependency direction?
- Does the change preserve authentication, authorization, idempotency, transaction, redaction, and observability requirements?
- Does persistence remain migration-defined, forward-only, and consistent with schema docs?
- Do tests cover happy path, denial, validation failure, business/state conflict, idempotent replay, and rollback/no-mutation where
  the current slice requires them?
- Did unit-testable behavior follow a red/green TDD loop with one behavior per cycle?
- Are environment-specific values externalized through typed, validated configuration?
- Is active ticket work isolated in a dedicated worktree when parallel implementation work is in progress?
- Is the work on a ticket-scoped feature branch whose name includes the GitHub ticket number?
- Is there a GitHub ticket, and will the PR summary include the ticket number?
- Were relevant formatting, lint, unit, integration, contract, migration, or acceptance checks run and reported?
