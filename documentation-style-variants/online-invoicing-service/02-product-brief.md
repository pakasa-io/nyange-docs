# Online Invoicing Service Product Brief

## Style

- Audience: product, business, and implementation readers.
- Tone: plain, outcome-oriented, minimally technical.
- Format: narrative sections with clear scope boundaries.

## Product Intent

The online invoicing service helps small service businesses get paid without
managing invoices in spreadsheets, email threads, or disconnected payment tools.
The MVP should prove that a business can create a customer, issue a correct
invoice, send a payment link, receive payment status, and know which invoices
need follow-up.

The product should feel dependable before it feels broad. The first release
should prioritize reliable invoice state, correct totals, and a workflow that can
evolve without forcing a rewrite.

## Primary Users

The primary user is a small business operator who sells services, creates
invoices manually, and needs a simple way to track payment.

Examples:

- Consultant billing after completed work.
- Contractor issuing milestone invoices.
- Small agency billing monthly project work.

## MVP Journey

1. The user creates or selects a customer.
2. The user drafts an invoice with line items, taxes, issue date, and due date.
3. The user sends the invoice or copies a hosted payment link.
4. The customer opens the invoice and pays.
5. The business sees whether the invoice is draft, sent, paid, partially paid,
   failed, cancelled, or overdue.
6. The business follows up on unpaid overdue invoices.

## In Scope

- Customer records.
- Invoice drafts and sent invoices.
- Line items, subtotal, tax, total, issue date, and due date.
- Hosted payment links.
- Payment status updates.
- Overdue invoice list.
- Manual reminders.

## Out of Scope

- Multi-currency support.
- Accounting software sync.
- Subscriptions and recurring invoices.
- Quote workflows.
- Inventory.
- Approval flows.
- Enterprise permissions.

## Product Rules

- Every invoice belongs to one business account and one customer.
- Invoice numbers are unique inside a business account.
- Draft invoices can change.
- Sent invoices preserve issued financial details.
- Paid invoices are locked from direct edits.
- Failed payments do not reduce the amount due.
- Overdue invoices are unpaid invoices past their due date.

## Success Criteria

- A user can issue an invoice without manual calculation outside the product.
- Invoice totals remain consistent after sending.
- Payment status can be trusted.
- Overdue invoices are easy to find.
- The model can support future expansion without changing core invoice meaning.

## Open Decisions

- Whether partial payment is part of the first MVP.
- Whether tax is free-form or managed through saved tax rates.
- Whether reminders are manual only at launch.
