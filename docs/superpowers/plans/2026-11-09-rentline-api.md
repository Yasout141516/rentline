# Rentline API Implementation Plan (plan 1 of 3)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Rentline REST API: Express 5 + TypeScript on Supabase Postgres with RLS, monthly invoices, late fees, credits, Stripe payments, an email outbox and maintenance requests, all covered by tests that run in CI.

**Architecture:** Every request runs in one Postgres transaction that starts in *user mode* (`role authenticated`, RLS on) and enters *system mode* only through `tx.asSystem()` for money, jobs, the webhook and the outbox. Services hold the business rules; routes only validate (Zod) and shape responses; jobs are plain functions of `(sql, asOf)`. Outside services (Stripe, Resend, Storage, Supabase Auth) sit behind small interfaces so tests can swap fakes in where the network would be needed.

**Tech Stack:** Node.js 22, TypeScript (strict, ESM), Express 5, Zod, `@asteasolutions/zod-to-openapi` + `swagger-ui-express`, `postgres` (postgres.js), `@supabase/supabase-js`, `stripe`, `resend`, `multer`, `file-type`, `helmet`, `cors`, `express-rate-limit`, Vitest + Supertest, Supabase CLI.

**Spec:** `docs/01-prd.md`, `docs/02-design.md` (§ numbers below refer to it), `docs/03-schema.md`.

**The other two plans** (written after this one ships): plan 2 is the Next.js web app (§13); plan 3 is seed data, deploys, `jobs.yml`, the nightly demo reset and the README (§15, §17).

## Global Constraints

- Node.js 22 LTS; TypeScript `strict: true`; ESM (`"type": "module"`).
- Money is integer cents everywhere (`*_cents`); no floats in code, tests or JSON.
- Dates in JSON are `YYYY-MM-DD`; "today" and every month boundary use `America/Toronto`.
- Currency is `cad`; Stripe runs in test mode only.
- The `db` module exports no bare query function (§4); route code only ever sees a `Tx`.
- Every app table lives in schema `app`; `app` is not added to the exposed API schemas in `supabase/config.toml`.
- Error body is always `{ "error": { "code", "message", "details"? } }`; a row the caller can't see is `404`, never `403`.
- List endpoints: `limit` default 50, max 100; `offset` default 0.
- Secrets come only from env vars; `.env.example` is committed, `.env` never is.
- Commit after every task with a `feat:`/`test:`/`chore:` message ending in the `Co-Authored-By` line.

## Review Focus

1. **Money sent as a non-integer** (`1450.5`, `"1450"`, `-100`, `1e3`): expect `400 validation_failed`, never a stored value. Test added in Task 9.
2. **The job running late in the evening Toronto time on the last day of a month** (already the next day in UTC): expect invoices for the Toronto month, not the UTC one. Test added in Task 6.
3. **A tenant who signs up with different capitalisation of the invited email** (`Daniel.O@Example.com` vs `daniel.o@example.com`): expect the lease to link. Test added in Task 3.
4. **The same Stripe event delivered twice at the same moment**: expect one ledger row, both deliveries answered `200`. Test added in Task 10.
5. **Changing `tenant_email` after the tenant has linked**: expect `422 tenant_already_linked` and no new invitation. Test added in Task 5.

---

## File structure

```
rentline/
  .env.example                       all env vars, with local defaults
  .github/workflows/ci.yml           Task 14
  supabase/
    config.toml                      Task 1 (supabase init), auth confirmations on
    migrations/20261109000000_init.sql   Task 1: 03-schema.md SQL + updated_at trigger + bucket
  api/
    package.json  tsconfig.json  vitest.config.ts
    src/
      config.ts          loadConfig(): Config
      server.ts          entry: loadConfig → real deps → createApp → listen
      app.ts             createApp(deps: AppDeps): express.Express
      errors.ts          AppError, mapError
      dates.ts           todayInToronto, monthsBetween helpers
      money.ts           assertPayableAmount
      validation.ts      shared Zod pieces (cents, isoDate, pagination, uuid params)
      db/db.ts           createDb(url): Db  (requestTx, systemTx, end)
      auth/verify.ts     createAuthVerifier(...): AuthVerifier
      auth/middleware.ts requireAuth, requireRole
      integrations/stripe.ts   createStripeCheckout(...): CheckoutApi; verifyStripeEvent
      integrations/mailer.ts   createResendMailer, createLogMailer
      integrations/storage.ts  createSupabaseStorage(...): PhotoStorage
      services/profiles.ts     getMe (+ tenant linking)
      services/properties.ts   properties + units
      services/leases.ts
      services/invoices.ts     reads, ledger, dashboard
      services/invoiceLock.ts  lockInvoice
      services/adjustments.ts
      services/payments.ts     checkout, cancel, reads
      services/webhooks.ts     handleStripeEvent
      services/outbox.ts       enqueueEmail, sendPendingEmails, templates
      services/maintenance.ts
      jobs/generateInvoices.ts  jobs/lateFees.ts  jobs/expirePayments.ts  jobs/run.ts
      routes/*.ts              one router per resource
      openapi.ts
    scripts/job.ts            npm run job -- --as-of YYYY-MM-DD
    test/
      helpers/env.ts users.ts app.ts fixtures.ts
      fakes/checkout.ts mailer.ts
      *.test.ts
```

