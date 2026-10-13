# Security model and release checklist

## Threat boundaries

Treat every browser field as untrusted. Auth identity comes from a verified Supabase bearer token (`auth.getUser()`); paid status comes from the active subscription row and its snapshot entitlements. The client cannot set a plan, amount, verified-payment flag, role, subscription end date, or usage counter. Authenticated UI visibility is not authorization: each paid Edge Function independently calls the effective-subscription/feature checks, while critical mutations call server-only database functions.

## Payment review gates

The payment request is created with a server-calculated amount and snapshots of the selected plan's terms. It remains `pending_payment` until the customer submits a reference and becomes `pending_verification`. Approval is rejected unless:

1. The authenticated caller still has an active administrator role in `administrator_roles`.
2. The request is locked and still awaiting verification, and is not already rejected/cancelled.
3. The submitted transaction reference exists and the reviewer confirms the matching reference.
4. The verified received amount exactly matches the order amount.
5. The reviewer explicitly attests to checking the actual transaction record and recipient.
6. The plan snapshot remains one of the two paid plan codes and has a valid amount/duration.
7. A unique `payment_request_id` constraint allows at most one subscription for the request.

The approval function inserts the subscription and its entitlements, marks the request approved, notifies the user and writes audit events in one database transaction. Duplicate retries return the already-created subscription rather than creating another. This is a human-reviewed flow—not an independent UPI settlement integration.

## Database and role rules

- RLS is enabled on all application tables; payments, subscriptions, support conversations, usage, notices and audit logs are scoped to the owner/admin.
- A profile has no editable plan or role field.
- Direct client writes to plan, payment, membership, entitlements, usage, administrator roles, and audit logs are revoked.
- The initial owner role is granted only by the manual verified-account bootstrap SQL.
- Plan edits go through server-side validation and audit logging. Purchases preserve agreed activation-time terms.
- The support reply and status Edge Functions verify admin authorization and use transactional audited RPCs. Direct client status updates are revoked.

## Required pre-launch acceptance checks

Run the source checks (`python scripts/verify_project.py`), JS test suite (`npm test`), typecheck (`npm run typecheck`), production build (`npm run build`), and SQL invariants script after applying migrations in staging. Then manually test against a non-production Supabase project:

- [ ] Public pricing contains exactly two cards, Pro and Advanced, with positive INR prices; no zero-price membership.
- [ ] API rejects any plan code other than PRO/ADVANCED, including lowercase/modified/browser-submitted values.
- [ ] Login alone creates only a profile; it does not create subscription/entitlements.
- [ ] Pending and rejected payment requests do not unlock paid endpoints.
- [ ] A client cannot update payment amount/status, plan catalogue, subscription, entitlement, role, usage counters, or audit logs.
- [ ] Changing local storage, query parameters, or a profile field cannot alter membership access.
- [ ] Pro-only entitlement is rejected for a member without the right snapshot; Advanced analytics is rejected for Pro.
- [ ] A valid approved Pro request activates Pro only; a valid approved Advanced request activates Advanced only.
- [ ] Reviewer identity/role is checked server-side; non-admin approval returns forbidden.
- [ ] A repeated approval is idempotent and there is only one subscription for a payment request.
- [ ] Repeated/duplicate transaction references cannot be reused for another payment request.
- [ ] Wrong verified amount/reference, missing attestation, or pending-payment (reference not submitted) cannot be approved.
- [ ] Expired, future-start, cancelled, and suspended terms cannot access paid Edge Functions.
- [ ] A rejected request does not disable a separate legitimate active term.
- [ ] Cross-user reads of payment refs, subscription data, support tickets/messages, notifications, usage records and profile data fail under RLS.
- [ ] Admin plan/payment setting/status mutations are rejected for a non-admin and produce audit logs for an admin.
- [ ] Support reply/status changes create the expected customer notification and audit record.
- [ ] Checkout stays blocked until the configured QR image loads and recipient details are configured.
- [ ] Google OAuth redirects, email verification, password reset, and production email delivery work with the actual Supabase/provider configuration.
- [ ] Run through responsive/keyboard/reduced-motion checks and review all legal copy with a qualified reviewer.

## Remaining real-world risks/requirements

- Payment evidence is manually verified, not queried from a bank/UPI API.
- Supabase service-role and OAuth settings are deployment secrets/configuration and must be reviewed in the actual project.
- Apply the expiry scheduled job described in `DEPLOYMENT.md`; the effective-subscription query already checks timestamps even if the status-cleanup job has not run.
- Privacy, terms, refund wording, support identity and applicable local legal/tax requirements must be checked by the owner before launch.
- This ZIP has not been penetration-tested, audited by an independent security firm, or deployed to production.
