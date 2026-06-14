# Prompt: Remove Launch Approval Governance

## Task

Remove the launch approval governance concept from every canonical file in
`business/`. This concept covers the formal Product Manager sign-off gate,
launch-scope acceptance authority, launch-blocking authority, and any rule that
conditions a business capability on receiving "explicit launch approval" from a
named approver.

Two locations carry this concept. Remove both. Every other file should be
confirmed clean.

Do not remove general approval workflows (expense approval thresholds, inventory
adjustment approval, refund approval, BI-16 no-self-approval). Those are
operational separation-of-duties rules and are unrelated. Do not remove
"launch scope" qualifier language ("not in launch scope", "unless explicitly
added to launch scope", "explicit launch-scope entry with a named source
owner") — those are scope boundary statements, not approval process rules.
Do not remove `order.md` references to "launch-blocking policy gap" — that
is a configuration requirement, not a governance approval step.

## Read first

1. Read every file in `business/` completely.
2. Read `start-here/doc-style.md` for style rules.

---

## Update `business/identity-auth.md`

### Launch Governance section

Remove the entire `## Launch Governance` section (the block beginning
"Launch-scope acceptance, dependency-gate completion, and launch approval are
business-governance decisions..." through to the end of the section, ending
before `## Authorization Edge Cases`).

Do not replace it with a stub or note. The section simply does not exist.

### Verify nothing else in this file references the PM approval gate

Grep for: `Product Manager.*approv`, `launch.approval`, `launch.blocking
authority`, `dependency.gate`, `operationally usable`, `production.ready`,
`production-ready`, `scripted.*sign.off`, `launch system`, `launch timing`,
`formal approval authority`, `launch-blocking authority`. Remove any match that
is part of the PM gate concept.

---

## Update `business/catalog.md`

### Regulator accessories

Line ~192: "Regulator accessories require explicit launch approval before being
treated as sellable."

Remove the approval-gate framing entirely. Replace with a plain scope fact:
regulator accessories are not sellable at launch. The sentence must state a
permanent boundary, not a condition pending approval. Suggested rewrite:

> Regulator accessories are not sellable at launch.

Keep all surrounding content in the Launch Accessories section unchanged.

---

## Scan every other file in `business/`

Grep all remaining files for: `launch approval`, `launch.approval`,
`launch-approval`, `Product Manager.*approv`, `approv.*Product Manager`,
`formal approval authority`, `launch-blocking authority`,
`dependency.gate`, `operationally usable in isolation`, `complete launch
system`, `production-ready`, `scripted.*sign.off`, `Launch Governance`.

For each match: remove if it is part of the PM gate concept. Leave everything
else untouched.

---

## Style

- Doc-style chunk shape and tone from `start-here/doc-style.md`.
- If removing the `## Launch Governance` section leaves a gap between two
  sections, close it cleanly — no blank heading, no placeholder.
- Precise, concise, direct, neutral. No narrative filler.

---

## Verification checklist

Fail any item and fix before reporting done.

- [ ] `## Launch Governance` does not appear in any canonical file.
- [ ] No canonical file contains "launch approval" or "launch-approval" in any
      form.
- [ ] No canonical file contains "formal approval authority" in the context of
      a PM gate.
- [ ] No canonical file contains "launch-blocking authority".
- [ ] No canonical file contains "dependency-gate" or "dependency gate".
- [ ] No canonical file contains "operationally usable in isolation".
- [ ] No canonical file contains "complete launch system" or
      "production-ready" in the launch governance sense.
- [ ] `catalog.md` states regulator accessories are not sellable at launch as
      a plain fact, with no approval condition.
- [ ] BI-16, expense approval thresholds, inventory adjustment approval,
      refund approval workflows, and `order.md`'s "launch-blocking policy gap"
      are all untouched.
