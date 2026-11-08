# Rentline — Product Requirements

*Working name. Status: draft for review, 2026-11-08.*

## 1. Why this exists

Rentline is a portfolio project. Its job is to get its author a backend internship at a
PropTech/FinTech software company, and then to give them strong, honest things to talk about
in the interview.

So the product has two audiences:

1. **The pretend users:** landlords who collect rent and tenants who pay it.
2. **The real users:** a hiring engineer who spends about five minutes on the repo and the live
   demo, and an interviewer who asks "why did you do it this way?"

Every decision favours the second audience. That means fewer features done properly, with
reasoning the author can explain line by line. A long feature list is not the goal.

## 2. What it is

A rent collection system. Landlords set up properties, units and leases. Rent invoices are
created automatically each month. Tenants pay all or part of an invoice by card (Stripe, test
mode). Every charge and payment is recorded in an append-only ledger, and late fees are applied
automatically. Tenants can also report maintenance problems with photos.

It's a **Node.js + TypeScript REST API on Supabase**, which is the main piece, plus a simple
**Next.js web app** that uses the API.

## 3. Users and roles

| Role | Who | Can do |
|---|---|---|
| Landlord | Owns properties | Manage properties, units and leases; see invoices, ledgers and payments for their leases; manage maintenance requests on their properties |
| Tenant | Rents a unit | See their own leases, invoices and balance; pay invoices in full or in part; open maintenance requests and track them |

One account has one role, chosen at sign-up.

## 4. Features (the complete list)

1. **Accounts.** Email and password sign-up and sign-in through Supabase Auth, with the role
   picked at sign-up. Email confirmation is required.
2. **Properties and units.** A landlord creates, edits and deletes properties (name and address)
   and the units in them (a label such as "4B", plus bedrooms). Anything that has leases can't
   be deleted.
3. **Leases.** A landlord creates a lease on a unit with: tenant email, monthly rent, due day
   (1–28), late fee amount, grace days, start date and end date. Two leases on the same unit
   can't overlap in time. The tenant is emailed an invitation, and once they sign up with that
   email the lease becomes theirs automatically.
4. **Monthly rent invoices.** A scheduled job creates one invoice per lease per month the lease
   covers. Running the job again never creates a duplicate.
5. **Payments.** A tenant pays any amount from $1 up to what's still owed on an invoice (as long
   as it doesn't leave less than $1 owing), through
   Stripe Checkout (test mode, card only). Stripe tells the API when a payment succeeds, and the
   API records it exactly once, even if Stripe sends the notification more than once.
6. **Partial payments.** An invoice is *unpaid*, *partially paid* or *paid*, worked out from what
   has been paid against it. Overpaying isn't allowed.
7. **Ledger.** Every rent charge, late fee and payment is a permanent ledger row, and a lease's
   balance is the sum of its rows. Rows are never edited or deleted. Landlords and tenants can
   both see the ledger for a lease. To fix a mistake or waive a fee, the landlord adds a
   **credit adjustment** with a reason. It's a new row, so the history stays intact.
8. **Late fees.** If an invoice still has money owing after its due date plus the grace days, one
   late fee is added to it. A second fee is never added.
9. **Maintenance requests.** A tenant opens a request on their unit with a title, a description
   and up to 3 photos. The landlord moves it from *open* to *in progress* to *resolved*.
10. **Emails.** The tenant gets a lease invitation, a notice for each new invoice and a receipt
    for each payment. The landlord gets a notice for each new maintenance request. Emails never
    block or undo the action that caused them.
11. **Web app.** Landlord and tenant dashboards covering everything above. It works on a phone
    browser.

Security that applies to all of it: **Row Level Security** in Postgres means a user can only
ever read their own data, even if there's a bug in the API code.

## 5. Out of scope

Each of these is out on purpose, to keep the project small enough to explain fully:

- AI features
- Mobile app (the web app is responsive, and the API works for any client)
- Multiple currencies (everything is CAD)
- Credit balances or carrying overpayments forward
- Refunds, disputes and real money (Stripe stays in test mode)
- Prorated rent for partial months
- Changing rent in the middle of a lease
- Payment methods other than card, and autopay
- Several tenants on one lease, or one account being both landlord and tenant
- Comments or chat on maintenance requests
- Admin panel and multi-landlord organisations

## 6. What success looks like

A reviewer with five minutes can:

1. Open the live web app and sign in with the demo tenant shown on the login page. The live
   demo is login-only and reset nightly; sign-up and emails work when run locally.
2. See a balance and an overdue invoice with a late fee on it.
3. Pay part of it with Stripe's test card `4242 4242 4242 4242` and watch the invoice change to
   *partially paid* and a new ledger row appear.
4. Sign in as the demo landlord and see that payment, and a maintenance request with a photo.
5. Open the API docs (Swagger) and try an endpoint.
6. See a green CI badge, then read a README that explains the design and why.

And in the interview, the author can explain RLS, idempotency, the ledger, the outbox, and
storing money in cents, using this code as the example.

## 7. Quality bar

- Tests cover the money logic, RLS isolation and webhook handling, and run in CI on every push.
- The API and web app are deployed, with a live link in the README.
- A seed script creates the demo data, including a past-due invoice so late fees can be seen.
- The README has an architecture diagram, setup steps, the demo logins and a "design decisions"
  section.
