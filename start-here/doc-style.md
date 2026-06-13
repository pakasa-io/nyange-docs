# AI Agent Documentation Style Guide

## Purpose

Use this as the canonical style and tone for every documentation chunk in this
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
- Keep each document focused around a coherent reader task.
- Split or index material when doing so improves clarity, retrieval, ownership,
  or evolvability.
- Prefer short paragraphs, bullets, and tables over long prose.
- Use consistent terms for the same concept.
- Separate content by type using the Optional Labels when distinctions matter.
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
2. List atomic units.
   Identify each independently useful concept, decision, workflow, rule,
   interface, example, risk, or open question.
3. Assess cohesion and coupling.
   Keep units together when they share the same reader task, owner, lifecycle,
   and change cadence.
   Separate units when they can change independently, have different owners,
   target different reader tasks, or require different levels of detail.
4. Choose the document shape.
   - One document: all units serve one task and belong to one owner.
   - Sibling documents: units are independent but peers — link them to each
     other.
   - Index + focused documents: many siblings exist and a reader needs
     orientation first.
5. Verify navigability.
   A reader should know where to start, what to read next, and which document
   is authoritative for each decision or rule. If not, return to step 4.

After splitting, each new document must:

- State its own intent near the top.
- Include key assumptions the reader needs.
- Link to related documents.

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
- Express multi-branch conditions, eligibility gates, threshold decisions, or
  formulas as pseudo-code (see Pseudo-Code section below).

## Pseudo-Code for Conditions and Logic

Use concise pseudo-code when a condition, branch, gate, or formula is clearer
as logic than as prose. Keep the notation minimal: any reader should follow it
without programming-language knowledge.

**When to use:**

- Multi-branch threshold decisions
- Eligibility gates with AND / OR / NOT logic
- State-transition guards with conditions
- Price or quantity formulas with multiple terms
- A condition that suspends or permits a set of independent operations
- Field, evidence, or action obligations that vary by context (required / optional / prohibited)
- Ordered time-offset triggers, expiry windows, or SLA escalation chains
- Multi-criteria ranking where the order of criteria matters
- A set of state changes that must all succeed or all fail together
- Rate limits, rolling caps, or alert thresholds measured over a time window
- Any condition that needs more than one prose sentence to express unambiguously

**When not to use:**

- Single-condition facts that read clearly in one phrase
- Explanations of why a rule exists or business rationale for a decision
- Persona descriptions or sequential workflow steps with no branching
- Simple item lists or required-field enumerations (use a table or bullets instead)
- Information already captured cleanly in a table (permission matrix, fee schedule, field list)

**Notation:**

| Symbol | Meaning | Pattern |
|---|---|---|
| `→` | results in / transitions to | all |
| `if` / `else if` / `else` | branching | all |
| `AND` / `OR` / `NOT` | boolean operators (uppercase) | all |
| `>=` `<=` `>` `<` `==` `!=` | comparisons | all |
| `:=` | is defined as | all |
| `//` | inline note | all |
| `??` | fallback when left side is absent | formula |
| `×` | multiplication | formula |
| `in [a–b]` | range / interval | threshold |
| `none` | absent / not configured | eligibility gate, formula |
| `IN [...]` / `NOT IN [...]` | set membership / exclusion | conditional requirements |
| `blocked:` / `allowed:` | operations gated by a condition | blocked/allowed set |
| `ASC` / `DESC` | sort direction | priority ranking |
| `count(x, window=T)` | rolling aggregate within a time window | windowed threshold |
| `t = X` | time offset from a reference point | time-sequence |

Use plain English identifiers. Indent to show nesting. Wrap in a fenced code
block with no language tag.

**Multi-branch threshold:**

```
if refund_amount < 50,000 UGX           → COLLECTIBLE        // no approval required
if refund_amount in [50,000–500,000]    → PENDING_APPROVAL (Outlet Manager)
if refund_amount > 500,000 UGX          → PENDING_APPROVAL (Super Admin)
```

