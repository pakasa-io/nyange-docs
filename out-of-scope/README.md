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
| [Express Delivery Fee](2026-06-14-express-delivery-fee.md) | Deferred |
| [Mobile-Money Payments](2026-06-13-mobile-money-payments.md) | Deferred |
| [Outlet-Local Pricing and Guardrails](2026-06-14-outlet-local-pricing-guardrails.md) | Deferred |
| [Competitive Outlet Claiming](2026-06-14-competitive-outlet-claiming.md) | Deferred |
| [Delivery Assignment Ranking](2026-06-14-delivery-assignment-ranking.md) | Deferred |
| [Post-Collection Price Adjustments](2026-06-14-post-collection-price-adjustments.md) | Deferred |

Revisit express delivery fee when the Product Manager approves a customer-facing
priority delivery offer with service-level promise, eligibility, capacity, fee,
refund or waiver, reporting, and authorization rules.

Revisit mobile-money payments when COD-only launch operations create a measured
adoption, cash-risk, reconciliation, or customer-convenience problem that cannot
be solved by cash-process controls within the MVP.

Revisit outlet-local pricing and guardrails when outlets need independently
managed local price schedules that cannot be handled through global online
pricing and Super Admin catalog administration.

Revisit competitive outlet claiming when order volume, service-area overlap, or
outlet capacity pressure makes static service-area assignment too manual or too
slow for daily operations.

Revisit delivery assignment ranking when manual dispatcher assignment causes
measured dispatch delays, uneven agent workload, or avoidable missed deliveries.

Revisit post-collection price adjustments when the Product Manager approves a
post-delivery adjustment policy with source owner, authorization, posting,
receipt, reporting, and Refund handoff rules.
