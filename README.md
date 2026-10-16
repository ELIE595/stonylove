# Stony Love — Lot 1 (secure foundation)

Next.js (App Router, TypeScript strict) + Tailwind v4 + Supabase (Auth, Postgres, RLS).
This release contains the secure base. Discovery, matches and chat screens are **not built yet** (their database rules and RPCs are, and are tested).

## What works today
- Sign-up (18+ check, email confirmation), sign-in, sign-out, protected routes (`/app`, `/admin`) enforced in `src/proxy.ts` and again in server components.
- Profile editing with a visibility switch (hidden by default; needs a 20+ character bio to become visible).
- Moderation queue at `/admin` for moderators/admins (resolve reports; decisions go to `audit_logs`).
- Light / dark / system theme (cookie, no flash), nonce-based CSP, security headers.
- Database: `supabase/migrations/20261016000001_core_schema.sql` (tables, RLS, column grants, `like_user`, `resolve_report`, `set_suspension`, `get_my_profile`).

## Setup
1. Install Node 22+ and run `npm ci`.
2. Create a Supabase project. In **SQL editor** (or `supabase db push` if you use the CLI) run the migration file.
3. `cp .env.example .env.local` and fill the three variables. Never put the `service_role` key in any `NEXT_PUBLIC_*` variable.
4. Supabase dashboard → Authentication: enable **Confirm email**; set Site URL to `NEXT_PUBLIC_SITE_URL`; add `<SITE_URL>/auth/callback` to Redirect URLs; configure SMTP and CAPTCHA/rate limits for production.
5. `npm run dev` → http://localhost:3000
6. Make a first admin by hand after signing up: `update public.profiles set role = 'admin' where id = '<user uuid>';` (run in the SQL editor; clients cannot do this).

## Scripts
`npm run lint` · `npm run typecheck` · `npm test` (all) · `npm run test:unit` · `npm run test:rls` · `npm run build` · `npm run audit`

## Security model (short)
- Clients get **column-level grants** only: they cannot read others' `birth_date`/`role`, cannot set `role`, `suspended_at`, `sender_id`, likes or matches directly.
- Likes/matches go through `like_user()` (advisory lock per pair, unique constraints, one match + one conversation).
- Messages: readable/writable only by the two matched members and only while neither has blocked the other; `client_id` makes retries idempotent; no edit/delete.
- Staff actions are SECURITY DEFINER functions that check the caller's role and write `audit_logs`.
- Messages are **not** end-to-end encrypted (stated on `/safety`).

## Not done / not verified (be aware)
- No screens yet for discovery, matches, chat, photos, stickers, notifications, blocking/report buttons, account deletion/export, legal pages.
- RLS tests run on in-memory Postgres (PGlite) with a Supabase shim, sequentially: **concurrency of `like_user` and Supabase Realtime are untested**. Run the migration on a real Supabase project and re-test.
- No Playwright E2E, Semgrep, Gitleaks, Trivy or OWASP ZAP runs yet; no external pentest. No in-app rate limiter (rely on Supabase Auth limits + CAPTCHA + your host's WAF).
- `style-src` allows `'unsafe-inline'` (React style attributes); scripts are nonce-only.
- `npm audit` reports 5 high issues in dev-only lint tooling (braces/micromatch via eslint-config-next); `npm audit --omit=dev` reports 0.
- System fonts are used (no Google Fonts fetch) so the build works offline.
