# Rentline — Design Specification

*Status: draft for review, 2026-11-08. Read with [01-prd.md](01-prd.md) and
[03-schema.md](03-schema.md).*

Items marked **(call)** were picked as sensible defaults rather than decided by the owner. They
can be changed before the build starts.

## 1. Architecture

```
 Browser ── Next.js web app (Vercel)
               │  sign-in only: supabase-js → Supabase Auth  → gets a JWT
               │  all data:     fetch → Express API, "Authorization: Bearer <JWT>"
               ▼
          Express API (Render) ──────────────► Supabase Postgres  (schema "app", RLS on)
               │   ▲                          Supabase Storage   (private bucket)
               │   │ webhook                  Supabase Auth      (JWT verification)
               ├──►Stripe Checkout (test mode, CAD)
               └──►Resend (email)

 GitHub Actions ── every push: lint, typecheck, tests (against a local Supabase)
                └─ hourly: POST /internal/jobs/run  (scheduled job)
```

The web app never reads app data from Supabase directly. Everything goes through the API, so the
backend stays the clear centre of the project.

## 2. Repository layout

One Git repo with two npm packages:

```
rentline/
  api/                 Express + TypeScript
    src/
      routes/          HTTP only: parse, validate (Zod), call a service, shape the response
      services/        business rules (leases, invoices, payments, ledger, late fees, maintenance, email)
      db/              connection, transaction helpers (asUser / asSystem), queries
      jobs/            the scheduled job steps
      integrations/    stripe.ts, resend.ts, storage.ts
      openapi.ts       OpenAPI document built from the Zod schemas
    test/
    scripts/seed.ts
  web/                 Next.js (App Router) + TypeScript + Tailwind + shadcn/ui
  supabase/
    migrations/        plain SQL migrations: the source of truth for the schema and RLS
    config.toml
  .github/workflows/   ci.yml, jobs.yml
  README.md
  docs/
```

## 3. Stack

