# Rentline — Database Schema

*Status: draft for review, 2026-11-08. Postgres 15+ on Supabase. This is the intended first
migration; the real files go in `supabase/migrations/`.*

## Overview

```
auth.users ─1:1─ profiles
                   │ (landlord)
                   ├─< properties ─< units ─< leases >─ profiles (tenant, nullable until linked)
                   │                              │
                   │                              ├─< invoices ─< ledger_entries
                   │                              │       └─────< payments ──1:1── ledger_entries (payment rows)
                   │                              └─< maintenance_requests ─< maintenance_photos
stripe_events     (standalone: webhook idempotency)
email_outbox      (standalone: transactional outbox)
```

Conventions:
- `uuid` primary keys (`gen_random_uuid()`)
- `timestamptz` for moments and `date` for calendar days
- money is `integer` cents; the largest rent is far below 2^31 cents ($21M)
- `landlord_id` is copied down onto `units` and `leases` and kept correct by **composite foreign
  keys**, so the database itself guarantees a lease's landlord owns its unit and property, and
  RLS policies stay simple (`landlord_id = auth.uid()`)
- everything lives in schema `app`, which is not exposed through Supabase's REST API

## Migration SQL

```sql
create extension if not exists btree_gist;   -- for the "no overlapping leases" constraint
create extension if not exists citext;       -- case-insensitive emails

create schema app;
grant usage on schema app to authenticated;

-- ───────────────────────── types ─────────────────────────
create type app.user_role          as enum ('landlord', 'tenant');
create type app.ledger_entry_type  as enum ('rent_charge', 'late_fee', 'payment', 'adjustment');
create type app.payment_status     as enum ('pending', 'succeeded', 'expired', 'failed');
create type app.maintenance_status as enum ('open', 'in_progress', 'resolved');
create type app.email_status       as enum ('pending', 'sending', 'sent', 'failed');
create type app.email_template     as enum ('lease_invitation', 'invoice_created',
                                            'payment_receipt', 'maintenance_opened');

-- ───────────────────────── profiles ─────────────────────────
create table app.profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  role        app.user_role not null,
  full_name   text not null check (length(full_name) between 1 and 120),
  email       citext not null unique,
  created_at  timestamptz not null default now()
);

-- creates the profile from sign-up metadata; rejects sign-ups without a valid role
create function app.handle_new_user() returns trigger
language plpgsql security definer set search_path = '' as $$
begin
  insert into app.profiles (id, role, full_name, email)
  values (
    new.id,
    (new.raw_user_meta_data ->> 'role')::app.user_role,
    coalesce(nullif(new.raw_user_meta_data ->> 'full_name', ''), split_part(new.email, '@', 1)),
    new.email
  );
  return new;
end $$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function app.handle_new_user();

-- ───────────────────────── properties & units ─────────────────────────
create table app.properties (
  id             uuid primary key default gen_random_uuid(),
  landlord_id    uuid not null references app.profiles(id),
  name           text not null check (length(name) between 1 and 120),
  address_line1  text not null,
  city           text not null,
  province       text not null,
  postal_code    text not null,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  unique (id, landlord_id)                         -- target for the composite FK below
);
create index on app.properties (landlord_id);

create table app.units (
  id           uuid primary key default gen_random_uuid(),
  property_id  uuid not null,
  landlord_id  uuid not null,
  label        text not null check (length(label) between 1 and 40),
  bedrooms     smallint check (bedrooms between 0 and 20),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  -- deleting a property removes its units; a unit with a lease blocks that (leases → units is restrict)
  foreign key (property_id, landlord_id) references app.properties (id, landlord_id) on delete cascade,
  unique (property_id, label),
  unique (id, landlord_id)
);
create index on app.units (landlord_id);

-- ───────────────────────── leases ─────────────────────────
create table app.leases (
  id              uuid primary key default gen_random_uuid(),
  unit_id         uuid not null,
  landlord_id     uuid not null,
  tenant_id       uuid references app.profiles(id),          -- null until the tenant signs up
  tenant_email    citext not null,
  rent_cents      integer not null check (rent_cents >= 100),
  due_day         smallint not null check (due_day between 1 and 28),
  late_fee_cents  integer not null default 0 check (late_fee_cents = 0 or late_fee_cents >= 100),
  grace_days      smallint not null default 5 check (grace_days between 0 and 28),
  start_date      date not null,
  end_date        date not null,
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  check (end_date > start_date),
  foreign key (unit_id, landlord_id) references app.units (id, landlord_id) on delete restrict,
  -- no two leases on the same unit may overlap in time
  exclude using gist (unit_id with =, daterange(start_date, end_date, '[]') with &&),
  unique (id, landlord_id)                         -- target for maintenance_requests' composite FK
);
create index on app.leases (landlord_id);
create index on app.leases (tenant_id);
create index on app.leases (tenant_email) where tenant_id is null;   -- tenant linking

-- ───────────────────────── invoices ─────────────────────────
create table app.invoices (
  id            uuid primary key default gen_random_uuid(),
  lease_id      uuid not null references app.leases(id) on delete restrict,
  period_month  date not null check (extract(day from period_month) = 1),
  due_date      date not null,
  amount_cents  integer not null check (amount_cents > 0),     -- rent at creation time
  created_at    timestamptz not null default now(),
  unique (lease_id, period_month),                             -- one invoice per lease per month
  unique (id, lease_id)                                        -- target for ledger_entries' composite FK
);
create index on app.invoices (due_date);

-- ───────────────────────── payments ─────────────────────────
create table app.payments (
  id                          uuid primary key default gen_random_uuid(),
  invoice_id                  uuid not null references app.invoices(id) on delete restrict,
  tenant_id                   uuid not null references app.profiles(id),
  amount_cents                integer not null check (amount_cents >= 100),
  status                      app.payment_status not null default 'pending',
  stripe_checkout_session_id  text unique,
  stripe_payment_intent_id    text unique,
  flagged_overpayment         boolean not null default false,
  created_at                  timestamptz not null default now(),
  completed_at                timestamptz
);
create index on app.payments (invoice_id) where status = 'pending';
create index on app.payments (tenant_id);

-- ───────────────────────── ledger ─────────────────────────
create table app.ledger_entries (
  id            uuid primary key default gen_random_uuid(),
  lease_id      uuid not null references app.leases(id) on delete restrict,
  invoice_id    uuid not null,
  type          app.ledger_entry_type not null,
  amount_cents  integer not null,
  payment_id    uuid unique references app.payments(id),      -- one ledger row per payment
  created_by    uuid references app.profiles(id),             -- the landlord, for adjustments
  description   text not null,                                -- for adjustments: the required reason
  created_at    timestamptz not null default now(),
  -- charges are positive; payments and adjustments (credits) are negative
  check ( (type in ('rent_charge', 'late_fee') and amount_cents > 0)
       or (type in ('payment', 'adjustment') and amount_cents < 0) ),
  -- a payment row must point at its payment; other rows must not
  check ( (type = 'payment') = (payment_id is not null) ),
  -- an adjustment must record who made it, with a real reason
  check ( (type = 'adjustment') = (created_by is not null) ),
  check ( type <> 'adjustment' or length(description) >= 5 ),
  -- the row's lease must be its invoice's lease: one source of truth, enforced by the database
  foreign key (invoice_id, lease_id) references app.invoices (id, lease_id) on delete restrict
);
create index on app.ledger_entries (lease_id, created_at);
create index on app.ledger_entries (invoice_id);
create unique index one_rent_charge_per_invoice on app.ledger_entries (invoice_id) where type = 'rent_charge';
create unique index one_late_fee_per_invoice    on app.ledger_entries (invoice_id) where type = 'late_fee';

-- append-only: refuse UPDATE and DELETE for everyone, including the service role
create function app.ledger_is_append_only() returns trigger
language plpgsql as $$
begin
  raise exception 'ledger_entries is append-only';
end $$;

create trigger ledger_no_update_delete
  before update or delete on app.ledger_entries
  for each row execute function app.ledger_is_append_only();

-- row triggers don't fire on TRUNCATE, so block that too (the nightly demo reset drops the schema instead)
create trigger ledger_no_truncate
  before truncate on app.ledger_entries
  for each statement execute function app.ledger_is_append_only();

-- ───────────────────────── invoice balances (derived, never stored) ─────────────────────────
create view app.invoice_balances with (security_invoker = true) as
select
  i.*,
  coalesce(sum(le.amount_cents)  filter (where le.type in ('rent_charge', 'late_fee')), 0) as charged_cents,
  coalesce(-sum(le.amount_cents) filter (where le.type = 'payment'), 0)                   as paid_cents,
  coalesce(-sum(le.amount_cents) filter (where le.type = 'adjustment'), 0)                as credited_cents,
  coalesce(sum(le.amount_cents), 0)                                      as outstanding_cents,
  case
    when coalesce(sum(le.amount_cents), 0) <= 0              then 'paid'
    when count(*) filter (where le.type = 'payment') > 0     then 'partially_paid'
    else 'unpaid'
  end                                                                    as status,
  (coalesce(sum(le.amount_cents), 0) > 0
     and (now() at time zone 'America/Toronto')::date > i.due_date)      as is_overdue
from app.invoices i
left join app.ledger_entries le on le.invoice_id = i.id
-- every invoice column is listed (not just i.id) so Postgres can push a
-- "where lease_id = …" filter into the view instead of aggregating every invoice first
group by i.id, i.lease_id, i.period_month, i.due_date, i.amount_cents, i.created_at;

-- ───────────────────────── maintenance ─────────────────────────
create table app.maintenance_requests (
  id           uuid primary key default gen_random_uuid(),
  lease_id     uuid not null,
  landlord_id  uuid not null,                                 -- copied from the lease; FK keeps it honest
  tenant_id    uuid not null references app.profiles(id),
  title        text not null check (length(title) between 3 and 120),
  description  text not null check (length(description) between 1 and 4000),
  status       app.maintenance_status not null default 'open',
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  resolved_at  timestamptz,
  foreign key (lease_id, landlord_id) references app.leases (id, landlord_id) on delete restrict
);
create index on app.maintenance_requests (landlord_id, status);
create index on app.maintenance_requests (lease_id);
create index on app.maintenance_requests (tenant_id);

create table app.maintenance_photos (
  id            uuid primary key default gen_random_uuid(),
  request_id    uuid not null references app.maintenance_requests(id) on delete cascade,
  storage_path  text not null unique,
  content_type  text not null check (content_type in ('image/jpeg', 'image/png', 'image/webp')),
  size_bytes    integer not null check (size_bytes between 1 and 5242880),
  created_at    timestamptz not null default now()
);
create index on app.maintenance_photos (request_id);

-- ───────────────────────── stripe events (webhook idempotency) ─────────────────────────
create table app.stripe_events (
  id           text primary key,               -- Stripe event id, e.g. evt_...
  type         text not null,
  received_at  timestamptz not null default now()
);

-- ───────────────────────── email outbox ─────────────────────────
create table app.email_outbox (
  id          uuid primary key default gen_random_uuid(),
  dedupe_key  text not null unique,             -- e.g. 'invoice_created:<invoice_id>'
  to_email    citext not null,
  template    app.email_template not null,
  payload     jsonb not null,
  status      app.email_status not null default 'pending',
  attempts    smallint not null default 0,
  last_error  text,
  claimed_at  timestamptz,                      -- set when a sender claims the row (status 'sending')
  created_at  timestamptz not null default now(),
  sent_at     timestamptz
);
create index on app.email_outbox (created_at) where status in ('pending', 'sending');
```

