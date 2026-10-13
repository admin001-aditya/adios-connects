# AdiOS Connect — paid membership platform

AdiOS Connect is a React + TypeScript + Vite application with Supabase Auth/Postgres/RLS, privileged Supabase Edge Functions, and Netlify SPA routing. The membership catalogue is paid-only: **PRO** and **ADVANCED** are the only plan codes accepted by the database and payment-request API. A logged-in account has no paid entitlements until a matching manual payment request is approved by an authorized administrator.

> **Deployment status:** This ZIP is source code and deployment tooling. It has not been connected to, migrated into, or deployed over your existing production project. Real Google OAuth, email-delivery behavior, UPI recipient details, the genuine QR asset, and end-to-end payment approval require configuration and verification by the owner. No credentials or payment details have been invented.

## Functionality in this source tree

- Supabase email/password signup and login, email-verification callback handling, Google OAuth wiring, password-reset flow, session restoration, and logout.
- A premium responsive public site and membership selection screen with exactly two purchasable plan previews: Pro (proposed ₹199/month) and Advanced (proposed ₹499/month). When connected, the server catalogue is authoritative.
- Server-created payment requests with server-calculated plan prices and activation-term snapshots; UPI reference submission; idempotent authorized review with exact amount/reference checks, audit events, notification, and atomic subscription/entitlement activation.
- An effective-subscription helper and atomic calendar-month usage counters; expired/suspended/cancelled subscriptions do not pass paid endpoint checks.
- Profile display-name settings and authentication recovery links.
- Distinct Pro/Advanced dashboard presentation, real activity data, Advanced analytics sourced from real payment/support records, a support usage meter when entitlement data exists, and support-priority queue classification.
- Administrator control room for payment verification, plans/limits/modules/support priority, real QR recipient settings, membership lifecycle actions, support thread/status actions, public FAQ/announcement content, and audit-log review.
- PostgreSQL row-level security and server-only service-role operations for sensitive mutations.
- Privacy/terms/payment-policy drafts, Netlify redirects, migration files, security checklist, payment setup guide, and automated/static acceptance checks.

### Deliberately not represented as implemented

- No data-export feature is enabled (it is explicitly seeded as unavailable).
- No automatic recurring billing exists. Subscriptions do not auto-renew.
- No payment gateway is connected. The included payment flow is **manual UPI payment verification**; the admin must inspect the real payment record.
- No QR image or UPI ID is included in the ZIP. The owner adds the authentic QR and details.
- No outbound-email provider is configured. The app uses in-app notifications; configure an email provider separately before promising transactional email.
- No optional payment-screenshot upload is implemented in this version; the user submits a transaction reference/date and the reviewer uses the actual payment record.
- There is no deployment of your current website from here. Do not overwrite the existing Netlify site until the staging checklist passes.

## Requirements

- Node.js 20 or newer and npm.
- A Supabase project with Auth, PostgreSQL, and Edge Functions.
- Supabase CLI for applying migrations and deploying functions.
- A Netlify site for hosting the built SPA.
- A verified owner UPI destination and genuine QR image before checkout can open.

## Local development

1. Install dependencies: `npm install`.
2. Copy `.env.example` to `.env.local`.
3. Fill in `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` (publishable/anon key only). Keep all service-role keys out of `VITE_*`, browser code, and public assets.
4. Run `npm run dev`.
5. Run `npm test`, `npm run typecheck`, and `npm run build` before a release.
6. Run the independent source/entitlement consistency checks: `python scripts/verify_project.py`.

Without Supabase variables, public pricing is an **unconnected preview** and checkout remains disabled. Do not accept it as a live payment flow.

## Supabase deployment

Read [DEPLOYMENT.md](DEPLOYMENT.md), [SECURITY.md](SECURITY.md), and [PAYMENT_SETUP.md](PAYMENT_SETUP.md) before applying anything. Migrations are intended to be reviewed against a clean/staging database. Since a Supabase project URL was supplied for the existing app, **back up that database and inspect the existing schema first**—these files are not an instruction to blindly replace your production tables.

Recommended staging sequence:

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
supabase db push
supabase functions deploy get-checkout-config
supabase functions deploy create-payment-request
supabase functions deploy submit-payment-reference
supabase functions deploy review-payment
supabase functions deploy admin-update-plan
supabase functions deploy admin-update-payment-settings
supabase functions deploy create-support-ticket
supabase functions deploy member-insights
supabase functions deploy admin-set-subscription-status
supabase functions deploy admin-upsert-content
supabase functions deploy admin-reply-support-ticket
supabase functions deploy admin-update-support-ticket
supabase functions deploy mark-notification-read
```

The Supabase functions must have server-side Supabase URL/keys available in their environment. The service-role key is backend-only. Never paste it into this README, `.env.local`, Netlify public variables, or a screenshot.

## Initial administrator

Sign up and verify the intended owner email first. In the Supabase SQL Editor, review and run `supabase/manual/bootstrap_first_admin.sql`. It requires a verified Auth account and writes an audit record. It does not create a public administrator-registration endpoint. Change the email in that SQL file explicitly if ownership differs.

## Key security model

- The plan primary key has a database check constraint restricting plans to `PRO` and `ADVANCED`; both prices must be positive.
- The browser never supplies the authoritative amount, subscription status, actor identity, or entitlements. Edge functions derive the user from the verified Supabase bearer token.
- Sensitive writes use service-only, `SECURITY DEFINER` transaction functions that re-check administrator role and lock the request/subscription row.
- Entitlements are captured from the paid request at approval time; subscription rows retain their price/duration/plan version, modules, support priority, and individual feature limits. New catalogue edits do not silently rewrite those snapshots.
- RLS restricts customer payment requests, transactions, subscriptions, messages, notifications and usage records to the owner, with admin access verified separately.
- The payment review checks exact amount, transaction reference, an administrator attestation, current request status and the selected plan snapshot. This enforces a controlled workflow but cannot independently verify UPI settlement; the admin must look at the actual payment ledger/app.

See the [manual acceptance checklist](SECURITY.md) and `supabase/tests/paid_membership_invariants.sql`.