**Eligibility gate:**

```
outlet eligible :=
  zone_match
  AND operational
  AND supports_delivery_mode
  AND supports_payment_method
  AND sufficient_stock
  AND vendor_policy_met
  AND (capacity_limit == none OR active_orders < capacity_limit)
```

**State-transition guard:**

```
STAFF_VERIFIED →
  if verified_amount == order_total  → PAID
  if verified_amount <  order_total  → PARTIALLY_PAID
  if verified_amount >  order_total  → OVERPAID
```

**Formula:**

```
express_fee        := base_zone_fee × express_multiplier
express_multiplier := outlet_override ?? global_default (1.5)
```

**Blocked/allowed set** — a condition that gates multiple independent operations.
Use when a flag or status suspends a list of actions rather than routing a
single value to a single outcome.

```
if closing_overdue AND NOT super_admin_override:
  blocked:  cash_refund_payouts
  blocked:  outside_guardrail_price_changes
  blocked:  manual_ledger_adjustments
  allowed:  order_placement, picking, dispatch, cod_collection
```

**Conditional requirements** — output is a field or action obligation (required /
optional / prohibited), not a state or value. Use for evidence rules, validation
gates, and audit obligations.

```
GPS required when:
  action IN [arrival, failed_delivery, doorstep_defect]
  AND device_can_provide_gps

photo required when:
  damaged_returned_cylinder AND physical_cylinder_present
  OR defective_outgoing_cylinder

photo NOT required when:
  missing_cylinder  // time + location facts + reason note instead
```

**Time-sequence / expiry** — ordered time offsets as triggers. Use for expiry
windows, SLA escalation, rate-limit resets, and retention deadlines.

```
t = 0    → clock starts; reservation active
t = 30m  → warning sent; reservation still active
t = 60m  → if no reference submitted  → CANCELLED, reservation released
           if reference submitted      → clock stops permanently
```

**Priority ranking** — ordered multi-criteria sort where the output is a ranking,
not a yes/no decision. Use for allocation tie-breaking, queue ordering, and
assignment policies. List criteria in priority order; earlier criteria take
precedence over later ones.

```
rank outlets by:
  1. distance_to_customer     ASC
  2. delivery_fee             ASC
  3. active_order_load        ASC
  4. outlet_priority_score    DESC
```

**Atomic commitment group** — a set of state changes that must all succeed or all
fail together. Use to make the boundary of an atomic operation explicit. List
every participant; add a comment stating the all-or-nothing constraint.

```
PIN confirmation commits atomically:
  delivery_status
  order_status
  stock_commitment
  returned_cylinder_recording
  cash_collection
  payment_status
// all succeed or none do
```

**Windowed / rolling-window threshold** — threshold on a count or sum within a
resetting time window, not on a point value. Use for rate limits, rolling caps,
and alert thresholds. Distinguish the window duration from the threshold value.

```
if count(invalid_attempts, window=15min) >= 5  → lockout(15min)
if lockout_count >= 2
   OR count(lifetime_invalid_attempts) >= 10   → fallback required
```

## ID Guidance

IDs are useful when content needs traceability. Avoid IDs for disposable notes or
single-use prose.

Recommended prefixes, in Optional Labels order:

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
- `WF-` for workflows.
- `EVT-` for events.
- `INT-` for interfaces.
- `BND-` for boundary.
- `OWN-` for ownership.
- `ST-` for state.
- `INV-` for invariants.
- `FH-` for failure handling.
- `EVO-` for evolvability.
- `OBS-` for observability.
- `MIG-` for migration or rollout.
- `OUT-` for outcomes.
- `EDGE-` for edge cases.
- `R-` for risks.
- `EX-` for examples.
- `AC-` for acceptance criteria.
- `V-` for validation items.
- `NG-` for non-goals.
- `OQ-` for open questions.

If a needed prefix is not listed, define one using the same short uppercase
convention and use it consistently within the document.

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
