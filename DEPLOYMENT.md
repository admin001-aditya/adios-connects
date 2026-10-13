# Staging-first deployment guide

## Important: preserve the existing production app

Your supplied brief names an existing Netlify site and a Supabase project. This source ZIP has not inspected or modified their current files/schema. Do **not** point the first deployment at the current production site or run migrations directly on a live database without a backup and schema review.

## 1. Prepare local environment

- Install Node.js 20+ and the Supabase CLI.
- Run `npm install`.
- Copy `.env.example` to `.env.local` and set the Supabase project URL and frontend-safe publishable/anon key only.
- Keep `.env.local` out of source control. The `.gitignore` already excludes `.env*` except `.env.example`.
- Run `npm test`, `npm run typecheck`, `npm run build`, and `python scripts/verify_project.py`.

## 2. Create an isolated staging project

1. Create/choose a staging Supabase project.
2. Back up any target database before making schema changes. If using an existing project, inspect tables, foreign keys, policies, triggers, Auth redirects, and storage buckets first. The migrations are designed as a complete schema and may need a reviewed schema-specific adaptation when tables from an older app already exist.
3. Link the CLI to the staging project and review the SQL migration plan. Apply `supabase/migrations/` only after the schema comparison.
4. Run `supabase/tests/paid_membership_invariants.sql` in the SQL Editor or with the project’s SQL test runner.
5. Deploy all functions listed in `README.md`. Confirm the built-in server-side Supabase environment values are present. Do not create a client-side service-role variable.
6. Configure Supabase Auth → URL Configuration with the staging site URL and `/auth/callback`, `/reset-password` redirect paths. Configure Google OAuth client ID/secret in Supabase Auth → Providers and allow the exact redirect URL in the Google OAuth console.
7. Enable/verify the email confirmation and recovery settings. For branded/reliable delivery, configure a real SMTP/email provider in Supabase. Without provider setup, do not claim email delivery was tested.
8. After creating and verifying the intended owner’s Auth account, review and run `supabase/manual/bootstrap_first_admin.sql`.
9. Upload the genuine QR, configure UPI details in the admin area, and test with staging payment records.
10. Run the manual checklist in `SECURITY.md`, including rejected/wrong amount/duplicate/replayed requests and cross-user RLS checks.

## 3. Expiry status maintenance

Paid API access checks `starts_at <= now()` and `ends_at > now()` regardless of the stored status, so an expired term cannot keep paid access because a scheduled job is late. The scheduled task only updates historical status and creates expiry notifications. If `pg_cron` is enabled on the project, an operator may install the optional job below using the SQL Editor:

```sql
select cron.schedule(
  'adios-connect-expire-subscriptions',
  '*/15 * * * *',
  $$select public.expire_due_subscriptions();$$
);
```

First verify that the `pg_cron` extension is enabled and that the function can run under the expected database role. If unavailable, use a secure scheduled server-side job to call the function with server credentials; never expose service credentials in the browser. Do not schedule duplicate jobs.

## 4. Netlify staging build

- Create a **new staging Netlify site** connected to this project/repository.
- Build command: `npm run build`.
- Publish directory: `dist`.
- Add only `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, and optional `VITE_SUPPORT_EMAIL` to the frontend environment. These are client-visible values; no service-role key is permitted.
- `netlify.toml` includes SPA fallback routing.
- Test login/OAuth, membership selection, checkout setup-required state, payment reference submission, admin verification, dashboards, support replies and mobile routes.

## 5. Production rollout and rollback

1. Export/backup the production database and retain the current deployment/commit.
2. Apply any reviewed schema adaptation to a staging clone first; snapshot old plan/payment rows before any paid-only data cleanup.
3. Verify migration output, RLS policies, Edge Function logs, Auth redirects, QR asset, owner recipient, and all acceptance checks.
4. Deploy the tested commit as a new Netlify deploy/site first. Do not instantly change DNS or overwrite the existing site.
5. Use a controlled rollout and keep the previous site/commit available. Roll back by restoring the previous Netlify deploy and, only if necessary, executing a reviewed database restore/migration rollback plan. Never delete customer accounts to remove an old plan catalogue entry.
6. After launch, monitor Edge Function errors, payment references, audit logs, notification delivery, expiring terms and support queue priority.

## Existing database migration note

`supabase/manual/paid_membership_legacy_cleanup.sql` is a **manual compatibility script**, not part of the standard automatic migration chain. It archives rows before removal and preserves Auth accounts/unrelated data, but it only recognizes direct `plan_code`, `code`, or `name` columns. An older schema using `plan_id` relations needs a schema-specific migration after inspection. Test this script only on a backup/staging clone; stop if any query errors or row counts look unexpected.