| Concern | Choice |
|---|---|
| Runtime | Node.js 22 LTS, TypeScript (strict) |
| HTTP | Express 5 |
| Validation | Zod; OpenAPI built from the same schemas (`@asteasolutions/zod-to-openapi`), served with Swagger UI at `/docs` **(call)** |
| Database access | `postgres` (postgres.js) with plain SQL **(call: easiest to explain line by line; no ORM magic)** |
| Database, Auth, Storage | Supabase (local via Supabase CLI; hosted free tier for the demo) |
| Payments | Stripe Checkout, test mode, `cad`, card only |
| Email | Resend **(call)**; with no API key set, emails are logged instead of sent |
| Tests | Vitest + Supertest against a real local Supabase |
| Web | Next.js (App Router), Tailwind, **shadcn/ui**, lucide-react, next-themes, supabase-js for sign-in only (same base stack as the author's Ebb project); visual design in §13 |
| Hosting | API on Render, web on Vercel, Supabase hosted **(call)** |
| Timezone | All dates ("today", due dates, months) use `America/Toronto` **(call)** |

## 4. Authentication and authorisation

**Sign-up / sign-in** happens in the web app with supabase-js (email + password, email
confirmation on). At sign-up the web app passes `role` (`landlord` or `tenant`) and `full_name`
as user metadata. A database trigger on `auth.users` creates the matching `app.profiles` row.

**Every API request** carries the Supabase access token. Middleware:

1. Verifies the JWT with supabase-js `auth.getClaims(token)`, which checks the signature against
   the project's JWKS and handles both the newer asymmetric keys and the legacy shared secret,
   so local and hosted Supabase behave the same. This reuses the library instead of hand-rolling
   `jose`. It rejects with `401` if anything fails, and returns `503` if the keys can't be
   fetched.
2. Loads the profile (id, role, email) and attaches it to the request.
3. Role guards per route (`requireRole('landlord')`) reject with `403`.

**Database access has two modes, and this is the core security design:**

The API connects to Postgres as a privileged role. Every request runs inside a transaction, and
inside it the code says which mode each step uses:

- **User mode** (`asUser`): runs `set local role authenticated` and sets
  `request.jwt.claims` to the caller's verified claims. Postgres now applies RLS, so
  `auth.uid()` is the caller and queries can only see or change rows the policies allow.
- **System mode** (`asSystem`): `reset role`. RLS is bypassed. It's used only for money writes
  (invoices, ledger, payments), the Stripe webhook, scheduled jobs, the email outbox, and
  linking a tenant to their leases.

**Make the safe path the only easy path.** The `db` module exports no bare query function.
Routes get a request-scoped transaction that **starts in user mode**, and system mode is an
explicit `tx.asSystem(fn)` call that switches back to user mode when `fn` returns. Queries
can't run outside a transaction, because `set local` only lasts for one transaction and a stray
query on a pooled connection would run as the privileged role with no RLS. The job runner and
webhook get a separate `db.systemTx(fn)` that is never available inside route handlers.

Rule: **every read for a user, and every landlord/tenant edit of properties, units, leases and
maintenance, runs in user mode.** System mode is only entered after a user-mode step has proven
the caller is allowed (for example, a user-mode `select` of the invoice succeeded). The two modes
share one transaction, so a user action and its outbox email commit or roll back together.

**Second fence:** all app tables live in the `app` schema, which is not exposed through
Supabase's auto-generated REST API. Someone holding the public anon key can't reach these tables
from the browser at all, and RLS still protects them if that setting ever changes.

**Tenant linking:** a lease stores `tenant_email`. When a confirmed user with role `tenant` calls
`GET /v1/me`, system mode sets `tenant_id` on every lease where
`lower(tenant_email) = lower(<their verified email>)` and `tenant_id is null`. The `/me` call
happens on every web app load, so linking is automatic.

## 5. API

Base path `/v1`. JSON in and out. Money is always integer cents (`amount_cents`), and dates are
`YYYY-MM-DD` strings.

**Error shape:** `{ "error": { "code": "invoice_overpayment", "message": "...", "details": {...} } }`
with the right status (400 validation, 401, 403, 404, 409 conflict, 422 business rule, 500).
A row the caller can't see returns `404`, not `403`, so it doesn't leak that the row exists.

**One error mapper** (Express 5 passes errors from async handlers to it automatically, so no
wrapper library is needed). Every failure class has a named result:

| Source | Result |
|---|---|
| Zod validation | `400 validation_failed` with field details |
| Postgres `23P01` (exclusion: overlapping lease) | `409 lease_overlap` |
| Postgres `23505` (unique) | `409 conflict` with the constraint name mapped to a code |
| Postgres `23503` (foreign key: still in use) | `409 in_use` (for example, deleting a unit with a lease) |
| Postgres `23514` (check) | `422 invalid_value` (should be caught by Zod first; reaching here is logged as a bug) |
| Postgres `40001` / `40P01` (serialization / deadlock) | retried once, then `503 try_again` |
| Business rules in services | `422` with a specific code (`invoice_overpayment`, `invalid_amount`, …) |
| Stripe API error or timeout (10 s) | `502 payment_provider_unavailable`; the transaction rolls back |
| Storage upload error | `502 storage_unavailable`; uploaded files cleaned up |
| Anything else | `500 internal_error` with the request id; the stack is logged, never returned |

**List endpoints** take `limit` (default 50, max 100) and `offset`.

| Method & path | Who | What |
|---|---|---|
| `GET /health` | anyone | liveness |
| `GET /docs` | anyone | Swagger UI |
| `GET /v1/me` | any | profile; triggers tenant linking |
| `POST /v1/properties` · `GET /v1/properties` | landlord | create / list own |
| `GET` · `PATCH` · `DELETE /v1/properties/:id` | landlord | delete also deletes its units; refused (`409 in_use`) if any unit has a lease |
| `POST /v1/properties/:id/units` · `GET /v1/properties/:id/units` | landlord | |
| `PATCH` · `DELETE /v1/units/:id` | landlord | delete refused (`409`) if the unit has a lease |
| `POST /v1/leases` | landlord | creates the lease and queues the invitation email; overlap → `409`; `start_date` no more than 90 days in the past (§7) |
| `GET /v1/leases` | both | landlord: theirs; tenant: theirs |
| `GET /v1/leases/:id` | both | lease, its balance and its invoices |
| `PATCH /v1/leases/:id` | landlord | only `end_date`, `late_fee_cents`, `grace_days`, and `tenant_email` (only while no tenant is linked, which re-queues the invite); `end_date` can't move before the last invoiced month |
| `DELETE /v1/leases/:id` | landlord | only if it has no invoices |
| `POST /v1/leases/:id/resend-invite` | landlord | only while `tenant_id` is null |
| `GET /v1/leases/:id/ledger` | both | entries oldest first, with a running balance |
| `GET /v1/invoices` | both | filters: `lease_id`, `status` (`unpaid`/`partially_paid`/`paid`), `overdue=true` |
| `GET /v1/invoices/:id` | both | invoice, outstanding amount, its ledger rows and payments |
| `POST /v1/invoices/:id/adjustments` | landlord | body `{ amount_cents, reason }` → a credit on that invoice (§6) |
| `POST /v1/invoices/:id/checkout` | tenant | body `{ amount_cents }` → `{ payment_id, checkout_url }` |
| `GET /v1/payments` · `GET /v1/payments/:id` | both | the web app polls `/payments/:id` after returning from Stripe |
| `POST /v1/payments/:id/cancel` | tenant | expires a `pending` payment's Checkout Session (§18) |
| `POST /v1/maintenance` | tenant | `multipart/form-data`: `lease_id`, `title`, `description`, `photos[]` (0–3) |
| `GET /v1/maintenance` · `GET /v1/maintenance/:id` | both | the detail view includes signed photo URLs (valid 1 hour) |
| `PATCH /v1/maintenance/:id` | landlord | body `{ status }`; only forward moves (open → in_progress → resolved) |
| `POST /v1/webhooks/stripe` | Stripe | raw body, signature verified |
| `POST /internal/jobs/run` | GitHub Actions | header `X-Job-Secret`; optional `as_of` date, accepted only when `ALLOW_AS_OF=true` (dev/test/demo) |

**Landlord dashboard summary** (`GET /v1/dashboard`, landlord): total outstanding, number of
overdue invoices, open maintenance count, payments received this month, and `months` (charged
and collected per month for the last 12, for the chart in §13). It's one endpoint so the web
app's home page is one call.

## 6. Money rules

- All amounts are integer cents in CAD. There are no floats anywhere, including in the web app,
  which formats cents for display only.
- **Ledger sign convention:** charges (`rent_charge`, `late_fee`) are positive. Payments and
  adjustments (credits) are negative. Balance = `sum(amount_cents)`. A positive balance means the tenant owes money.
- Every ledger row belongs to exactly one invoice, so per-invoice and per-lease balances are both
  just sums.
- **Invoice status** isn't stored. It's worked out by the `app.invoice_balances` view:
  - `paid` when outstanding ≤ 0
  - `partially_paid` when some payment exists and outstanding > 0
  - `unpaid` when there are no payments
  - `is_overdue` = outstanding > 0 and today > due_date

  Working it out instead of storing it means it can never drift out of step with the ledger.
- **Corrections are credit adjustments (owner's decision).** History is never edited. To fix a
  wrong rent amount, waive a late fee, or give a goodwill credit, the landlord adds an
  `adjustment` row to an invoice: a negative amount with a required reason (5+ characters),
  recorded with who made it. Rules, checked in system mode while holding the invoice row lock
  (same lock as checkout, §8):
  - `100 ≤ amount_cents ≤ outstanding − pending payments`, so a credit can't push an invoice
    below zero, even against a checkout that's in progress
  - it can't leave 1–99 cents owing (the "no stranded cents" rule)
  - an invoice credited down to $0 shows as `paid`

  A wrong rent on a lease is fixed in two steps: credit the difference on each invoice
  already issued, then end that lease (`PATCH end_date`) and create a corrected one starting the
  next day.
- The ledger is append-only, enforced three ways: no update/delete RLS policies, privileges
  revoked, and a trigger that raises on `UPDATE`/`DELETE` even for the system role.

## 7. Invoice generation

Part of the scheduled job (§10). It's given an `as_of` date (normally today in Toronto).

For each lease, for every calendar month from the lease's start month through `as_of`'s month
that the lease covers (`start_date ≤ last day of month` and `end_date ≥ first day of month`),
**if no invoice exists for that month:**

- `period_month` = first day of the month
- `due_date` = the later of (that month's `due_day`, `start_date`)
- `amount_cents` = the lease's `rent_cents` (full month, no proration)
- In one system-mode transaction: insert the invoice, insert its `rent_charge` ledger row, and
  queue the "new invoice" email. The email is queued only if the lease has a linked tenant
  (otherwise the invitation covers it) and only for the current month's invoice, so backfilled
  months don't produce a burst of emails.

The unique constraint on `(lease_id, period_month)` makes this safe to run any number of times,
even two copies at once. Insert uses `on conflict do nothing`.

**Set-based, not a loop.** The months to invoice come from one `insert … select` over
`generate_series` joined to the active leases, with `on conflict do nothing returning`. The
ledger rows and outbox rows are inserted from that `returning` set. That's a fixed number of
queries per run however many leases exist. The same applies to late fees (§9).

Backfill: a lease created with a past start date gets invoices for the past months on the next
run. That's intended, and it's how the demo seed shows an overdue invoice. It's bounded:
`POST /v1/leases` rejects a `start_date` more than 90 days in the past, so one lease can create
at most 4 invoices at once.

## 8. Payments (Stripe)

**Starting a payment** (`POST /v1/invoices/:id/checkout`, tenant):

1. User mode: select the invoice (RLS proves it's theirs, otherwise `404`).
2. System mode, same transaction: `select … for update` on the invoice row. This queues
   concurrent checkouts for the same invoice one behind the other.
3. Work out `available = outstanding − sum(amount of this invoice's payments with status 'pending')`.
   Reject with `422 invoice_overpayment` if `amount_cents > available`. Reject with
   `422 invalid_amount` if `amount_cents < 100`, or if paying it would leave between 1 and 99
   cents owing. This "no stranded cents" rule, together with rent ≥ $1 and late fees being
   $0 or ≥ $1 (schema checks), means what's owed is always $0 or at least $1, so an invoice can
   always be paid off.
4. Insert a `payments` row: `pending`, the amount, tenant id.
5. Create a Stripe Checkout Session: mode `payment`, `cad`, one line item
   ("Rent — Unit 4B, December 2026"), `expires_at` = now + 30 min, `metadata.payment_id`,
   `client_reference_id = payment_id`, success/cancel URLs back to the web app, and the
   **Stripe idempotency key `checkout-<payment_id>`**.
6. Store the session id on the payment. Commit. Return the `checkout_url`.

If the Stripe call fails, the transaction rolls back and no pending payment is left behind.

*Trade-off, stated in the README:* the invoice row lock is held during the Stripe call
(usually well under a second, with a hard 10 s timeout). That blocks only other checkouts and
adjustments on **that one invoice**, and it keeps the overpayment check simple and correct. If
Stripe created a session but the response was lost, the transaction rolls back and the orphan
session is never shown to anyone. It expires on its own, and its webhook is ignored as an
unknown payment.

**Webhook** (`POST /v1/webhooks/stripe`, mounted with `express.raw` before the JSON parser):

1. Verify the `Stripe-Signature` with the webhook secret, or return `400`.
2. One system-mode transaction:
   - `insert into app.stripe_events (id, type) … on conflict do nothing`. If the event was
     already recorded, commit and return `200`. **This is the "exactly once" guarantee.**
   - `checkout.session.completed` with `payment_status = 'paid'`: lock the invoice row, mark the
     payment `succeeded` **whatever its current local status** (a payment the safety net or a
     cancel already marked `expired` is still money taken), store the payment intent id and
     `completed_at`, insert the `payment` ledger row (`−amount`), set `flagged_overpayment` if
     needed, and queue the receipt email.
   - `checkout.session.expired`: mark the payment `expired`. That frees its amount for a new
     checkout.
   - Any other event type: recorded and ignored.
3. Return `200`. On an unexpected error return `500` so Stripe retries. The event row rolls back
   with everything else, so the retry is processed fresh.

**Overpayment edge case:** step 3 of checkout prevents overpaying, but money that Stripe has
taken is always recorded, whatever happens. If a succeeded payment would push an invoice below
zero, it's still recorded and the payment gets `flagged_overpayment = true`, visible to the
landlord. Refunding is out of scope. This should be close to impossible but is handled honestly.

**Safety net:** the scheduled job marks `pending` payments older than 2 hours as `expired`, in
case an `expired` webhook was missed.

## 9. Late fees

Part of the scheduled job. For each invoice where:

- `as_of > due_date + grace_days`, and
- outstanding > 0, and
- the lease's `late_fee_cents > 0`, and
- no `late_fee` ledger row exists for it yet,

insert one `late_fee` ledger row for `late_fee_cents`. A partial unique index (one `late_fee`
per invoice) guarantees it, even when the job runs twice at once. A tenant who pays in full
after the fee was added still owes the fee. The invoice stays open until that's paid.

## 10. The scheduled job

`POST /internal/jobs/run`, called **hourly** by a GitHub Actions cron (it also wakes the free
Render instance). It's also runnable locally: `npm run job -- --as-of 2026-12-08`.

Steps, in order, each idempotent:

1. Generate missing invoices (§7)
2. Apply late fees (§9)
3. Expire stale pending payments (§8)
4. Send pending emails (§11)

Each step is a plain function taking `(db, asOf)`, so tests can run any date without touching
the clock. The response lists what each step did (counts).

## 11. Email (transactional outbox)

Actions never call Resend directly. They insert a row into `app.email_outbox` **in the same
transaction** as the change they're about. If the change rolls back, no email is queued, and a
committed change always has its email queued.

- Every outbox row has a unique `dedupe_key` (for example `invoice_created:<invoice_id>`), so
  re-running an action can't queue the same email twice.
- After a request commits, the API tries to send its queued rows straight away (without
  making the response wait). Anything that fails is retried by the job, up to 5 attempts, and
  then marked `failed`.
- **Two senders can't send the same row.** The immediate send and the job may overlap, so a
  sender first *claims* rows:
  `update … set status = 'sending', claimed_at = now(), attempts = attempts + 1 where id in
  (select id … where status = 'pending' … for update skip locked limit 50) returning *`.
  On success the row becomes `sent`; on error it goes back to `pending` (or `failed` at 5
  attempts). Rows stuck in `sending` for more than 10 minutes (a crashed sender) are put back
  to `pending` by the job.
- As a second guard, each send passes the row's `dedupe_key` as Resend's `Idempotency-Key`
  header, a native feature, so even a retry after a lost response doesn't send twice.
- Templates (plain HTML, kept simple): `lease_invitation`, `invoice_created`,
  `payment_receipt`, `maintenance_opened`.
- **Resend limit:** without a verified domain, Resend only delivers to the account owner's own
  address. For the demo that's fine. The README says so, and with no API key emails are logged.

## 12. Maintenance photos

- Up to 3 photos per request, max 5 MB each, `image/jpeg`, `image/png` or `image/webp`, checked
  by file signature, not only by the declared type. Handled with `multer` in memory.
- Stored in a **private** Supabase Storage bucket `maintenance-photos` at
  `<request_id>/<uuid>.<ext>`, uploaded by the API with the service key.
- Order: the API generates the request's `id` first, uploads the photos under that id, then
  inserts the request and its photo rows in one user-mode transaction. If any upload or the
  insert fails, nothing is saved and photos already uploaded in that attempt are deleted (best
  effort; an orphaned file is harmless because nothing points to it).
- Requests can only be opened on a lease that is current today (enforced by the RLS insert
  policy).
- Viewing: after a user-mode read of the request succeeds, the API returns 1-hour signed URLs.

## 13. Web app

It uses supabase-js for sign-in and sign-up only, and calls the API for everything else.

### Visual design: "G · shadcn/ui" (owner's choice, 2026-11-09)

The approved drafts are on the "G · shadcn/ui (chosen)" page of the design canvas (a private
claude.ai artifact; screenshots go in `docs/ui/` when the web app is built). The look is
standard shadcn/ui, so the drafts map one-to-one onto real components:

| Concern | Choice |
|---|---|
| Components | shadcn/ui (Radix primitives, copied into `web/components/ui`): Sidebar, Card, Badge, Tabs, Table, Dialog, Sheet, Alert, RadioGroup, Input, Select, Textarea, Checkbox, Button |
| Styling | Tailwind CSS with shadcn's CSS-variable theme; **zinc** neutrals; primary = zinc-900 (white in dark mode) |
| Type | **Geist** for text, **Geist Mono** for references and keyboard hints; money uses tabular figures |
| Icons | lucide-react (stroke icons, `aria-hidden` when decorative, `aria-label` on icon-only buttons) |
| Theme | Light by default, dark via `next-themes` (`class` strategy); every screen must work in both |
| Status colours | Never colour alone: a dot **and** a word (Paid, Partly paid, Overdue · 39d, Due Nov 15) |
| Layout | Landlord: inset sidebar layout. Tenant: top nav, max-width 1040px. Responsive down to 375px; wide tables scroll inside their card |
| Voice | Plain words that say what happened and what to do: "October rent is 39 days late", never "Delinquency detected" |

### Screens (route → drafted screen)

**Public**
- `/login`: split layout; email and password, plus one-click **demo account** buttons and a
  "resets nightly" note (the live demo is login-only, §17)
- `/signup`: landlord / tenant **role picker** cards; the tenant hint explains invite-email
  linking. Hidden in production (`NEXT_PUBLIC_SIGNUP_ENABLED=false`)
- `/check-email`: confirm-your-address message

**Landlord** (sidebar: Overview, Rent roll, Properties, Payments, Maintenance)
- `/landlord`: four summary cards (collected this month, overdue, collection rate this year,
  open maintenance), a **charged vs collected bar chart** (6 months / this year toggle), and the
  rent-roll table with All / Overdue / Due soon / Paid tabs, a filter box and row actions
- `/landlord/properties`: one card per property with totals and a units table (tenant, lease,
  this month's status); "invite sent, not signed up yet" shown for unlinked tenants
- `/landlord/leases/new`: form grouped as Unit and tenant / Rent / Dates, field-level errors
  (an overlapping lease shows on the start date and in an alert), and a "What happens next"
  panel
- `/landlord/leases/[id]`: balance, rent, late fee and next invoice cards; Ledger / Invoices /
  Payments / Details tabs; the ledger shows type badges and an **"Added by"** column
  (Automatic, Stripe, or the landlord's name). **Add credit** opens a Sheet with invoice,
  amount, required reason and a before/after summary
- `/landlord/maintenance`: list and detail side by side (Open / In progress / Resolved tabs),
  a three-step status bar, photos via signed URLs, activity history, and one forward-only
  status button

**Tenant** (top nav: Home, Invoices, Payments, Maintenance)
- `/tenant`: an overdue Alert with **Pay now**, balance / next rent / paid this year cards,
  recent payments, and the open maintenance request
- `/tenant/invoices`: the invoice list; **Pay** opens a Dialog with Full balance (pre-selected) or
  Custom amount, the "can't leave less than $1" rule, and "remaining after payment"
- `/tenant/payments/[id]` (Stripe return): "Confirming your payment" while `pending` (polls
  every 2 s), then "Payment received"; a cancelled checkout returns to the invoice with the
  amount freed (§18)
- `/tenant/maintenance/new`: title, details and up to 3 photos with type and size errors shown
  per file; `/tenant/maintenance` and `/tenant/maintenance/[id]` list and show requests

**Data the overview needs:** `GET /v1/dashboard` also returns `months`: charged and collected
cents for each of the last 12 months (one grouped query over the ledger), which feeds the
chart and the collection-rate card.

## 14. Testing

Vitest + Supertest against a local Supabase (`supabase start`), with the database reset between
test files. Required coverage:

- **RLS isolation:** a tenant can't read another tenant's leases, invoices, ledger, payments or
  maintenance requests, and a landlord can't read another landlord's data. These are tested both
  through the API and with direct SQL in user mode.
- **Invoices:** running the job twice gives one invoice per month; backfill works; `due_date`
  respects `start_date`; there are no invoices outside the lease dates.
- **Payments:** an overpaying checkout is rejected; pending payments reduce what's available; a
  duplicate webhook event is applied once; `expired` frees the amount; partial then full payment
  ends as `paid`.
- **Late fees:** applied after the grace period, only once, and not when the invoice is paid.
- **Adjustments:** can't exceed outstanding minus pending payments, can't leave 1–99 cents, need
  a reason, are landlord-only, and are rejected for another landlord's invoice (`404`); a full
  credit makes the invoice `paid`.
- **Ledger:** `UPDATE`, `DELETE` and `TRUNCATE` are refused even in system mode.
- **The two database modes (highest risk):** a test that, for every `GET` route with an `:id`,
  requests another user's id and expects `404`. That catches a route that accidentally reads in
  system mode. A unit test checks that the transaction is back in user mode after `asSystem`
  returns *and* after it throws.
- **Concurrency:** two checkouts started at the same moment, each for the full outstanding
  amount, give exactly one `201` and one `422 invoice_overpayment`. The same is tested for
  checkout racing an adjustment, and for two outbox senders claiming at once (each row is sent
  once).
- **Grants:** in user mode, a landlord inserting a lease with `tenant_id` set fails, and so
  does updating `tenant_id`.
- **Leases:** overlapping leases on a unit are rejected (`409`).
- **Outbox:** a rolled-back action leaves no email; the dedupe key holds.

Stripe is never called in tests. The Stripe client is behind a small interface with a fake, and
webhook payloads are signed with `stripe.webhooks.generateTestHeaderString`.

## 15. CI/CD

- `ci.yml` on every push and PR: install, lint, typecheck (api + web), start Supabase with the
  CLI, apply migrations, run tests.
- `jobs.yml`: hourly cron that calls the job endpoint on the deployed API.
- **Free tiers only (owner's decision).** Known costs of that, all stated in the README: the
  Render API sleeps when idle, so the first request takes about a minute; GitHub disables
  scheduled workflows after 60 days without repo activity; and Supabase pauses an inactive
  project after about a week. Regular commits keep everything awake, and the README says how to
  wake the demo if it has paused.
- Deploys: Render and Vercel deploy `main` automatically. Migrations are applied to the hosted
  Supabase with `supabase db push`, run by hand for now **(call)**.

## 16. Security checklist

- `helmet`; CORS limited to the web app's origin; `express-rate-limit` on the API
- JWT verified on every request; role guards; RLS in user mode; `app` schema not exposed
- Stripe webhook signature verified; job endpoint behind a secret; `as_of` disabled in production
- Service key, Stripe secret, webhook secret, Resend key and job secret only come from
  environment variables. `.env.example` is committed and `.env` never is.
- Uploaded files are checked by signature and size and stored privately

## 17. Demo data

**The live demo is login-only (call, from review).** Supabase's built-in email and Resend
without a verified domain can't email strangers, so in production the web app hides sign-up
(`NEXT_PUBLIC_SIGNUP_ENABLED=false`) and the login page shows the demo accounts. Sign-up,
email confirmation and the invitation flow all work locally, where Supabase's local mail
catcher (Mailpit) shows every email. The README explains this.

**Nightly reset (call, from review).** A GitHub Actions workflow (`demo-reset.yml`) rebuilds
the hosted demo every night: it drops and recreates the `app` schema by re-running the
migrations, deletes the demo auth users through the admin API, and runs the seed. This is a
whole-database rebuild by an admin workflow, not an edit to ledger rows, so the append-only
rule still holds for the running application. The README states this distinction. Seed dates
are **relative to today**, so the overdue invoice and late fee are always there after a reset.
A reviewer who leaves Checkout open holds that amount for at most 30 minutes, which is
acceptable.

`npm run seed` (local or hosted):

- One landlord and two tenants with fixed demo passwords shown in the README, created through
  the Supabase admin API with email already confirmed
- Two properties and four units
- Three leases: one current and fully paid, one with an overdue invoice and a late fee, and one
  invited tenant who hasn't signed up yet
- Past invoices produced by running the job steps with past `as_of` dates, so the data looks
  real
- One maintenance request with a photo

Seeded rows get backdated `created_at` values so the history looks real.

## 18. Smaller decisions (calls, from review)

- Lease `end_date` is **inclusive** (it matches the `'[]'` range in the overlap constraint).
- User-mode updates use **column-level grants** (see schema), so a landlord can't set
  `tenant_id` on a lease or rewrite a tenant's maintenance text, even through a bug.
- Webhooks for a payment id the database doesn't know get `200` and are ignored. Local and
  hosted share one Stripe test account, so this is normal.
- The "cancel" return URL from Checkout calls the API to expire that session straight away
  (`POST /v1/payments/:id/cancel`), which frees the amount without waiting 30 minutes.
- Swagger UI has an "Authorize" box. The web app's account menu has a "copy API token" item so
  a reviewer can paste their token there and try endpoints.

## 19. Left to the builder

Folder naming inside `api/src`, the exact SQL of each query, web component structure, styling
details, and logging format (keep it structured JSON with a request id).

---

## ENG REVIEW REPORT

*2026-11-08 · reviewed: 01-prd.md, 02-design.md, 03-schema.md · no code exists yet.*

### Architecture as understood
A Next.js web app signs in with Supabase Auth and calls an Express 5 API with the JWT. The API
runs every request in one Postgres transaction that starts in **user mode** (role
`authenticated`, so RLS applies). It switches to **system mode** only for money, jobs, the
webhook and the outbox. App tables live in a non-exposed `app` schema. Money is integer cents
in an append-only ledger, and invoice status is derived in a view, never stored. Stripe Checkout
plus a signature-checked, event-id-deduplicated webhook records payments exactly once. An hourly,
idempotent, set-based job creates invoices, applies late fees, expires stale payments and drains
a transactional email outbox. Sources of truth: the ledger for money, `auth.users` for identity,
Stripe for "was money taken".

### Issues found and resolved in this review (all now written into the docs)

| # | Area | Issue | Fix applied |
|---|---|---|---|
| 1 | Security | Table-level `insert` on `leases` let a landlord set `tenant_id` and attach any user to a lease, bypassing invitation and linking | Column-level insert grants on `leases` and `maintenance_requests` (schema) plus a test (§14) |
| 2 | Security / data flow | Nothing structural stopped a route from querying in system mode or outside a transaction (where `set local` doesn't apply on a pooled connection, so no RLS) | The `db` module has no bare query function; route transactions start in user mode; `asSystem` is scoped; `systemTx` isn't available to routes (§4). Highest-risk path now has a dedicated cross-user `404` sweep test (§14) |
| 3 | Data flow | `ledger_entries.lease_id` duplicated `invoices.lease_id` with nothing keeping them equal | Composite FK `(invoice_id, lease_id)` |
| 4 | Data flow | `maintenance_requests.landlord_id` was a copy checked only by an RLS policy | Composite FK `(lease_id, landlord_id)` |
| 5 | Edge case | Email outbox: the immediate sender and the job could both send the same row | Claim with `for update skip locked` and a `sending` status with stuck-row recovery, plus Resend `Idempotency-Key` = `dedupe_key` (§11) |
| 6 | Edge case | A completed-payment webhook for a payment already marked `expired` locally was undefined | The webhook always records money taken, whatever the local status (§8) |
| 7 | Edge case / performance | Backfill from an old `start_date` was unbounded, with one email per backfilled month | `start_date` ≥ today − 90 days; invoice email only for the current month (§7) |
| 8 | Performance | Per-lease loops in the job would grow with lease count | Set-based `insert … select` over `generate_series` with `returning` (§7) |
| 9 | Performance | `invoice_balances` grouped only by `i.id`, so a `lease_id` filter couldn't be pushed into the view | Group by every invoice column (schema) |
| 10 | Integrity | Row triggers don't fire on `TRUNCATE`, which weakened the "append-only even for system" claim | Statement-level truncate trigger |
| 11 | Error handling | No named handling per failure class | One error mapper with a table of Postgres codes, Stripe, storage and unknown errors (§5) |
| 12 | Reuse | Hand-rolled `jose` JWKS verification wouldn't cover both Supabase key types | supabase-js `auth.getClaims()` (§4) |
| 13 | Edge case | Photo upload vs. row insert order was unspecified; requests could be opened on ended leases | Pre-generated id, upload, then insert; clean up on failure; RLS requires a current lease (§12) |
| 14 | Consistency | "Delete property refused if any unit has a lease" didn't match an FK that blocked any units at all | Units cascade from property; leases restrict, mapped to `409 in_use` |

### Accepted trade-offs (deliberate, documented)
- The invoice row lock is held during the Stripe API call (10 s timeout). This is simpler and
  correct, and contention is per invoice only (§8).
- `invoice_balances.is_overdue` uses the real clock while jobs use `as_of`. Tests that need
  "overdue" at a fake date check `due_date` against `as_of` directly, not the view's flag.
- `profiles.email` is copied once at sign-up. Changing your email is out of scope, so it can't
  drift. If email change is ever added, it needs a sync trigger.
- The hourly job is a single caller with idempotent steps, so no job lock is needed. Overlapping
  runs are safe by constraint.

### Still open
Nothing blocks the build. The owner's decisions are in: credit adjustments, free tiers only,
a login-only live demo with a nightly reset, and a public repo.

### Verdict
**Safe to start building.** The foundations (two-mode RLS access, ledger constraints,
idempotent webhook and job, outbox) are sound, and the first migration can be written straight
from 03-schema.md.
