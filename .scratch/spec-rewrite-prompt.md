# Prompt: Rewrite Business Domain Specification by Aggregate

## Task

Rewrite `business/business-domain-specification.md` into a set of focused,
per-aggregate documentation files following the style guide at
`start-here/doc-style.md`.

The source spec is authoritative. Do not invent, remove, or alter any business
rules, invariants, lifecycle states, transitions, personas, permissions, or
decisions. Only restructure, reformat, and redistribute content.

## Read first

Before writing anything:

1. Read `business/business-domain-specification.md` completely.
2. Read `start-here/doc-style.md` completely.
3. Read `AGENTS.md` for operating scope and terminology.

## Output location

Write all new files to `business/`. Replace `business/business-domain-specification.md`
with the output set. Create a navigation index at `business/README.md`.

## Aggregate split

Create one focused document per aggregate. Each document covers the lifecycle,
business rules, invariants, personas, permission rows, and flows owned by that
aggregate.

| File | Aggregate | Primary source sections |
|---|---|---|
| `identity-auth.md` | Identity & Authorization | §2 Personas, §3 Access Matrix, §4 Auth Model, E-01–E-10 |
| `catalog.md` | Catalog & Pricing | §7.2 Refill Pricing Matrix, §7.3 Bundle Pricing, §7.4 Price Guardrails, §7.5 Express Delivery Fee |
| `order.md` | Order | §6.1 Order Lifecycle, §7.1 Outlet Allocation, §7.6 Cascade & Reassignment, §7.13 Mobile Money Order Expiry, §7.15 Cart Behaviour, F-01, F-02 |
| `payment.md` | Payment | §6.2 Payment Lifecycle, §7.10 Late Payment References |
| `delivery.md` | Delivery | §6.3 Delivery Lifecycle, §7.7 Agent Cash Handling, §7.12 Failed Delivery Fee Waiver, F-03, F-06, F-07 |
| `inventory.md` | Inventory | §6.4 Reservation Lifecycle, §6.5 Outlet Transfer Lifecycle, §6.8 Vendor Refill Batch Lifecycle, §6.9 Refill Exchange Request Lifecycle, §7.17 Stock Count Behaviour, §7.18 Low-Stock Alerts |
| `refund.md` | Refund | §6.6 Refund Lifecycle |
| `finance.md` | Finance | §7.8 Daily Closing, §7.9 Expense Controls, §7.19 Delivery Cost Reporting, BI-13 (settlements), BI-14 (receipt numbers), F-04, F-05 |
| `support.md` | Support | §6.7 Support Case Lifecycle, §7.16 Operational Risk Alerts |
| `notifications.md` | Notifications | §7.20 Notification Channel Boundaries |

**Split notes:**

- §6.9 Refill Exchange Request Lifecycle spans two aggregates. The field leg
  (PENDING through RETURN_RECORDED, doorstep recording rules) belongs in
  `delivery.md`. The intake leg (INTAKE_PENDING through COMPLETED/FAILED,
  outlet intake confirmation rules) belongs in `inventory.md`. Both files must
  cross-reference each other and the shared lifecycle diagram.
- F-04 Post-Payment Outlet Reassignment spans order, payment, and finance. Place
  it in `finance.md` (it is primarily a settlement flow) with cross-references
  from `order.md` and `payment.md`.
- §7.15 Cart Behaviour spans catalog and order. Place it in `order.md`; add a
  cross-reference from `catalog.md` for catalog-change and price-change cart
  effects.

## How to handle Business Invariants (§5, BI-01–BI-25)

Distribute each invariant inline to the aggregate it most tightly couples to.
Do not create a standalone invariants file. In each file, collect relevant
invariants under an `## Invariants` section. Add a cross-reference comment
where an invariant is shared across aggregates (e.g., BI-08 couples delivery
and inventory; place it in `delivery.md` with a note in `inventory.md`).

Ownership guidance:

