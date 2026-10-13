# AdiOS Connect — Validation Report

Generated on 2026-10-13. This report separates checks completed in the generation environment from deployment-dependent validation that still must be run by the owner.

## Completed here

- `python scripts/verify_project.py`: **PASS — 41 dependency-free source/entitlement checks**.
- TypeScript parser pass over `src/`, `tests/`, and `supabase/functions/`: **PASS — 45 TypeScript/TSX files parsed, 0 syntax diagnostics**.
- Paid catalogue and migration source review: catalogue seed includes only `PRO` and `ADVANCED`; prices are positive; payment requests are server-priced; pending requests do not activate subscriptions; privileged review is role-checked and transactionally creates the subscription, copied entitlements, notifications, and audit entries.
- Archive contents are checked with `unzip -t` after creation.

## Not completed here

- `npm run build`: **BLOCKED** because `npm install` did not complete in this environment. The command reports missing React, React Router, Lucide, and React JSX-runtime packages. This is an environment/dependency-install blocker, not a successful build.
- `npm test`: **BLOCKED** because Vitest and the test dependencies are not installed (`vitest: not found`). The React test is included but has not passed in this environment.
- `supabase/tests/paid_membership_invariants.sql`: **NOT EXECUTED**; no connected staging Supabase database or local PostgreSQL CLI was available.
- Google OAuth, email verification/recovery delivery, actual UPI settlement, admin payment review, cron expiry notifications, responsive browser QA, and production deployment: **NOT END-TO-END TESTED**.

## Required next validation after installing dependencies

```bash
npm install
npm run test:static
npm test
npm run typecheck
npm run build
```

Then apply migrations to an isolated Supabase staging project, deploy the Edge Functions, run `supabase/tests/paid_membership_invariants.sql`, bootstrap the verified administrator using the manual SQL, configure the authentic UPI QR/recipient, and execute every pre-launch checklist in `SECURITY.md` before production release.
