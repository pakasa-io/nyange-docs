# Notifications

**Intent**: Define launch notification channel boundaries and event-to-channel
assignments.

**Reader task**: Use this document to decide which channel a notification may
use and to confirm that notification delivery does not control business state.

**Source**: §7.20 Notification Channel Boundaries

## Boundary

- Notifications are communication side effects of domain events.
- Notifications do not own order, payment, inventory, delivery, refund, support,
  or finance state.
- Failed or unsent notifications may be surfaced for operational review where
  relevant, but they must not block or reverse committed business activity.

## Business Rules

- General business notifications use push and email at launch.
- SMS is reserved for customer authentication OTP only.
- WhatsApp is not launch behavior.
- Customer-configurable notification preferences are not launch behavior.
- Notification types are classified as transactional or non-transactional for
  future policy use.
- Customer Support Agent notification requests must use approved transactional
  notification templates or events with safe structured parameters.
- Support-authored freeform message bodies are not supported.

## Channel Assignments

Launch channels are assigned by event type, not customer preference.

| Event | Channels |
| --- | --- |
| Order confirmation | Push, Email |
| Ready for pickup | Push, Email |
| Out for delivery | Push only |
| Payment confirmation | Push, Email |
| Failed delivery | Push, Email |
| Outlet low-stock alert | Push only |

## Sensitive Content

### Refund Collection Codes

- Refund collection codes are not sent in push, email, or SMS message bodies.
- A notification may tell the customer that a refund is collectible.
- A notification may identify the collection outlet.
- A notification may direct the customer to the authenticated customer
  experience.
- A permissioned Customer Support Agent or Super Admin may perform audited code
  reveal only after customer verification.

## Non-Goals

- WhatsApp integration.
- Customer-configurable channel preferences.
- SMS for anything other than customer authentication OTP.