| Invariant | Primary file |
|---|---|
| BI-01 (available stock never below zero) | `inventory.md` |
| BI-02 (price snapshot write-once) | `order.md` |
| BI-03 (payment reference unique per provider) | `payment.md` |
| BI-04 (financial records write-only) | `finance.md` |
| BI-05 (financial closure terminal) | `finance.md` |
| BI-06 (reserved stock unavailable) | `inventory.md` |
| BI-07 (COD no pre-payment gate) | `order.md` |
| BI-08 (delivery confirmation all-or-nothing) | `delivery.md` |
| BI-09 (no partial order completion) | `order.md` |
| BI-10 (cylinder sizes match on refill) | `delivery.md` |
| BI-11 (outlet stock isolated) | `inventory.md` |
| BI-12 (refund liability persists) | `refund.md` |
| BI-13 (settlements are accounting entries) | `finance.md` |
| BI-14 (receipt numbers permanent and sequential) | `finance.md` |
| BI-15 (audit records permanent) | `finance.md` |
| BI-16 (no self-approval) | `identity-auth.md` |
| BI-17 (outlet policy pre-checked before assignment) | `order.md` |
| BI-18 (collection code single-use and perishable) | `refund.md` |
| BI-19 (closure blocked until cylinders accounted for) | `delivery.md` |
| BI-20 (order totals immutable after placement) | `order.md` |
| BI-21 (every cancellation attributed) | `order.md` |
| BI-22 (outlet disable blocked by active orders) | `order.md` |
| BI-23 (critical mutations idempotent) | `order.md` |
| BI-24 (new cylinder purchases outright) | `catalog.md` |
| BI-25 (launch currency UGX only) | `finance.md` |

## How to handle Section 1 (Domain in One Page)

Make §1 the opening section of `business/README.md`. Follow it with the
navigation table linking to every aggregate file.

## Style guide requirements

Apply all of the following to every output file:

**Chunk shape**: Heading → Intent → Context → Main content →
Decisions/Constraints/Risks/Open Questions → Links. Short files may skip
later sections; dense files may add subsections.

**Labels**: Use Optional Labels from `start-here/doc-style.md` only where they
add clarity. Do not add labels to satisfy a template. Good candidates: `State`,
`Invariants`, `Business Rules`, `Permissions`, `Workflows`, `Failure Handling`,
`Boundary`, `Edge Cases`.

**IDs**: Preserve all existing IDs verbatim: BI-xx, F-xx, E-xx, P-xx, §x.x
cross-references. Assign new IDs using style-guide prefixes only to items that
will be referenced from other documents or need tracing.

**Sizing**: Apply the Document Sizing algorithm. If any aggregate file requires
skimming unrelated sections to complete one reader task, split it into sibling
files and add a local index heading. Lifecycle, permissions, flows, and
invariants for the same aggregate can share one file when they are all needed
together.

**State diagrams**: Reproduce all ASCII lifecycle diagrams exactly. Do not
reformat them.

**Permission rows**: The complete access matrix (all rows, all columns, all
scope notes) lives in `identity-auth.md` as the authoritative source. Each
aggregate file may include a trimmed table of the rows directly relevant to its
domain, with a note that the full matrix is in `identity-auth.md`.

**Tone**: Precise, concise, direct, neutral. No narrative filler, persuasion,
or marketing language. Prefer bullets, tables, and short paragraphs over long
prose blocks.

**"At launch" qualifiers**: Preserve every "at launch" qualifier verbatim. Do
not soften, move, or drop them.

**Threshold values**: Preserve every UGX amount, time window, percentage,
count limit, and retention period exactly as written.

## Permitted improvements

Only these structural improvements are permitted beyond redistribution:

- Convert long prose blocks into bullets or tables where meaning is preserved
  exactly.
- Add a one-sentence Intent statement near the top of each file.
- Add cross-reference links between files where content is split.
- Apply consistent heading levels across all output files.
- Remove duplicate preamble text that appears verbatim in multiple sections of
  the source.

Do not add new content, examples, interpretations, or implementation guidance.

## Verification checklist

After writing all files, verify:

- [ ] Every BI-xx appears in exactly one file under an Invariants heading.
- [ ] Every lifecycle from the source has its full state list, all state rules,
      and terminal state markers in the output.
- [ ] The full access matrix in `identity-auth.md` is complete: all rows, all
      persona columns, all scope notes from §3.
- [ ] §6.9 is present in both `delivery.md` (field leg) and `inventory.md`
      (intake leg) with cross-references.
- [ ] `business/README.md` links to every output file and opens with §1 content.
- [ ] No BI, F-xx, E-xx, or P-xx ID is missing from the output set.
- [ ] All "at launch" qualifiers from the source are present in the output.

Do not report the task as complete until this checklist passes.