`updated_at` is set by a small shared `before update` trigger on `properties`, `units`, `leases`
and `maintenance_requests` (left to the builder).

## Row Level Security

All tables have RLS enabled. **User mode** (role `authenticated`) gets only the grants and
policies below. **System mode** (the privileged role) bypasses RLS and is used only as described
in the design spec, §4.

```sql
alter table app.profiles             enable row level security;
alter table app.properties           enable row level security;
alter table app.units                enable row level security;
alter table app.leases               enable row level security;
alter table app.invoices             enable row level security;
alter table app.payments             enable row level security;
alter table app.ledger_entries       enable row level security;
alter table app.maintenance_requests enable row level security;
alter table app.maintenance_photos   enable row level security;
alter table app.stripe_events        enable row level security;   -- no policies: system only
alter table app.email_outbox         enable row level security;   -- no policies: system only

-- grants: reads everywhere a user might look; writes only where users edit directly
grant select on app.profiles, app.properties, app.units, app.leases, app.invoices,
                app.invoice_balances, app.payments, app.ledger_entries,
                app.maintenance_requests, app.maintenance_photos to authenticated;
grant insert, delete on app.properties, app.units to authenticated;
grant delete on app.leases to authenticated;
-- inserts on leases and maintenance are column-level too: a landlord can't create a lease with
-- tenant_id already set (that would attach any user to it), and a tenant can't pick a status
grant insert (unit_id, landlord_id, tenant_email, rent_cents, due_day, late_fee_cents,
              grace_days, start_date, end_date) on app.leases to authenticated;
grant insert (id, lease_id, landlord_id, tenant_id, title, description) on app.maintenance_requests to authenticated;
grant insert on app.maintenance_photos to authenticated;
-- updates are column-level: only the columns a user may ever change
grant update (name, address_line1, city, province, postal_code, updated_at) on app.properties to authenticated;
grant update (label, bedrooms, updated_at)                                  on app.units      to authenticated;
grant update (end_date, late_fee_cents, grace_days, tenant_email, updated_at) on app.leases   to authenticated;
grant update (status, resolved_at, updated_at)                    on app.maintenance_requests to authenticated;
-- invoices, payments, ledger_entries, stripe_events, email_outbox: no user writes, ever

-- profiles: yourself, plus the people you share a lease with
create policy profiles_read on app.profiles for select to authenticated using (
  id = auth.uid()
  or exists (select 1 from app.leases l
             where (l.landlord_id = auth.uid() and l.tenant_id = profiles.id)
                or (l.tenant_id  = auth.uid() and l.landlord_id = profiles.id))
);

-- properties & units: the owning landlord does everything; a tenant may read the ones they rent
create policy properties_landlord on app.properties for all to authenticated
  using (landlord_id = auth.uid()) with check (landlord_id = auth.uid());
create policy properties_tenant_read on app.properties for select to authenticated using (
  exists (select 1 from app.units u join app.leases l on l.unit_id = u.id
          where u.property_id = properties.id and l.tenant_id = auth.uid())
);

create policy units_landlord on app.units for all to authenticated
  using (landlord_id = auth.uid()) with check (landlord_id = auth.uid());
create policy units_tenant_read on app.units for select to authenticated using (
  exists (select 1 from app.leases l where l.unit_id = units.id and l.tenant_id = auth.uid())
);

-- leases
create policy leases_landlord on app.leases for all to authenticated
  using (landlord_id = auth.uid()) with check (landlord_id = auth.uid());
create policy leases_tenant_read on app.leases for select to authenticated
  using (tenant_id = auth.uid());

-- invoices, ledger, payments: read-only, via the lease
create policy invoices_read on app.invoices for select to authenticated using (
  exists (select 1 from app.leases l where l.id = invoices.lease_id
          and (l.landlord_id = auth.uid() or l.tenant_id = auth.uid()))
);
create policy ledger_read on app.ledger_entries for select to authenticated using (
  exists (select 1 from app.leases l where l.id = ledger_entries.lease_id
          and (l.landlord_id = auth.uid() or l.tenant_id = auth.uid()))
);
create policy payments_read on app.payments for select to authenticated using (
  tenant_id = auth.uid()
  or exists (select 1 from app.invoices i join app.leases l on l.id = i.lease_id
             where i.id = payments.invoice_id and l.landlord_id = auth.uid())
);

-- maintenance: tenant opens on their own lease; landlord reads and updates on theirs
create policy maint_tenant_read on app.maintenance_requests for select to authenticated
  using (tenant_id = auth.uid());
create policy maint_tenant_insert on app.maintenance_requests for insert to authenticated with check (
  tenant_id = auth.uid()
  and status = 'open'
  and exists (select 1 from app.leases l where l.id = lease_id
              and l.tenant_id = auth.uid() and l.landlord_id = maintenance_requests.landlord_id
              -- only on a lease that is current today
              and (now() at time zone 'America/Toronto')::date between l.start_date and l.end_date)
);
create policy maint_landlord_read on app.maintenance_requests for select to authenticated
  using (landlord_id = auth.uid());
create policy maint_landlord_update on app.maintenance_requests for update to authenticated
  using (landlord_id = auth.uid()) with check (landlord_id = auth.uid());

create policy photos_read on app.maintenance_photos for select to authenticated using (
  exists (select 1 from app.maintenance_requests r where r.id = maintenance_photos.request_id
          and (r.tenant_id = auth.uid() or r.landlord_id = auth.uid()))
);
create policy photos_tenant_insert on app.maintenance_photos for insert to authenticated with check (
  exists (select 1 from app.maintenance_requests r
          where r.id = request_id and r.tenant_id = auth.uid())
);
```

