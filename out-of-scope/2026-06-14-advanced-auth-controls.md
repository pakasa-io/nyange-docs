# Advanced Authentication Controls

## Status

- Status: Deferred
- Date Added: 2026-06-14
- Source: MVP authentication simplification decision on 2026-06-14
- Owner: Product Manager

## Summary

Advanced authentication controls are not part of the mid-stage MVP. MVP
authentication uses one Cognito user pool, one human app client, passwordless
SMS OTP for all human users, stateless JWT validation, and current Nyange
Identity facts for business authority.

## Proposed Scope

If moved back into scope, advanced authentication controls may include:

- password login;
- privileged MFA;
- step-up authentication for sensitive actions;
- staff or admin invite lifecycle;
- multiple Cognito user pools;
- multiple Cognito app clients for human users;
- custom OTP issuance, verification, retry, expiry, or lockout logic;
- application-owned server sessions;
- device management;
- Nyange-owned authentication telemetry warehouse;
- token-embedded business grants or outlet scopes.

## Captured Prior Business Rules

These details were removed from `business/` so MVP documentation keeps
authentication, authorization facts, and aggregate-owned authorization decisions
separate.

- Customer-only accounts authenticated with SMS OTP.
- Privileged accounts authenticated with username/password plus MFA.
- Privileged accounts could not use customer SMS OTP when privileged grants or
  scopes were active or pending activation.
- Users with privileged access had to complete password and MFA setup or a
  recovery path before exercising normal business permissions.
- Privileged account recovery, lost phone, password reset, and MFA reset were
  Super Admin-mediated.
- Super Admin recovery required two distinct active Super Admin actors.
- Credential readiness was modeled as a separate business access gate.
- Experiences were derived from permissions and session assurance.

## Why It Is Out of Scope

These controls add credential lifecycle, recovery, invitation, session,
provider-segmentation, and audit semantics before the MVP has proven the need
for them. They also conflict with the launch decision that all human users use
the same passwordless SMS OTP authentication path.

## Impact of Deferral

- All human users use passwordless SMS OTP through Cognito.
- Nyange does not store passwords, MFA factors, OTP state, refresh tokens,
  device fingerprints, or application-owned server sessions.
- Staff, delivery, finance, manager, and Super Admin authority is assigned by
  Super Admin through Identity grants and outlet scopes.
- Sensitive business operations rely on current Identity facts,
  aggregate-owned authorization rules, reason codes, and audit requirements.
- Account disablement, grant removal, and outlet-scope removal take effect on
  the next protected API request through current Identity fact checks.

## Revisit Trigger

Revisit when production risk, compliance requirements, account-takeover
incidents, staff onboarding volume, device trust requirements, or audit
investigations show that passwordless SMS OTP plus current Identity fact checks
is insufficient for privileged operations.

## Notes

- MVP Identity behavior is defined in
  [../business/identity-auth.md](../business/identity-auth.md).
- Aggregate-owned authorization behavior remains defined by each enforcing
  aggregate document.
