# Online Invoicing Service Checklist Spec

## Style

- Audience: implementation agents and reviewers.
- Tone: terse, actionable, review-friendly.
- Format: checklist with acceptance-oriented statements.

## Product Boundary

- [ ] Build for small service businesses.
- [ ] Support one business account owning customers, invoices, payments, and
  reminders.
- [ ] Optimize for correctness, reliability, and evolvability.
- [ ] Treat the release as a mid-stage MVP.

## Core Journey

- [ ] User can create a customer.
- [ ] User can create a draft invoice for a customer.
- [ ] User can add line items, tax, issue date, and due date.
- [ ] User can send the invoice or copy a hosted payment link.
- [ ] System can record payment status.
- [ ] User can see sent, paid, partially paid, failed, cancelled, and overdue
  invoices.
- [ ] User can manually trigger reminders for eligible overdue invoices.

## Business Rules

- [ ] Invoice numbers are unique per business account.
- [ ] Draft invoices are editable.
- [ ] Sent invoices lock issued financial details.
- [ ] Paid invoices are not directly editable.
- [ ] Failed payments do not reduce invoice balance.
- [ ] Settled payments reduce remaining balance.
- [ ] Overdue state is derived from due date and unpaid balance.
- [ ] Reminder triggers do not duplicate the same reminder for the same invoice.

## Data Requirements

- [ ] Store business account identifier.
- [ ] Store customer identity and contact details.
- [ ] Store invoice number, status, issue date, due date, currency, subtotal,
  tax, total, paid amount, and balance.
- [ ] Store invoice line item description, quantity, unit price, and amount.
- [ ] Store payment provider reference, amount, status, and event identifier.
- [ ] Store reminder trigger, timestamp, and delivery state.

## Reliability Requirements

- [ ] Payment events are idempotent.
- [ ] Invoice status can be recalculated from invoice and payment records.
- [ ] Duplicate payment events do not double-count payment.
- [ ] Failed delivery of invoice or reminder is visible and retryable.

## Evolvability Requirements

- [ ] Payment provider state is separate from invoice lifecycle state.
- [ ] Sent invoice details are snapshotted.
- [ ] Customer edits do not mutate sent invoice history.
- [ ] Model leaves room for recurring invoices, accounting sync, and more
  payment methods later.

## Out of Scope

- [ ] Multi-currency.
- [ ] Accounting sync.
- [ ] Subscription billing.
- [ ] Quote-to-invoice conversion.
- [ ] Inventory.
- [ ] Approval workflows.
- [ ] Enterprise permissions.

## Open Questions

- [ ] Is partial payment required for MVP?
- [ ] Are taxes free-form values or saved tax rates?
- [ ] Are reminders manual only at launch?
