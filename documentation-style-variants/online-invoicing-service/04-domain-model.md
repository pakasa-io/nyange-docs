# Online Invoicing Service Domain Model

## Style

- Audience: domain modelers, product agents, and engineering agents.
- Tone: taxonomic, precise, terminology-first.
- Format: glossary, entity ownership, lifecycle, and rules.

## Domain Summary

An online invoicing service enables a business account to bill customers for
services, request payment, record payment outcomes, and identify invoices that
need follow-up.

The domain centers on invoice correctness: an invoice must preserve what was
issued, what is owed, what has been paid, and what action is next.

## Glossary

| Term | Meaning |
| --- | --- |
| Business Account | Tenant boundary for one invoicing business |
| Customer | Person or organization receiving invoices |
| Invoice | Issued request for payment |
| Draft Invoice | Editable invoice before issue |
| Sent Invoice | Issued invoice with locked financial details |
| Line Item | Billable service, product, or fee on an invoice |
| Payment Link | Hosted URL where the customer can pay |
| Payment | Recorded payment attempt or settlement |
| Balance | Invoice total minus settled payments |
| Overdue Invoice | Invoice with unpaid balance after due date |
| Reminder | Follow-up communication for an unpaid invoice |

## Entity Ownership

| Entity | Owner | Notes |
| --- | --- | --- |
| BusinessAccount | Platform | Owns tenant-level invoice numbering |
| Customer | BusinessAccount | Can appear on many invoices |
| Invoice | BusinessAccount | References customer and stores issued snapshot |
| InvoiceLineItem | Invoice | Used to calculate totals |
| Payment | Invoice | Reduces balance only when settled |
| Reminder | Invoice | Records follow-up intent and delivery state |

## Invoice Lifecycle

1. Draft: invoice is being prepared and can be edited.
2. Sent: invoice is issued and financial details are locked.
3. Partially Paid: invoice has at least one settled payment and remaining
   balance.
4. Paid: invoice balance is zero.
5. Overdue: invoice is unpaid after due date.
6. Cancelled: invoice is void and no longer payable.

## Domain Rules

- A business account controls its own invoice number sequence.
- A customer record can change, but sent invoices preserve issued customer
  details.
- A draft invoice can change line items, tax, due date, and customer.
- A sent invoice cannot change issued financial details.
- A payment event must be retained even when it fails.
- Only settled payments reduce balance.
- Overdue status is derived, not manually assigned.
- Reminder eligibility requires unpaid balance and a non-cancelled invoice.

## MVP Boundary

In the MVP, the domain includes customer management, invoice creation, invoice
sending, hosted payment links, payment status, overdue tracking, and manual
reminders.

The MVP excludes multi-currency, subscriptions, accounting sync, quotes,
inventory, approval workflows, and enterprise permissions.

## Model Risks

- If invoice details are not snapshotted at send time, later customer or line
  item edits can corrupt historical invoices.
- If payment events are not idempotent, duplicate provider events can overstate
  payment.
- If overdue status is stored as manual state, it can drift from due date and
  balance.

## Open Questions

- Are partial payments a required invoice state?
- Should taxes be represented as line-level or invoice-level values?
- Should reminders become scheduled automation after manual reminders prove
  useful?
