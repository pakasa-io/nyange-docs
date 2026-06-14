# Prompt: Remove Operational Risk Alerts from Canonical Docs

## Task

Remove every trace of operational risk alerts from every file in `business/`.
This includes explicit mentions, deferral notes, back-links to the out-of-scope
tracker, and any capability row or boundary claim that exists solely because
risk alerts were once in scope.

The out-of-scope record at
`out-of-scope/2026-06-14-operational-risk-alerts.md` is the permanent home for
this content. It is not a canonical business document. Do not touch it. Do not
link to it from any canonical file. Canonical files must read as if operational
risk alerts were never part of the domain.

## Read first

1. Read every file in `business/` completely.
2. Read `start-here/doc-style.md` for style rules.

---

## Update `business/support.md`

### Non-Goals section

Remove the bullet that mentions operational risk alerts and links to the
out-of-scope tracker. The remaining Non-Goals content stays untouched.

### Verify nothing else in this file references risk alerts

Grep for "risk", "alert", "rolling", "threshold", "custody exception" in the
context of risk monitoring. Remove any match.

---

## Scan every other file in `business/`

Grep all files for: `risk.alert`, `risk alert`, `operational.risk`,
`rolling.window`, `rolling window`, `staff.risk`, `staff risk`, `§7\.16`,
`out-of-scope.*risk`, `risk.*out-of-scope`.

For each match:

- If the sentence or row exists solely because of operational risk alerts, remove
  it entirely.
- If the sentence belongs to a broader rule that incidentally mentions risk
  alerts (e.g. a notification channel table row, an access matrix row, a
  persona capability), remove only the risk-alert part and preserve the rest if
  it remains coherent. If removing it leaves the surrounding content incoherent
  or empty, remove the containing block too.
- Do not soften or qualify. Remove.

Known locations to check, based on prior document state:

**`business/identity-auth.md`** — The access matrix may contain an
`Operational risk alerts` capability row. Remove it. Remove any persona
description bullet that names operational risk alerts as a granted capability.

**`business/notifications.md`** — The channel table may contain a
`Configured operational risk alert` row. Remove it. Remove any channel rule or
boundary statement whose only subject is risk-alert notifications.

**`business/delivery.md`** — Delivery may reference risk alerts in the context
of delivery assignment (e.g. a note that risk-alert context is visible during
agent assignment). Remove it.

**All other files** — Treat every match as above.

---

## Style

- Doc-style chunk shape and tone from `start-here/doc-style.md`.
- If removing a bullet or row leaves a section with only one item, leave it
  as-is; do not collapse or merge sections unless the section heading itself
  becomes meaningless without the removed content.
- Precise, concise, direct, neutral. No narrative filler.

---

## Verification checklist

Fail any item and fix before reporting done.

- [ ] No file in `business/` contains "risk alert" or "risk alerts" in any
      form.
- [ ] No file in `business/` contains "operational risk" in any form.
- [ ] No file in `business/` contains "rolling window", "rolling-window",
      "staff-risk", or "staff risk" in any form.
- [ ] No file in `business/` links to `out-of-scope/2026-06-14-operational-risk-alerts.md`
      or any path containing "risk-alerts".
- [ ] No file in `business/` references `§7.16`.
- [ ] The `Operational risk alerts` capability row does not appear in any
      permissions table in any file.
- [ ] No persona description in `identity-auth.md` names risk alerts as a
      capability.
- [ ] No notification channel rule in `notifications.md` covers risk-alert
      events.
