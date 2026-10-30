# Berdahiya Wholesale Hub

B2B wholesale garment marketplace for India (Supabase backend).

## Status
Backend SQL for Steps 2-9 is written but NOT confirmed as fully applied
or tested. There is no production frontend in this repo yet; `prototype/`
is a visual mock-up only. Read `docs/PRODUCTION_READINESS.md` first.
Not production ready.

## Website (static)
`site/index.html` is a single-file site with real Supabase sign-in
(register, login, logout, password reset, account page). Upload it to
Cloudflare Pages. It still shows placeholder products. In Supabase set
Authentication > URL Configuration: Site URL and Redirect URLs to your
site address.

## Database
Migrations are in `supabase/migrations`, named in run order.
Run them one at a time (Supabase Dashboard > SQL Editor, new query tab
each) or with the Supabase CLI. After pasting, check the last line reads
"END OF ..." before pressing Run.

Then run, in order, from `supabase/checks`:
`step8_checks.sql`, `step9_checks.sql`, `production_audit.sql`.
They are read-only.

Before launch on production only, see the commented lines at the end of
the `hardening_fixes` migration (drop the dev payment simulation).

## Secrets
Never commit `.env`. The service-role key is server-only.
