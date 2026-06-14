# Prompt: Remove Support Domain

## Task

Delete `business/support.md` and remove all traces of the support domain from
every other file in `business/`. The Customer Support Agent persona (P-07), the
support fallback boundary, and operational risk alerts are out of scope. They
must not appear in any canonical file — not as removed stubs, not as
cross-references, not as out-of-scope notes. Cut clean.

Where a capability previously required a Customer Support Agent actor, either
the capability is also out of scope and is removed, or it is re-attributed to
the next appropriate actor in scope (Super Admin or Outlet Manager). Make that
call on a case-by-case basis below.

## Read first

1. Read every file in `business/` completely.
2. Read `start-here/doc-style.md` for style rules.

---

## Delete `business/support.md`

Remove the file. Do not replace it with a stub or redirect.

---

## Update `business/README.md`

Remove the `support.md` row from the navigation table. Do not add a note or
placeholder.

---

## Update `business/identity-auth.md`

This file has the deepest exposure.

### P-07 Customer Support Agent persona

Remove the entire P-07 persona section.

### P-08 Area Manager

P-08 currently references operational risk alerts ("Monitors performance and
escalates exceptions across assigned outlets"). Remove that sentence. Keep all
other P-08 responsibilities that remain in scope.

### Access matrix

- Remove the `P-07 Support` column from the matrix entirely.
- Remove the `Operational risk alerts` capability row (it was P-06 scoped and
  P-10 full; with support removed, this monitoring feature is out of scope).
- Remove the `Order reassignment escalated` capability row if it has not already
  been removed by the order flow rewrite. If it is already gone, skip.
- Remove the `Customer notification requests` capability row if it was scoped to
  P-07; if the row still has meaningful entries for other personas, update it.

### Support Fallback Boundary section

Remove the entire "Support Fallback Boundary" section (the block covering
Customer Support Agent fallback scope, cross-outlet access rules, and the rule
that agents cannot approve or pay refunds through a support-owned workflow).

### Authorization Edge Cases

Remove **E-03** (cross-outlet support fallback requires explicit Super Admin
grant). It is predicated on the P-07 persona.

### Remaining "Customer Support Agent" references

Grep the file for every remaining occurrence of "Customer Support Agent",
"support fallback", "P-07", and "support agent". Remove or re-attribute each
one:

- If it describes a capability in scope, re-attribute to Super Admin.
- If it describes a capability that is itself out of scope, remove the sentence.

---

## Update `business/refund.md`

### Collection-code regeneration and reveal

P-07 appears in two places in the refund lifecycle: collection-code
regeneration/unlock and audited customer-verified reveal. In both cases,
re-attribute the capability to Super Admin only. Update the rule bodies to
name Super Admin where Customer Support Agent currently appears.

### Permissions table

Remove the P-07 column from the refund permissions table.

### Any remaining "Customer Support Agent" references

Grep and remove or re-attribute as above.

---

## Update `business/delivery.md`

### Operational risk alerts

Remove the sentence "Operational risk alerts may be shown to permissioned
Outlet Managers or Super Admins" or any equivalent at line ~181. Risk alerts
are out of scope entirely.

### Evidence retention and support review references

- Remove references to Customer Support Agent access to evidence files
  (line ~401: "Evidence files are unavailable through ordinary Customer Support
  Agent or...").
- Remove any step in delivery flows that names P-07 as an actor.
- Remove `support_review` from any state or lifecycle diagram that includes it
  (line ~388).

### Permissions table

Remove the P-07 column from the delivery permissions table.

### Any remaining "Customer Support Agent" references

Grep and remove or re-attribute as above.

---

## Update `business/notifications.md`

### Customer Support Agent notification rules

Remove:
- The rule that Customer Support Agent notification requests must use approved
  transactional channels.
- The rule that support-authored freeform message bodies are not supported.
- Any notification channel row or rule scoped to P-07.

### Operational risk alert channel rule

Remove the `Configured operational risk alert | Push only` row (or equivalent)
from the channel table. Risk alerts are out of scope.

### Boundary statement

Remove "support" from the list of domains that notifications does not own
(line ~14: "Notifications do not own order, payment, inventory, delivery,
refund, support..."). Rewrite the list without it.

### Any remaining "Customer Support Agent" or "support" references in a
domain-ownership sense

Grep and remove.

---

## Update `business/finance.md`

### Expense category additions

Line ~172: "Outlets may request category additions through Customer Support
Agent or Super Admin escalation." Remove "Customer Support Agent or" — escalation
path is Super Admin only.

### Any remaining "Customer Support Agent" references

Grep and remove or re-attribute to Super Admin.

---

## Update `business/catalog.md`

### Vendor additions

Line ~64: "Outlets may request vendor additions through Customer Support Agent
or Super Admin escalation." Remove "Customer Support Agent or" — escalation path
is Super Admin only.

### Any remaining "Customer Support Agent" references

Grep and remove or re-attribute to Super Admin.

---

## Scan remaining files

Grep `business/order.md`, `business/payment.md`, `business/inventory.md` for:
`Customer Support Agent`, `P-07`, `support fallback`, `support.md`,
`operational risk alert`, `risk alert`. Fix every match using the same rules:
remove if out of scope, re-attribute to Super Admin if the capability survives.

---

## Style

- Doc-style chunk shape and tone from `start-here/doc-style.md`.
- Remove BI-xx, F-xx, E-xx IDs that have no body after the rewrite (E-03
  specifically).
- Precise, concise, direct, neutral. No narrative filler.

---

## Verification checklist

Fail any item and fix before reporting done.

- [ ] `business/support.md` does not exist.
- [ ] `business/README.md` contains no link or reference to `support.md`.
- [ ] No canonical file contains "P-07" or "Customer Support Agent".
- [ ] No canonical file contains "support fallback" or "Support Fallback
      Boundary".
- [ ] No canonical file contains "operational risk alert" or "risk alert".
- [ ] No canonical file contains "E-03".
- [ ] The P-07 column does not appear in any permissions table in any file.
- [ ] The `Operational risk alerts` capability row does not appear in any
      permissions table.
- [ ] Collection-code regeneration in `refund.md` names Super Admin, not
      Customer Support Agent.
- [ ] Vendor and expense-category escalation in `catalog.md` and `finance.md`
      name Super Admin only.
- [ ] `delivery.md` contains no reference to `support_review`,
      "Customer Support Agent", or evidence access by support actors.
- [ ] `notifications.md` contains no support-domain channel rules or
      support-domain boundary claims.
