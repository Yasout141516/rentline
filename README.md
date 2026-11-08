# Rentline

A rent collection backend: landlords set up properties, units and leases; rent invoices are
generated automatically each month; tenants pay in full or in part through Stripe; every charge,
payment, late fee and credit lands in an append-only ledger; tenants report maintenance issues
with photos.

**Stack:** Node.js · TypeScript · Express · Supabase (Postgres, Auth, Storage, Row Level
Security) · Stripe · Resend · Next.js · Vitest · GitHub Actions

## Status

Design complete, build not started. Read the design:

1. [Product requirements](docs/01-prd.md): what it does and what it deliberately doesn't
2. [Design specification](docs/02-design.md): architecture, API, money rules, payments, jobs,
   testing, and the engineering review
3. [Database schema](docs/03-schema.md): tables, constraints, RLS policies, and why each
   constraint exists
