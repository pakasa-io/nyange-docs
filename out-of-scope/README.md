# Out-of-Scope Tracker

Use this folder to document features, functionality, and proposals intentionally
excluded from the mid-stage MVP.

Each item should have its own Markdown file based on `TEMPLATE.md`.

## Rules

- Record the reason the item is out of scope.
- Separate confirmed decisions from open proposals.
- Include the revisit trigger or condition for reconsidering the item.
- Do not add out-of-scope items to MVP requirements unless they are explicitly
  moved back into scope.

## Naming

Use short, stable filenames:

```text
YYYY-MM-DD-short-item-name.md
```

## Index

| Item | Status |
| --- | --- |
| [Mobile-Money Payments](2026-06-13-mobile-money-payments.md) | Deferred |
| [Support Case Management](2026-06-13-support-case-management.md) | Deferred |

Revisit mobile-money payments when COD-only launch operations create a measured
adoption, cash-risk, reconciliation, or customer-convenience problem that cannot
be solved by cash-process controls within the MVP.

Revisit support case management when direct owning-domain workflows cannot
provide enough traceability for customer complaints, cross-domain exceptions,
SLA handling, or support workload management.
