# Berdahiya Wholesale Hub: production readiness report

**Verdict: NOT ready for production.** Critical payment, testing and
refund items are incomplete (see section 2).

## How this review was done (and its limits)

- Claude had **no access** to your website code or your Supabase
  project during this work. Nothing could be run, built or opened in a
  browser, so **no end-to-end test was performed**.
- What was reviewed: the SQL migrations written for Steps 2 to 9, by
  reading them. What was observed: only screenshots you sent
  (Step 2, Step 3, Step 4 part 4 and Step 5 part 2 ran successfully;
  Step 9 part 2 failed and was fixed, but a re-run is not confirmed).
- Run `production_audit.sql` to get evidence from the real database.

## 1. Implemented in SQL, not yet tested

Auth roles and RLS, seller applications, products and images,
bulk pricing, inventory, marketplace search, cart, multi-seller
checkout, order statuses and history, notifications, 2% commission
in paise with order snapshots, audit log, admin functions, payout and
refund records, payment table and payment-verification gate.

## 2. Incomplete or missing (blockers marked)

- **BLOCKER: no frontend reviewed or tested.** Screens for most
  steps were never written; no journey (buyer, seller, admin) has
  been run.
- **BLOCKER: no payment gateway code.** No order creation with a
  provider, no webhook endpoint, so signature verification,
  duplicate-event handling in code, and test payments are untested.
  The database refuses to confirm an order without a verified
  captured payment and rejects duplicate provider payment ids.
- **BLOCKER: cancelling a paid order creates no refund.** The order
  is cancelled, stock returns and commission is reversed, but nothing
  records that money must go back to the buyer.
- **BLOCKER: migrations not confirmed.** Steps 6 to 9 were never
  confirmed as applied.
- Refunds cannot be started for completed orders, and sellers can
  request payouts the moment an order completes (no return window).
- Partial refunds do not adjust commission; only a full refund
  reverses it.
- Search computes ratings and sales for every match before paging;
  slow at large catalogues.
- Sellers rejected once cannot re-apply.
- No rate limiting on carts, reports or payout requests beyond
  Supabase defaults. No monitoring or backups configured.

## 3. Issues found in review

Fixed in files (apply by running them):

- Plain `CASE` inside a PL/pgSQL `IF` broke Step 9 part 2 (fixed).
- Any signed-in user could upload to the logo bucket
  (`hardening_fixes.sql`).
- Future functions would be callable by default
  (`hardening_fixes.sql`).

Not fixed (decisions or code needed): the blockers above, plus the
development payment simulation function must be dropped before launch.

## 4. Tests

- **Passed (observed):** Step 2, 3, Step 4 part 4 and Step 5 part 2
  scripts ran without error.
- **Failed and fixed:** Step 4 paste truncated (split into parts);
  Step 9 part 2 syntax error (fixed).
- **Not run:** all journeys, RLS tests with real accounts, concurrent
  order test, payment tests, build, mobile, console checks, and the
  read-only check files for Steps 8 and 9.

## 5. Journey test checklists (all currently NOT TESTED)

**Buyer:** register; verify email; log in; browse; search and filter;
add to cart; checkout; pay (test); see status; cancel unpaid order.

**Seller:** register; apply; wait for approval; upload product with
images; set stock; receive order after payment; accept, process,
ship; see commission and earnings; request payout.

**Admin:** log in; approve seller; approve product; view orders
(phone masked); set commission; record refund and payout; read audit
log.

**Must-pass security tests:** a buyer cannot read another buyer's
order; a seller cannot read another seller's order, edit another
seller's product, set their own status to approved, or change
commission; a pending seller cannot publish; nobody can set their own
role to admin; anonymous users cannot call admin functions; two
buyers ordering the last items at once cannot oversell.

## 6. Environment variables (names only)

Browser (public):
`VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`
(rename for your framework, for example `NEXT_PUBLIC_`).

Server only, never in browser code or Git:
`SUPABASE_SERVICE_ROLE_KEY`; for a gateway later:
`PAYMENT_KEY_ID`, `PAYMENT_KEY_SECRET`, `PAYMENT_WEBHOOK_SECRET`.

## 7. Deploying

1. Supabase: confirm email on, set Site URL and redirect URLs,
   configure SMTP, turn on password protection and CAPTCHA.
2. Take a backup, then run migrations in order: step2, step3,
   step4 parts 1 to 4, step5 parts 1 and 2, step6 parts 1 and 2,
   step7 parts 1 and 2, step8 parts 1 to 3, step9 parts 1 to 3,
   `hardening_fixes.sql`. Run the check files and
   `production_audit.sql`; fix every finding.
3. Create the first admin by SQL only (set `profiles.role` for your
   user in the SQL editor).
4. Check Dashboard > Advisors (Security and Performance).
5. Build the site, set environment variables on the host
   (Vercel, Netlify or similar), deploy, test on phones.
6. Backups: daily backups (paid plans) plus point-in-time recovery
   for orders and money tables; test a restore.
7. Monitoring: Supabase logs, an error tracker (for example Sentry),
   an uptime monitor, alerts for failed webhooks, and a daily
   report comparing payments, orders and commission.

## 8. Test payments to live payments (never automatic)

1. Build the webhook: verify the signature on the raw body, store the
   provider payment id once, ignore repeats, then call
   `apply_verified_payment` from trusted server code.
2. Test with test keys: success, failure, duplicate event, wrong
   amount, tampered signature, refund.
3. Remove `dev_simulate_payment` from production.
4. Review the audit log and ledger against test payments.
5. Put live keys only in production environment settings; keep test
   keys elsewhere. Register the live webhook secret.
6. Make one small real payment and one real refund, reconcile, then
   open to users. Switching is a manual decision by you.
