# Online Invoicing Service Engineering Spec

## Style

- Audience: engineers and implementation agents.
- Tone: direct, constraint-heavy, correctness-oriented.
- Format: contracts, state, invariants, and failure handling.

## System Boundary

The invoicing domain manages customers, invoices, line items, payment records,
invoice status, and overdue tracking for a business account.

The domain does not own payment processing, email delivery infrastructure,
accounting sync, or identity. It consumes payment events and produces invoice
state.

## Core Model

```text
BusinessAccount
  Customer
    Invoice
      InvoiceLineItem[]
      Payment[]
      Reminder[]
```

## Invoice States

```text
draft -> sent -> paid
              -> partially_paid
              -> overdue
              -> cancelled
```

Rules:

- `draft` invoices are editable.
- `sent` invoices have issued financial details locked.
- `partially_paid` invoices retain unpaid balance.
- `paid` invoices are closed to direct edits.
- `overdue` is derived from due date and unpaid balance.
- `cancelled` invoices cannot accept new payments.

## Required Capabilities

- Create and update customer records.
- Create invoice drafts.
- Add, update, and remove draft line items.
- Calculate subtotal, tax, total, paid amount, and remaining balance.
- Send invoice and generate hosted payment link.
- Ingest payment events idempotently.
- List invoices by status, customer, issue date, due date, and overdue state.
- Trigger manual reminders for eligible unpaid invoices.

## Invariants

- Invoice number is unique per business account.
- Invoice total is reproducible from stored invoice components.
- Payment event processing is idempotent by external event identifier.
- Payment failure records are retained but do not reduce balance.
- A paid invoice cannot become unpaid without an explicit adjustment flow.
- Reminder triggers must not create duplicate sends for the same invoice and
  trigger type.

## Reliability Requirements

- Payment event ingestion must tolerate duplicate and out-of-order events.
- Invoice status must be recalculable from invoice and payment records.
- Sending an invoice must not depend on synchronous payment provider success
  beyond payment link creation.
- Failed reminder delivery must be visible and retryable.

## Evolvability Requirements

- Keep invoice lifecycle separate from payment provider state.
- Keep customer identity separate from invoice snapshots.
- Store issued invoice details so future customer edits do not rewrite history.
- Leave extension points for recurring invoices, accounting sync, and additional
  payment methods without changing invoice identity.

## Out of Scope

- Multi-currency.
- Subscription billing.
- Accounting sync.
- Quote conversion.
- Inventory.
- Approval workflows.
- Enterprise role management.

## Open Questions

- Should partial payment be supported in the first release?
- What is the canonical event source for payment settlement?
- Should reminder scheduling be deferred behind manual reminders?
