# Notifications

**Intent**: Define the notification channel boundaries and event-to-channel assignments for the Nyange platform at launch.

**Source**: §7.20 Notification Channel Boundaries

---

## Business Rules

- General business notifications use push and email at launch.
- SMS is reserved for customer authentication OTP only.
- WhatsApp and customer-configurable notification preferences are not launch behavior.
- Notification types are classified as transactional or non-transactional for future policy use.
- Notifications are best-effort and must not block or reverse committed order, payment, inventory, delivery, refund, support, or financial-ledger activity.
- Failed or unsent notifications are surfaced for operational review where relevant.

## Channel Assignments

Launch notification channels are assigned by event type, not customer preference:

| Event | Channels |
|---|---|
| Order confirmation | Push, Email |
| Delivery PIN | Push, Email |
| Payment confirmation | Push, Email |
| Failed delivery | Push, Email |
| Reservation-expiry warning | Push, Email |
| Dispatch notification | Push only |
| Exhausted-candidate Super Admin intervention alert | Push only |
| Outlet low-stock alert | Push only |
| Configured operational risk alert | Push only |

## Refund Collection Codes

Refund collection codes are **not** sent in push, email, or SMS message bodies.

Notifications may:
- Tell the customer a refund is collectible.
- Identify the collection outlet.
- Direct the customer to the authenticated customer experience or to a permissioned Customer Support Agent or Super Admin for audited reveal after customer verification.

## Delivery PIN

Delivery PIN notifications use push and email at launch and are governed by the PIN exposure and fallback rules in [delivery.md](delivery.md) (§6.3).

## Support-Requested Notifications

Support-requested customer notifications must use approved transactional notification templates or events with safe structured parameters. Support-authored freeform message bodies are not supported. The same launch notification boundaries apply.

## Non-Goals

- WhatsApp integration (not launch behavior).
- Customer-configurable channel preferences (not launch behavior).
- SMS for anything other than customer authentication OTP.