## Shared interfaces (defined once; later tasks use these exact names)

```ts
// src/db/db.ts
export type Sql = postgres.TransactionSql;
export type Claims = { sub: string; email: string; role: string; [k: string]: unknown };
export interface Tx { sql: Sql; asSystem<T>(fn: (sql: Sql) => Promise<T>): Promise<T>; afterCommit(fn: () => void): void }
export interface Db {
  requestTx<T>(claims: Claims, fn: (tx: Tx) => Promise<T>): Promise<T>;
  systemTx<T>(fn: (sql: Sql) => Promise<T>): Promise<T>;
  end(): Promise<void>;
}
// src/errors.ts
export class AppError extends Error { constructor(public status: number, public code: string, message: string, public details?: unknown) }
// src/app.ts
export interface AppDeps { config: Config; db: Db; auth: AuthVerifier; checkout: CheckoutApi; mailer: Mailer; storage: PhotoStorage; now: () => Date }
export interface AuthVerifier { verify(token: string): Promise<Claims> }            // throws AppError 401 / 503
export interface CheckoutApi {
  createSession(p: { paymentId: string; amountCents: number; description: string; successUrl: string; cancelUrl: string; expiresAt: Date; idempotencyKey: string }): Promise<{ sessionId: string; url: string }>;
  expireSession(sessionId: string): Promise<void>;
}
export interface Mailer { send(m: { to: string; subject: string; html: string; idempotencyKey: string }): Promise<void> }
export interface PhotoStorage { upload(path: string, bytes: Buffer, contentType: string): Promise<void>; remove(paths: string[]): Promise<void>; signedUrl(path: string, seconds: number): Promise<string> }
```

`req.profile: { id: string; role: 'landlord' | 'tenant'; email: string }` and `req.claims: Claims` are set by `requireAuth` (Task 3).

---

### Task 1: Repo scaffold, Supabase and the first migration

**Files:**
- Create: `api/package.json`, `api/tsconfig.json`, `api/vitest.config.ts`, `.env.example`, `supabase/config.toml` (via `npx supabase init`), `supabase/migrations/20261109000000_init.sql`
- Test: `api/test/schema.test.ts`, `api/test/helpers/env.ts`

**Interfaces:**
- Produces: the full schema of `03-schema.md`; `test/helpers/env.ts` exporting `testEnv: { databaseUrl, supabaseUrl, anonKey, serviceRoleKey }` read from `supabase status -o env` (run once, cached) or from env vars in CI.