Things RLS deliberately does **not** enforce, which the service layer handles and tests cover:
- the order of maintenance status changes (forward only)
- which lease fields a landlord may change after invoices exist
- the overpayment check (it needs a row lock, see design §8)

## Storage

A private bucket, `maintenance-photos`. No storage policies for `authenticated`: only the API
(service key) uploads, and viewers get short-lived signed URLs after the API's user-mode read
succeeds.

## Why each constraint exists (interview notes)

| Constraint | Stops |
|---|---|
| `unique (lease_id, period_month)` | duplicate invoices when the job runs twice |
| `one_late_fee_per_invoice` | a second late fee on the same invoice |
| `stripe_events.id` primary key | applying the same Stripe webhook twice |
| `ledger_entries.payment_id unique` | one payment producing two ledger rows |
| sign `check` on ledger | a "payment" that adds debt, or a charge that reduces it |
| append-only trigger | rewriting money history, even by mistake from the API |
| `adjustment` rows (negative, with `created_by` and a reason) | fixing mistakes by editing history: corrections are new rows, like an accountant's credit note |
| `exclude using gist` on leases | two tenants renting the same unit at the same time |
| composite FKs on `landlord_id` | a lease that claims a landlord who doesn't own the unit |
| composite FK `(invoice_id, lease_id)` on ledger | a ledger row whose lease disagrees with its invoice's lease |
| column-level insert grant on `leases` | a landlord attaching an arbitrary user to a lease by setting `tenant_id` |
| truncate trigger on ledger | wiping money history with `TRUNCATE`, which row triggers don't see |
| `email_outbox.dedupe_key unique` | the same email being sent twice |
