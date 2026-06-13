# Online Invoicing Service: Business Domain Specification

## Style

- Audience: AI agents first, humans second.
- Tone: precise, bounded, implementation-ready.
- Format: stable headings, explicit facts, IDs, rules, and open questions.

## Document Metadata

- Domain: Online invoicing
- Product stage: Mid-stage MVP
- Primary users: Small service businesses
- Primary objective: Create, send, collect, and track invoices reliably
- Priority attributes: correctness, reliability, evolvability

## Facts

- F-001: A business account owns customers, invoices, and payment records.
- F-002: A customer can receive one or more invoices.
- F-003: An invoice contains one or more line items.
- F-004: A sent invoice can be paid through a hosted payment link.
- F-005: Invoice status is derived from invoice lifecycle and payment state.
- F-006: Overdue status is determined by due date and unpaid balance.

## Core User Journeys

- J-001: Create a customer.
- J-002: Draft an invoice for that customer.
- J-003: Add line items, taxes, due date, and payment instructions.
- J-004: Send the invoice by email or shareable payment link.
- J-005: Record payment success, partial payment, failure, or cancellation.
- J-006: Identify overdue invoices and trigger reminder workflows.

## Domain Entities

| Entity          | Description                                 | Owned By        |
|-----------------|---------------------------------------------|-----------------|
| BusinessAccount | Tenant boundary for invoicing data          | Platform        |
| Customer        | Buyer receiving invoices                    | BusinessAccount |
| Invoice         | Bill issued by a business to a customer     | BusinessAccount |
| InvoiceLineItem | Priced unit of work, product, or fee        | Invoice         |
| Payment         | Attempt or settlement against an invoice    | Invoice         |
| Reminder        | Communication triggered for unpaid invoices | Invoice         |

## Business Rules

- BR-001: Invoice numbers must be unique per business account.
- BR-002: Draft invoices may be edited freely.
- BR-003: Sent invoices must preserve invoice number, customer, line item,
  subtotal, tax, total, currency, issue date, and due date.
- BR-004: Paid invoices must not be edited directly.
- BR-005: A payment may fully or partially reduce invoice balance.
- BR-006: An invoice is overdue when it has unpaid balance after its due date.
- BR-007: Payment failures must not change the invoice balance.
- BR-008: Reminder eligibility depends on invoice status, due date, and unpaid
  balance.

## MVP Scope

In scope:

- Customer creation and lookup.
- Invoice draft, send, view, and status tracking.
- Line item pricing with subtotal, tax, and total calculation.
- Hosted payment link generation.
- Payment status ingestion.
- Overdue invoice listing.
- Manual reminder trigger.

Out of scope:

- Multi-currency invoicing.
- Accounting package sync.
- Subscription billing.
- Quote-to-invoice conversion.
- Inventory tracking.
- Approval workflows.
- Enterprise roles and permissions.

## Correctness Requirements

- CR-001: Invoice totals must be reproducible from stored line item, discount,
  tax, and currency values.
- CR-002: Status transitions must reject invalid lifecycle moves.
- CR-003: Payment events must be idempotent.
- CR-004: Reminder logic must not send duplicate reminders for the same trigger.

## Open Questions

- OQ-001: Are partial payments required for MVP?
- OQ-002: Should taxes be manually entered or selected from saved tax rates?
- OQ-003: Should reminders be manual only, or scheduled after MVP validation?
