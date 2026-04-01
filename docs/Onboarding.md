
Welcome to Constructmart. This document gets you from zero to productive in 2 days. Follow it in order — each step assumes the previous ones are done.

---

## Before day 1 — request these accesses

Ask the team lead to set these up before your first day. Without them you will be blocked immediately.

- [ ] GitHub organisation invite (constructmart)
- [ ] AWS IAM user with developer permissions
- [ ] Razorpay test account access
- [ ] MSG91 test account access
- [ ] Sentry organisation invite
- [ ] Datadog organisation invite (read access)
- [ ] `.env` values for local development
- [ ] Staging API URL and test credentials

---

## Day 1 — get the project running locally

### Step 1 — clone and install

```bash
git clone <repo-url>
cd constructmart
npm install
```

### Step 2 — environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in the values you received from the team lead. Every variable is explained in `docs/environment.md`. The only ones you need for local development are:

- `DATABASE_URL` — local PostgreSQL (already set in .env.example)
- `REDIS_URL` — local Redis (already set in .env.example)
- `OPENSEARCH_URL` — local OpenSearch (already set in .env.example)
- `JWT_SECRET` — any random string for local dev
- `RAZORPAY_KEY_ID` + `RAZORPAY_KEY_SECRET` — test keys from team lead
- `MSG91_AUTH_KEY` — test key from team lead

### Step 3 — start local services

```bash
docker-compose up -d
```

This starts PostgreSQL, Redis, and OpenSearch locally. Wait about 30 seconds for OpenSearch to finish initialising, then verify all three are up:

```bash
docker-compose ps
# All three should show "Up"
```

### Step 4 — run migrations and seed data

```bash
npm run migrate
# Should print: "Migrations complete. X files applied."

npm run seed
# Seeds: cities, pincodes (Delhi NCR), categories,
#        GST rates, and one test supplier + products
```

### Step 5 — start the API

```bash
npm run dev
```

The API starts on `http://localhost:3000`. Verify it is healthy:

```bash
curl http://localhost:3000/health
# Expected response:
# {
#   "status": "ok",
#   "db": "ok",
#   "redis": "ok",
#   "search": "ok",
#   "version": "x.x.x"
# }
```

If any service shows `"error"` instead of `"ok"`, check `docs/runbooks/local-setup-troubleshooting.md`.

### Step 6 — import the Postman collection

The Postman collection lives at `docs/postman-collection.json`. Import it into Postman. It contains example requests for every API endpoint with test data pre-filled.

Start with the auth flow:

1. `POST /api/v1/auth/otp/request` — request an OTP
2. Check the console log (OTPs are logged in dev mode, not actually sent via SMS)
3. `POST /api/v1/auth/otp/verify` — verify the OTP, receive a JWT token
4. Use that token as Bearer in subsequent requests

### Step 7 — run the test suite

```bash
npm run test
# All tests should pass on a fresh setup
```

If any test fails on a fresh clone, that is a bug — raise it immediately rather than working around it.

---

## Day 1 — read these documents

With the project running, spend the rest of day 1 reading. Do not skip this. Engineers who skip the reading spend weeks being confused by non-obvious design decisions.

**Read in this order:**

1. `docs/architecture.md` — the big picture. How everything connects. Read the two request flow sections carefully.
    
2. `docs/database.md` — every table and why it exists. Pay special attention to:
    
    - The "Rules every engineer must follow" section at the top
    - The `orders` vs `sub_orders` explanation
    - The `products` vs `supplier_products` explanation
3. `docs/decisions/ADR-002-marketplace-model.md` — why the platform works the way it does. Most design decisions trace back to this one.
    
4. `docs/decisions/ADR-003-money-in-paise.md` — read the code examples. You will write financial code. Get this right.
    
5. `docs/decisions/ADR-004-outbox-pattern.md` — understand why we never write to PostgreSQL and OpenSearch in the same API request.
    

---

## Day 2 — understand the codebase

### Trace the order flow in code

The order flow is the most complex part of the system. Understanding it means you understand almost everything else.

Follow this path through the code:

```
src/modules/cart/cart.routes.ts
    → POST /cart/checkout
    → src/modules/cart/cart.service.ts → checkout()
        → reads cart from Redis
        → verifies prices from PostgreSQL (not OpenSearch)
        → groups items by supplier_id
        → calculates GST using src/shared/gst/gst.service.ts
        → calls src/modules/orders/orders.service.ts → createOrder()
            → opens PostgreSQL transaction
            → inserts orders row
            → inserts sub_orders rows (one per supplier)
            → inserts order_items rows
            → reserves stock in supplier_products
            → inserts search_sync_outbox rows (stock changed)
            → commits transaction
        → calls Razorpay to create payment order
        → returns payment_order_id to client

src/modules/payments/payments.routes.ts
    → POST /payments/webhook
    → src/modules/payments/payments.service.ts → handleWebhook()
        → verifies Razorpay signature
        → checks payment_webhook_log for duplicate
        → updates orders and sub_orders status
        → enqueues job in BullMQ → q:order-confirm

src/workers/order-confirm.ts
    → consumes from q:order-confirm
    → sends WhatsApp notifications
    → generates e-way bill if needed
    → generates GST invoice PDF
```

### Understand the search sync worker