- [ ] **Step 1: Scaffold.** `npx supabase init` at the repo root; in `config.toml` set `[auth.email] enable_confirmations = true` and leave `[api] schemas` without `app`. `api/package.json` scripts: `test` = `vitest run`, `typecheck` = `tsc --noEmit`, `lint` = `eslint .`, `dev` = `tsx watch src/server.ts`, `job` = `tsx scripts/job.ts`. `vitest.config.ts`: `fileParallelism: false`, `testTimeout: 20000`.
- [ ] **Step 2: Write the migration.** Copy every SQL block of `03-schema.md` verbatim, in order (extensions, schema, types, tables, triggers, view, RLS, grants, policies), then add: a shared `app.touch_updated_at()` `before update` trigger on `properties`, `units`, `leases`, `maintenance_requests`; and `insert into storage.buckets (id, name, public) values ('maintenance-photos', 'maintenance-photos', false)`.
- [ ] **Step 3: Write the failing schema tests** in `test/schema.test.ts`, connecting as `postgres` with `testEnv.databaseUrl`. Seed the minimum rows by direct SQL (profiles need `auth.users`: create users with the admin API helper from Task 2's `createTestUser`, or here with `supabase.auth.admin.createUser`):
  - `ledger rejects UPDATE` → `update app.ledger_entries set amount_cents = 1` rejects with message containing `append-only`
  - `ledger rejects DELETE` and `ledger rejects TRUNCATE` → same
  - `ledger sign check` → inserting a `payment` row with `amount_cents = 500` rejects (`23514`)
  - `one late fee per invoice` → second `late_fee` row for the same invoice rejects (`23505`)
  - `leases cannot overlap on a unit` → second lease `2026-06-01..2027-05-31` on a unit leased `2026-01-01..2026-12-31` rejects (`23P01`)
  - `lease landlord must own the unit` → lease with another landlord's id rejects (`23503`)
  - `ledger lease must match invoice lease` → rejects (`23503`)
  - `invoice_balances derives status` → invoice with rent 140000, payment −40000 → `status = 'partially_paid'`, `outstanding_cents = 100000`; add adjustment −100000 → `status = 'paid'`
  - `sign-up creates a profile` → `auth.admin.createUser({ email, password, email_confirm: true, user_metadata: { role: 'tenant', full_name: 'T' } })` → one `app.profiles` row with role `tenant`
- [ ] **Step 4: Run** `npx supabase start` then `npx supabase db reset`, then `npm test --prefix api -- schema`. Expected: all pass. (Write the tests before Step 2's SQL if you want to see them fail first; either order is fine for a migration.)
- [ ] **Step 5: Commit** `chore: scaffold api, supabase and the initial schema migration`.

### Task 2: Config, errors and the two-mode database module

**Files:**
- Create: `api/src/config.ts`, `api/src/errors.ts`, `api/src/db/db.ts`, `api/test/helpers/users.ts`
- Test: `api/test/db.test.ts`, `api/test/errors.test.ts`

**Interfaces:**
- Produces: `loadConfig(env = process.env): Config` (throws listing every missing required var). Required: `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `JOB_SECRET`, `WEB_ORIGIN`. Optional: `RESEND_API_KEY` (unset → log mailer), `EMAIL_FROM` (default `Rentline <onboarding@resend.dev>`), `ALLOW_AS_OF` (default `false`), `PORT` (default `3001`); `AppError`; `mapError(err: unknown, requestId: string): { status: number; body: { error: { code: string; message: string; details?: unknown } } }`; `createDb(url: string): Db`; `isRetryable(err): boolean` (Postgres `40001`, `40P01`); test helper `createTestUser(role: 'landlord'|'tenant', opts?: { email?: string }): Promise<{ id: string; email: string; token: string; claims: Claims }>` (admin `createUser` with `email_confirm: true`, then `signInWithPassword` for a real access token; claims are the decoded token payload).

- [ ] **Step 1: Write failing tests** in `db.test.ts`:
  - `user mode applies RLS` → landlord A inserts a property in `requestTx(A.claims)`; `requestTx(B.claims, tx => tx.sql\`select * from app.properties where id = ${id}\`)` returns 0 rows
  - `asSystem bypasses RLS and returns to user mode` → inside one `requestTx(B)`: `asSystem` sees the row; after it returns, `select current_setting('role')` is `authenticated`
  - `asSystem restores user mode when it throws` → `asSystem(() => { throw new Error('x') })` caught inside the tx; the next query's `current_setting('role')` is `authenticated`
  - `systemTx runs as the privileged role` → `select current_user` is `postgres` and RLS is not applied
  - `serialization failure is retried once` → a fn that throws a fake `{ code: '40001' }` the first time succeeds on the second call; a fn that always throws it rejects with `AppError(503, 'try_again')`
  - `afterCommit runs only after commit` → callback registered in a tx that later throws is never called
- [ ] **Step 2: Write failing tests** in `errors.test.ts` for `mapError`: Zod error → `400 validation_failed` with `details`; `{ code: '23P01' }` → `409 lease_overlap`; `23505` → `409 conflict` with `details.constraint`; `23503` → `409 in_use`; `23514` → `422 invalid_value`; `42501` (RLS denied an insert) → `404 not_found`; `AppError(422,'invalid_amount')` passes through; unknown error → `500 internal_error`, message contains the request id, no stack in the body.
- [ ] **Step 3: Run** `npm test --prefix api -- db errors`. Expected: FAIL (modules missing).
- [ ] **Step 4: Implement.** `requestTx`: `sql.begin(async t => { await t\`set local role authenticated\`; await t\`select set_config('request.jwt.claims', ${JSON.stringify(claims)}, true)\`; ... })`. `asSystem`: `reset role`, run `fn`, then `set local role authenticated` in a `finally`. Wrap both `requestTx` and `systemTx` in one retry on `isRetryable`. `afterCommit` callbacks are collected and run after `sql.begin` resolves.
- [ ] **Step 5: Run** the same command. Expected: PASS.
- [ ] **Step 6: Commit** `feat: config, error mapping and the user/system database module`.

### Task 3: App shell, auth and `GET /v1/me` with tenant linking

**Files:**
- Create: `api/src/app.ts`, `api/src/server.ts`, `api/src/auth/verify.ts`, `api/src/auth/middleware.ts`, `api/src/services/profiles.ts`, `api/src/routes/me.ts`, `api/test/helpers/app.ts`, `api/test/fakes/checkout.ts`, `api/test/fakes/mailer.ts`
- Test: `api/test/auth.test.ts`

**Interfaces:**
- Consumes: `Db`, `AppError`, `mapError`, `createTestUser`.
- Produces: `createApp(deps: AppDeps)`; `requireAuth` (sets `req.claims`, `req.profile`, and `req.tx`-runner `req.run<T>(fn: (tx: Tx) => Promise<T>): Promise<T>` = `deps.db.requestTx(req.claims, fn)`); `requireRole(role)`; `createAuthVerifier(supabaseUrl, anonKey): AuthVerifier` using `supabase.auth.getClaims(token)`; `getMe(tx: Tx, claims: Claims): Promise<Profile & { linked_leases: number }>`; test helper `makeTestApp(overrides?): { app, deps, mailer: RecordingMailer, checkout: FakeCheckout }` using real `Db`, real `AuthVerifier`, real storage, fake checkout and mailer, and `now` overridable.

- [ ] **Step 1: Write failing tests** in `auth.test.ts` (Supertest against `makeTestApp().app`):
  - `GET /health` → `200 { ok: true }` without a token
  - `no token` → `401 unauthorized`; `garbage token` → `401 unauthorized`
  - `GET /v1/me` as a landlord → `200` with `{ id, role: 'landlord', email, full_name }`
  - `tenant linking` → a system-SQL lease with `tenant_email = 'daniel.<rand>@example.com'`; tenant created with that email calls `/v1/me` → `linked_leases: 1` and the lease's `tenant_id` is the tenant
  - `tenant linking ignores case` (Review Focus 3) → invite `daniel.<rand>@example.com`, sign up as `Daniel.<rand>@Example.com` → linked
  - `a landlord with the invited email is not linked` → `linked_leases: 0`, `tenant_id` still null
  - `requireRole` → a tenant calling a landlord-only test route gets `403 forbidden`
- [ ] **Step 2: Run** `npm test --prefix api -- auth`. Expected: FAIL.
- [ ] **Step 3: Implement.** `app.ts` mounts in order: request id (`crypto.randomUUID()`, echoed as `X-Request-Id`), `helmet()`, `cors({ origin: config.webOrigin })`, rate limit (300 req / 15 min per IP), the Stripe webhook router with `express.raw({ type: 'application/json' })` (Task 10 fills it), `express.json({ limit: '100kb' })`, routers, then the error middleware calling `mapError` and logging `{ requestId, code, stack }` as JSON. Linking in `getMe`: `tx.asSystem` → `update app.leases set tenant_id = $id where tenant_id is null and lower(tenant_email) = lower($email)` only when the profile role is `tenant`.
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: app shell, JWT auth and /v1/me with tenant linking`.

### Task 4: Properties and units

**Files:**
- Create: `api/src/validation.ts`, `api/src/services/properties.ts`, `api/src/routes/properties.ts`
- Test: `api/test/properties.test.ts`, `api/test/helpers/fixtures.ts`

**Interfaces:**
- Produces: `pagination` Zod schema (`limit` 1–100 default 50, `offset` ≥ 0 default 0); `uuidParam`; fixtures `makeProperty(landlord)`, `makeUnit(landlord, propertyId)`, `makeLease(landlord, unitId, overrides)` (direct system SQL, return ids) used by every later test file.

- [ ] **Step 1: Write failing tests:**
  - `create and list own properties` → `POST /v1/properties` `{ name, address_line1, city, province, postal_code }` → `201` with `id`; `GET` lists it
  - `another landlord gets 404` on `GET/PATCH/DELETE /v1/properties/:id`
  - `tenant gets 403` on `POST /v1/properties`
  - `create, rename and delete a unit` → `POST /v1/properties/:id/units { label: '2B', bedrooms: 1 }`; duplicate label → `409 conflict`
  - `deleting a property deletes its empty units` → `204`, units gone
  - `deleting a property with a leased unit` → `409 in_use`; same for `DELETE /v1/units/:id`
  - `limit=101` → `400 validation_failed`
- [ ] **Step 2: Run** `npm test --prefix api -- properties`. Expected: FAIL.
- [ ] **Step 3: Implement** with user-mode SQL only (`landlord_id` comes from `req.profile.id`; RLS and the column grants enforce the rest). `404` when an update/delete affects 0 rows.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: properties and units`.

### Task 5: Leases and the outbox enqueue

**Files:**
- Create: `api/src/dates.ts`, `api/src/services/outbox.ts` (enqueue part), `api/src/services/leases.ts`, `api/src/routes/leases.ts`
- Test: `api/test/leases.test.ts`, `api/test/dates.test.ts`

**Interfaces:**
- Produces: `todayInToronto(now: Date): string`; `addDays(date: string, n: number): string`; `firstOfMonth(date: string): string`; `enqueueEmail(sql: Sql, e: { dedupeKey: string; to: string; template: 'lease_invitation'|'invoice_created'|'payment_receipt'|'maintenance_opened'; payload: Record<string, unknown> }): Promise<boolean>` (`on conflict (dedupe_key) do nothing`, returns whether a row was inserted).

- [ ] **Step 1: Write failing tests:**
  - `dates.test.ts`: `todayInToronto(new Date('2026-12-01T04:30:00Z')) === '2026-11-30'`; `addDays('2026-10-01', 5) === '2026-10-06'`
  - `create a lease queues one invitation` → `201`; one `email_outbox` row, template `lease_invitation`, `dedupe_key = 'lease_invitation:<lease_id>:1'`
  - `overlapping lease` → `409 lease_overlap`
  - `start_date more than 90 days ago` (now = 2026-11-09, start 2026-08-10) → `422 start_date_too_old`; 2026-08-11 → `201`
  - `rent below 100 cents` → `400`; `late_fee_cents = 50` → `400`; `due_day = 29` → `400`
  - `GET /v1/leases` → landlord sees theirs; linked tenant sees theirs; other tenant sees none; `GET /v1/leases/:id` for another tenant → `404`
  - `PATCH tenant_email while unlinked` → `200` and a second invitation (`...:2`)
  - `PATCH tenant_email after linking` (Review Focus 5) → `422 tenant_already_linked`, no new outbox row
  - `PATCH end_date before the last invoiced month` → `422 end_date_before_invoices` (create an invoice for 2026-11 by system SQL, try `end_date = 2026-10-31`)
  - `DELETE a lease with invoices` → `409 in_use`; without → `204`
  - `column grants hold in user mode` (§14) → inside `requestTx(landlord)`, inserting a lease with `tenant_id` set and updating `tenant_id` both reject (`42501`)
  - `resend-invite after linking` → `422 tenant_already_linked`
- [ ] **Step 2: Run** `npm test --prefix api -- leases dates`. Expected: FAIL.
- [ ] **Step 3: Implement.** Insert the lease in user mode (the column grant blocks `tenant_id`), then enqueue the invitation in `tx.asSystem` in the same transaction; the invite sequence number is `1 + count of earlier lease_invitation rows for the lease`. `PATCH` accepts only `end_date`, `late_fee_cents`, `grace_days`, `tenant_email` (Zod `.strict()`).
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: leases with invitations through the outbox`.

### Task 6: Job steps: invoices and late fees

**Files:**
- Create: `api/src/jobs/generateInvoices.ts`, `api/src/jobs/lateFees.ts`
- Test: `api/test/jobs.invoices.test.ts`, `api/test/jobs.lateFees.test.ts`

**Interfaces:**
- Produces: `generateInvoices(sql: Sql, asOf: string): Promise<{ invoices_created: number }>`; `applyLateFees(sql: Sql, asOf: string): Promise<{ late_fees_applied: number }>`. Both take a system-mode `Sql`.

- [ ] **Step 1: Write failing tests** (each test makes its own landlord, unit and lease; assert on that lease only):
  - `creates one invoice per covered month and a rent_charge row` → lease 2026-09-01..2027-08-31, due day 1, rent 140000; `asOf = '2026-11-09'` → invoices for 2026-09, 10, 11 with `amount_cents = 140000`, each with exactly one `rent_charge` ledger row of `140000`
  - `running twice creates nothing new` → second call returns `invoices_created` 0 for this lease (query the lease's invoices: still 3)
  - `two runs at once create one invoice per month` → `Promise.all` of two `systemTx` calls; still 3
  - `due_date respects start_date` → lease starting 2026-11-20 with due day 1 → `due_date = '2026-11-20'`
  - `no invoices outside the lease` → lease ending 2026-10-15 → no 2026-11 invoice
  - `invoice email only for the current month and a linked tenant` → backfilled months queue no `invoice_created`; the `asOf` month queues one, and none when `tenant_id` is null
  - `Toronto month at the UTC boundary` (Review Focus 2) → `generateInvoices(sql, todayInToronto(new Date('2026-12-01T04:30:00Z')))` creates the 2026-11 invoice and no 2026-12 invoice
  - late fees: `added after the grace days` → due 2026-10-01, grace 5, fee 5000: `asOf '2026-10-06'` adds nothing, `'2026-10-07'` adds one `late_fee` row of 5000
  - `only once` → run again and run two at once: still one
  - `not when paid` → a −140000 `payment` row (with a `payments` row) before the run → no fee
  - `not when late_fee_cents is 0`
- [ ] **Step 2: Run** `npm test --prefix api -- jobs`. Expected: FAIL.
- [ ] **Step 3: Implement** both as single set-based statements (§7, §9): `insert … select` over `generate_series(date_trunc('month', start_date), date_trunc('month', $asOf), '1 month')` joined to leases, `on conflict (lease_id, period_month) do nothing returning *`, then insert the `rent_charge` rows and outbox rows from that set in the same statement chain (CTEs). Late fees use `invoice_balances` and `on conflict do nothing` against `one_late_fee_per_invoice`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: invoice generation and late fees`.

### Task 7: Invoice reads, ledger and dashboard

**Files:**
- Create: `api/src/services/invoices.ts`, `api/src/routes/invoices.ts`, `api/src/routes/dashboard.ts`
- Test: `api/test/invoices.test.ts`

- [ ] **Step 1: Write failing tests** (set up a lease and run `generateInvoices` for it):
  - `GET /v1/invoices?lease_id=` → rows from `invoice_balances` with `status`, `outstanding_cents`, `is_overdue`
  - filters: `status=partially_paid` and `overdue=true` return only matching rows
  - `GET /v1/invoices/:id` → includes `ledger` and `payments` arrays; another tenant → `404`
  - `GET /v1/leases/:id/ledger` → oldest first, each row has `running_balance_cents`; after rent 140000, fee 5000, payment −40000 the last running balance is `105000`
  - `GET /v1/dashboard` → `outstanding_cents`, `overdue_count`, `open_maintenance_count`, `received_this_month_cents`, and `months` of length 12, each `{ month: 'YYYY-MM', charged_cents, collected_cents }`; values match the fixture; a tenant → `403`
- [ ] **Step 2: Run** `npm test --prefix api -- invoices`. Expected: FAIL.
- [ ] **Step 3: Implement** all reads in user mode. Running balance via `sum(amount_cents) over (order by created_at, id)`. `months` is one grouped query over `ledger_entries` joined to invoices for `period_month` in the last 12 Toronto months (charged = rent + fees, collected = −payments).
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: invoice reads, ledger with running balance, dashboard`.

### Task 8: Invoice lock, money rule and credit adjustments

**Files:**
- Create: `api/src/money.ts`, `api/src/services/invoiceLock.ts`, `api/src/services/adjustments.ts`
- Modify: `api/src/routes/invoices.ts`
- Test: `api/test/money.test.ts`, `api/test/adjustments.test.ts`

**Interfaces:**
- Produces: `assertPayableAmount(amountCents: number, availableCents: number): void` → throws `AppError(422,'invalid_amount')` if `< 100` or if `0 < available − amount < 100`, `AppError(422,'invoice_overpayment')` if `amount > available`; `lockInvoice(sql: Sql, invoiceId: string): Promise<{ id, lease_id, landlord_id, tenant_id, outstanding_cents, pending_cents, period_month }>` (system mode, `select … for update` on `app.invoices`, then reads the balance view and the sum of `pending` payments).

- [ ] **Step 1: Write failing tests:**
  - `money.test.ts`: `(100, 145000)` ok; `(99, 145000)` → `invalid_amount`; `(145000, 145000)` ok; `(144950, 145000)` → `invalid_amount` (leaves 50); `(145001, 145000)` → `invoice_overpayment`
  - `adjustments.test.ts`: `credit lowers what is owed` → `POST /v1/invoices/:id/adjustments { amount_cents: 5000, reason: 'Waive late fee' }` → `201`; ledger row type `adjustment`, `amount_cents = -5000`, `created_by` = landlord, `description` = reason
  - `reason shorter than 5 characters` → `400`
  - `credit above what is owed minus pending` → with a 100000 `pending` payment on a 145000 invoice, crediting 50000 → `422 invoice_overpayment`
  - `full credit makes the invoice paid` → status `paid`
  - `tenant cannot credit` → `403`; `another landlord's invoice` → `404`
- [ ] **Step 2: Run** `npm test --prefix api -- money adjustments`. Expected: FAIL.
- [ ] **Step 3: Implement** the route: user-mode select of the invoice (404 if not visible), then `tx.asSystem(sql => { lockInvoice; assertPayableAmount(amount, outstanding − pending); insert adjustment })`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: credit adjustments under the invoice lock`.

### Task 9: Checkout, payment reads and cancel

**Files:**
- Create: `api/src/integrations/stripe.ts` (checkout part), `api/src/services/payments.ts`, `api/src/routes/payments.ts`
- Modify: `api/src/routes/invoices.ts`, `api/test/fakes/checkout.ts`
- Test: `api/test/payments.test.ts`

**Interfaces:**
- Consumes: `lockInvoice`, `assertPayableAmount`, `CheckoutApi`.
- Produces: `createStripeCheckout(secretKey: string): CheckoutApi` (Stripe SDK, `timeout: 10000`, `maxNetworkRetries: 0`; session `mode: 'payment'`, `currency: 'cad'`, `payment_method_types: ['card']`, `metadata.payment_id`, `client_reference_id`, `expires_at` in seconds; errors become `AppError(502,'payment_provider_unavailable')`); `FakeCheckout` records calls, returns `{ sessionId: 'cs_test_<n>', url: 'https://checkout.test/<n>' }`, and can be told to fail.

- [ ] **Step 1: Write failing tests:**
  - `checkout creates a pending payment and a session` → tenant `POST /v1/invoices/:id/checkout { amount_cents: 145000 }` → `201 { payment_id, checkout_url }`; payment `pending`; fake called with `idempotencyKey: 'checkout-<payment_id>'`, `currency` implied CAD, `expiresAt = now + 30 min`, description `Rent — Unit 2B, October 2026`
  - `non-integer money is rejected` (Review Focus 1) → `1450.5`, `"145000"`, `-100`, `0` → `400 validation_failed`
  - `pending payments reduce what is available` → after a 100000 pending checkout on 145000, a second for 50000 → `422 invoice_overpayment`; for 45000 → `201`
  - `two checkouts at once for the full amount` → `Promise.all` → one `201`, one `422 invoice_overpayment`
  - `Stripe failure leaves nothing behind` → fake set to fail → `502 payment_provider_unavailable`, no `payments` row
  - `landlord cannot check out` → `403`; `other tenant's invoice` → `404`
  - `GET /v1/payments/:id` → owner tenant and the landlord see it; another tenant → `404`; `GET /v1/payments` lists only the caller's
  - `cancel expires the session and frees the amount` → `POST /v1/payments/:id/cancel` → `200`, status `expired`, fake `expireSession` called; a new full checkout now succeeds; cancelling a `succeeded` payment → `422 payment_not_pending`
- [ ] **Step 2: Run** `npm test --prefix api -- payments`. Expected: FAIL.
- [ ] **Step 3: Implement** exactly the six steps of §8 in one request transaction; success and cancel URLs are `${config.webOrigin}/tenant/payments/<id>` and `${config.webOrigin}/tenant/invoices/<invoice_id>?cancelled=<payment_id>`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: Stripe checkout with partial payments, reads and cancel`.

### Task 10: Stripe webhook and the stale-payment safety net

**Files:**
- Create: `api/src/services/webhooks.ts`, `api/src/routes/webhooks.ts`, `api/src/jobs/expirePayments.ts`
- Modify: `api/src/integrations/stripe.ts` (add `verifyStripeEvent(raw: Buffer, signature: string, secret: string): Stripe.Event` using `stripe.webhooks.constructEvent`)
- Test: `api/test/webhooks.test.ts`, `api/test/jobs.expire.test.ts`

**Interfaces:**
- Produces: `handleStripeEvent(sql: Sql, event: Stripe.Event, now: Date): Promise<'recorded' | 'duplicate' | 'ignored'>`; `expireStalePayments(sql: Sql, now: Date): Promise<{ payments_expired: number }>` (pending and `created_at < now − 2 h`). Test helper `signedEvent(payload): { body: string; signature: string }` using `stripe.webhooks.generateTestHeaderString` with the test webhook secret.

- [ ] **Step 1: Write failing tests:**
  - `bad signature` → `400 invalid_signature`, nothing stored
  - `completed event records the payment once` → `checkout.session.completed` with `payment_status: 'paid'`, `metadata.payment_id`, `payment_intent: 'pi_1'` → `200`; payment `succeeded`, one `payment` ledger row of `−amount`, one `payment_receipt` outbox row
  - `same event again` → `200`, still one ledger row
  - `same event twice at the same moment` (Review Focus 4) → `Promise.all` of two posts → both `200`, one ledger row
  - `completed after the payment was expired locally` → still recorded as `succeeded`
  - `overpayment is recorded and flagged` → credit the invoice down after checkout, then complete → ledger row written, `flagged_overpayment = true`
  - `expired event frees the amount` → status `expired`
  - `unknown payment id` and `other event types` → `200`, `ignored`, event row stored
  - `expireStalePayments` → a `pending` payment created 3 h before `now` becomes `expired`; one created 1 h before stays `pending`
- [ ] **Step 2: Run** `npm test --prefix api -- webhooks expire`. Expected: FAIL.
- [ ] **Step 3: Implement** §8's webhook steps inside `db.systemTx`; mount the router before `express.json` with `express.raw({ type: 'application/json' })`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: idempotent Stripe webhook and stale payment expiry`.

### Task 11: Sending the outbox

**Files:**
- Create: `api/src/integrations/mailer.ts`
- Modify: `api/src/services/outbox.ts` (sender + templates), `api/src/app.ts` (after-commit kick)
- Test: `api/test/outbox.test.ts`

**Interfaces:**
- Produces: `renderEmail(template, payload): { subject: string; html: string }`; `sendPendingEmails(db: Db, mailer: Mailer, now: Date): Promise<{ sent: number; failed: number; requeued: number }>`; `createResendMailer(apiKey: string, from: string): Mailer` (passes `idempotencyKey` as Resend's `Idempotency-Key`); `createLogMailer(): Mailer` (logs JSON, used when `RESEND_API_KEY` is unset); route helper `kickOutbox(req)` registering `tx.afterCommit(() => void sendPendingEmails(...))`.

- [ ] **Step 1: Write failing tests:**
  - `sends pending rows once` → enqueue 2 rows; `sendPendingEmails` → `sent: 2`, rows `sent`, mailer got 2 calls with `idempotencyKey` = each `dedupe_key`
  - `two senders at once send each row once` → `Promise.all` of two calls → mailer total calls = 2
  - `failures are retried then marked failed` → mailer throws: after 1st run status `pending`, `attempts = 1`; after the 5th run `failed`
  - `stuck sending rows are requeued` → a row in `sending` with `claimed_at` 11 min ago becomes `pending` (`requeued: 1`)
  - `a rolled-back action leaves no email` → in a `requestTx` that enqueues and then throws, no outbox row exists
  - `templates` → `renderEmail('payment_receipt', { amount_cents: 145000, ... })` subject contains `$1,450.00`
- [ ] **Step 2: Run** `npm test --prefix api -- outbox`. Expected: FAIL.
- [ ] **Step 3: Implement** the claim statement from §11 (`for update skip locked limit 50`), then send each claimed row outside the claim transaction and mark `sent` / back to `pending` / `failed` at 5 attempts. Call `kickOutbox` in the lease create/patch/resend, webhook and maintenance routes.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: outbox sender with claims, retries and Resend idempotency`.

### Task 12: Job endpoint and CLI

**Files:**
- Create: `api/src/jobs/run.ts`, `api/src/routes/internal.ts`, `api/scripts/job.ts`
- Test: `api/test/jobs.run.test.ts`

**Interfaces:**
- Produces: `runJob(deps: Pick<AppDeps,'db'|'mailer'|'now'>, asOf: string): Promise<{ as_of: string; invoices_created: number; late_fees_applied: number; payments_expired: number; emails: { sent: number; failed: number; requeued: number } }>` running the four steps in §10 order (steps 1–3 each in its own `systemTx`).

- [ ] **Step 1: Write failing tests:**
  - `wrong or missing X-Job-Secret` → `401`
  - `runs all steps` → `200` with the summary shape above
  - `as_of is refused unless ALLOW_AS_OF=true` → `400 as_of_not_allowed` with it off; accepted with it on
  - `as_of defaults to todayInToronto(now)`
- [ ] **Step 2: Run** `npm test --prefix api -- jobs.run`. Expected: FAIL.
- [ ] **Step 3: Implement.** `scripts/job.ts` parses `--as-of`, builds real deps from `loadConfig()`, calls `runJob`, prints the JSON summary, exits `1` on error.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: scheduled job endpoint and CLI`.

### Task 13: Maintenance requests with photos

**Files:**
- Create: `api/src/integrations/storage.ts`, `api/src/services/maintenance.ts`, `api/src/routes/maintenance.ts`, `api/test/fixtures/photo.jpg` (a real tiny JPEG), `api/test/fixtures/fake.jpg` (text bytes named .jpg)
- Test: `api/test/maintenance.test.ts`

**Interfaces:**
- Produces: `createSupabaseStorage(supabaseUrl, serviceRoleKey, bucket = 'maintenance-photos'): PhotoStorage`; `sniffImageType(bytes: Buffer): 'image/jpeg' | 'image/png' | 'image/webp' | null` (via `file-type`).

- [ ] **Step 1: Write failing tests:**
  - `tenant opens a request with photos` → multipart `lease_id`, `title`, `description`, two `photos` → `201`; two `maintenance_photos` rows, files exist at `<request_id>/<uuid>.jpg`, one `maintenance_opened` outbox row to the landlord
  - `fourth photo` → `400 too_many_photos`; `6 MB photo` → `400 photo_too_large`; `fake.jpg` → `400 unsupported_photo_type`; nothing stored in each case
  - `lease not current today` (ended yesterday) → `404` (RLS insert policy denies; map the `42501` RLS violation to `404 not_found`)
  - `upload failure cleans up` → storage stub fails on the 2nd file → `502 storage_unavailable`, the 1st file removed, no request row
  - `detail has signed URLs` → `GET /v1/maintenance/:id` → `photos[].url` set; another tenant → `404`
  - `landlord moves status forward only` → `open → in_progress` `200`, `in_progress → resolved` `200` with `resolved_at` set, `resolved → open` → `422 invalid_status_change`; tenant `PATCH` → `403`
  - `lists` → `GET /v1/maintenance?status=open` filters
- [ ] **Step 2: Run** `npm test --prefix api -- maintenance`. Expected: FAIL.
- [ ] **Step 3: Implement** §12's order: `multer.memoryStorage()` with `limits: { files: 3, fileSize: 5 * 1024 * 1024 }`, sniff every file, generate the request id, upload, then one user-mode transaction inserting the request (`landlord_id` from a user-mode lease read) and photo rows, enqueue the email in `asSystem`; on any failure remove uploaded paths.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat: maintenance requests with private photos`.

### Task 14: OpenAPI docs, cross-user sweep and CI

**Files:**
- Create: `api/src/openapi.ts`, `.github/workflows/ci.yml`
- Modify: `api/src/app.ts` (serve `/docs` and `/openapi.json`), route files (register their Zod schemas)
- Test: `api/test/docs.test.ts`, `api/test/isolation.test.ts`

- [ ] **Step 1: Write failing tests:**
  - `GET /openapi.json` → `200`, has paths for every route in §5 and `components.securitySchemes.bearerAuth`; `GET /docs/` → `200` HTML
  - `isolation.test.ts` (the highest-risk path, §14): build two landlords and two tenants with full data (property, unit, lease, invoice, payment, maintenance); for every `GET` route with `:id` in §5, call it as the *other* user of the same role → `404`; as a different role where the route allows both → still `404` for rows not theirs
- [ ] **Step 2: Run** `npm test --prefix api -- docs isolation`. Expected: docs FAIL; isolation should already PASS (if it fails, a route is reading in system mode: fix that route).
- [ ] **Step 3: Implement** `openapi.ts` with `OpenAPIRegistry` from each route's schemas; `swagger-ui-express` at `/docs` with `persistAuthorization: true`. `ci.yml` on push and pull request: checkout, Node 22, `npm ci --prefix api`, `supabase/setup-cli`, `supabase start`, export `supabase status -o env` values to the env, `npm run lint --prefix api`, `npm run typecheck --prefix api`, `npm test --prefix api`.
- [ ] **Step 4: Run** the full suite locally: `npm test --prefix api`. Expected: all PASS. Push a branch and confirm the CI run is green.
- [ ] **Step 5: Commit** `feat: OpenAPI docs, cross-user isolation sweep and CI`.

---

## Notes for the implementer

- **Test isolation:** the spec (§14) said "database reset between test files", but the ledger's truncate trigger makes truncation impossible by design. Tests isolate by data instead: every test creates its own users (random emails) and rows, and asserts only on those. CI starts from a fresh `supabase start`. *(Plan's call.)*
- **Local Docker:** the owner's network times out on Docker Hub (`registry-1`). If `supabase start` fails pulling images, pull the listed images through `mirror.gcr.io` and retag, as done for Ebb.
- **Explainability:** keep SQL in the service files as plain tagged templates; no query builders, no ORMs. Each service function should be readable top to bottom in an interview.
