# Notifications

**Intent**: Define launch notification channel boundaries and event-to-channel
assignments.

**Reader task**: Use this document to decide which channel a notification may
use and to confirm that notification delivery does not control business state.

**Source**: §7.20 Notification Channel Boundaries

## Boundary

- Notifications are communication side effects of domain events.
- Notifications do not own order, payment, inventory, delivery, refund, or
  finance state.
- Failed or unsent notifications may be surfaced for operational review where
  relevant, but they must not block or reverse committed business activity.

## Business Rules

- General business notifications use push at launch.
- SMS is reserved for human authentication OTP only.
- Email notification delivery is not launch behavior unless a future
  Identity/customer-profile rule defines email capture, verification, update,
  and consent semantics.
- WhatsApp is not launch behavior.
- Customer-configurable notification preferences are not launch behavior.
- Notification types are classified as transactional or non-transactional for
  future policy use.

## Channel Assignments

Launch channels are assigned by event type, not customer preference.

| Event | Classification | Channels |
| --- | --- | --- |
| Order confirmation | Transactional | Push |
| Claim blocked | Transactional | Push |
| Unclaimable closure | Transactional | Push |
| Ready for pickup | Transactional | Push |
| Out for delivery | Transactional | Push |
| Payment confirmation | Transactional | Push |
| Failed delivery | Transactional | Push |
| Refund collectible | Transactional | Push |

## Sensitive Content

### Refund Collection Codes

- Refund collection codes are not sent in push, SMS, or any future email message
  body.
- `Refund collectible` is sent when Refund moves a refund to `COLLECTIBLE`.
- The notification may tell the customer that a refund is collectible.
- The notification may identify the owning collection outlet.
- The notification may direct the customer to the authenticated customer
  experience.
- Failed or delayed `Refund collectible` notification fanout does not block the
  Refund state transition.

## Non-Goals

- WhatsApp integration.
- Email notification delivery.
- Customer-configurable channel preferences.
- SMS for anything other than human authentication OTP.

## Permissions

Notifications owns authorization decisions for Notifications-owned commands and
reads. Related rows are shown here for context; rows enforced by another
aggregate remain with that aggregate. Identity supplies actor, account, grant,
and outlet-scope facts from [identity-auth.md](identity-auth.md).

| Capability | P-06 | P-10 |
| --- | --- | --- |
| Notification template administration | - | Full |
| Customer notification requests | Scoped approved transactional only | Full |
