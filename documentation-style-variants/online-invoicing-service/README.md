# Online Invoicing Service Documentation Style Variants

These files describe the same hypothetical mid-stage MVP business domain:

An online invoicing service for small service businesses that need to create
customers, issue invoices, send payment links, accept payment, track invoice
state, and manage overdue invoices.

Use these variants to compare documentation style, tone, and format.

## Selected Standard

- Default:
  `01-ai-agent-contract.md`
- Fallback:
  `03-engineering-spec.md`

Use the AI-agent contract style as the de facto documentation style. Use the
engineering spec style only when the document needs implementation contracts,
state, invariants, failure handling, or reliability detail.

## Variants

- `01-ai-agent-contract.md`: structured, deterministic, optimized for AI agents.
- `02-product-brief.md`: concise product narrative for mixed product and business
  readers.
- `03-engineering-spec.md`: implementation-oriented specification with rules,
  invariants, and reliability concerns.
- `04-domain-model.md`: domain-driven format centered on terms, entities, and
  lifecycle.
- `05-checklist-spec.md`: terse checklist format for execution and review.

## Shared Scope

All variants use the same domain assumptions:

- Users are small service businesses.
- The MVP supports customers, invoices, line items, payment links, payment
  status, and overdue tracking.
- Invoice numbering is unique per business account.
- Draft invoices are editable.
- Sent invoices preserve issued financial details.
- Paid invoices are immutable except for explicit adjustment flows.
- Multi-currency, accounting sync, subscriptions, approvals, inventory, and
  enterprise roles are out of scope.