```
src/workers/search-sync.ts
    → polls search_sync_outbox every 5 seconds
    → for each unprocessed row:
        → fetches full product data from PostgreSQL
        → builds OpenSearch document
        → calls OpenSearch bulk API
        → marks row as processed
    → on failure: increments retry_count, sets failed_at
```

### Run the workers locally

```bash
npm run workers
# Starts all background workers in one process for local dev
# In production they run as separate processes
```

---

## Things that are non-obvious

Read this section carefully. These are the things that will confuse you if you encounter them without context.

### Orders have two status fields

`orders.status` is customer-facing and derived automatically by the order service based on its sub-orders. Never set it with a direct UPDATE. The order service recomputes it every time a sub-order status changes.

`sub_orders.status` is what suppliers and the operations team actually update. This is the real source of truth for delivery progress.

### Stock reservation happens at checkout, not at payment

When a customer submits their cart, stock is reserved in `supplier_products.reserved_quantity` immediately — before payment completes. This prevents two customers from buying the last unit simultaneously.

If payment fails or expires, the reservation is released by a scheduled cleanup job that runs every 15 minutes. See `src/workers/order-confirm.ts` → `releaseExpiredReservations()`.

### Delhi to Noida is interstate

Noida is in Uttar Pradesh. Gurgaon is in Haryana. Both feel like "Delhi" to users but are different states for GST purposes. A delivery from a Delhi warehouse to a Noida address uses IGST, not CGST + SGST.

Always use `pincodes.state_code` to determine this. Never use the city name string.

### OpenSearch documents are supplier_product level

One product listed by 5 suppliers = 5 OpenSearch documents. Each document contains the supplier-specific price and stock. When a user searches and sees results, each result card corresponds to one `supplier_product` row, not one `product` row.

### OTPs are logged to console in development

In development mode (`NODE_ENV=development`), OTPs are printed to the console instead of being sent via SMS. This saves MSG91 credits during development. Look for: `[OTP] Phone: 9999999999 OTP: 123456`

### Migration files must be backward-compatible

During a rolling deploy, the old and new app versions run simultaneously against the same database for ~60 seconds. A migration that drops or renames a column used by the old version will break live traffic.

Safe pattern:

1. Migration: add new column (old app ignores it, new app uses it)
2. Deploy new app code
3. Next migration: drop old column

### All city-scoped queries use a helper

Never write `SELECT * FROM products WHERE ...` without scoping to a city. Use the shared helper:

```typescript
// CORRECT
const products = await db.catalog.forCity(cityId).findMany({ ... });

// WRONG — missing city scope
const products = await db.query('SELECT * FROM products WHERE ...');
```

The helper adds `AND city_id = $cityId` automatically and throws if `cityId` is undefined — making it impossible to accidentally return products from all cities.

---

## Development workflow

### Branching

```
main          ← production-ready code, auto-deploys to staging
feature/xyz   ← your feature branch, opened from main
fix/xyz       ← bug fix branch
```

Never commit directly to `main`. Always open a pull request.

### Pull request checklist

Before opening a PR, make sure:

- [ ] Tests pass locally (`npm run test`)
- [ ] TypeScript has no errors (`npm run typecheck`)
- [ ] Lint passes (`npm run lint`)
- [ ] If you added a table or column, there is a migration file
- [ ] If you changed financial logic, a CA/team lead has reviewed it
- [ ] If you made a significant architectural decision, there is an ADR
- [ ] `CHANGELOG.md` is updated if this is a user-facing change

### Writing migrations

```bash
# Create a new migration file
npm run migrate:create -- "add_delivery_slots_table"
# Creates: src/db/migrations/013_add_delivery_slots_table.sql
```

Every migration file must have:

1. A comment block explaining what it does and why
2. The schema changes
3. Required indexes
4. A `-- rollback` section with DROP statements

See existing migrations for the exact format.

### Commit message format

```
feat: add supplier stock bulk update API
fix: correct GST calculation for inter-state orders
chore: update Razorpay SDK to v2.9.0
docs: add runbook for search sync worker failure
migration: add delivery_slots table
```

---

## Getting help

### If the local setup is not working

Check `docs/runbooks/local-setup-troubleshooting.md` first. It covers the most common issues with Docker, migrations, and OpenSearch initialisation.

### If you are unsure about a design decision

Check `docs/decisions/` — there is an ADR for every major decision made so far. If the question is not covered, ask the team lead before proceeding. Do not guess on financial logic, GST calculations, or schema changes.

### If something is broken in staging or production

Check `docs/runbooks/` for the relevant runbook. For P1 incidents (payment processing down, API returning 5xx for > 5 minutes), follow `docs/runbooks/incidents/p1-response.md`.

### Contacts

|Role|Name|Contact|
|---|---|---|
|Backend / DevOps|[your name]|[your contact]|
|Product / Business|[co-founder]|[contact]|
|GST / Compliance|[CA name]|[contact]|
|Razorpay support|—|+91-22-68213838|
|AWS support|—|via AWS console|

---

## 30-day goals

By the end of your first month you should be able to:

- [ ] Deploy a change to production independently
- [ ] Write and apply a database migration
- [ ] Debug a failing background worker using logs
- [ ] Explain the order splitting flow to someone else
- [ ] Identify whether a delivery is interstate or intra-state and explain why it matters for GST
- [ ] Add a new API endpoint with proper auth, validation, and error handling following existing patterns